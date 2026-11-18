<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 07 — Algorithms and data structures](README.md) · [📋 Phase 07](README.md) · [07.2 Analyzing time and space →](02-analyzing-time-and-space.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/01-algorithms-and-big-o.md) <!-- nav -->

# Lesson 07.1 — What an algorithm is, and Big-O notation

> **Phase 07 — Algorithms and data structures** · Lesson 1 of 20 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What an algorithm is, and what makes one algorithm better than another.
- How to count the steps of a small algorithm instead of timing it with a stopwatch.
- What Big-O notation says: `O(1)`, `O(log n)`, `O(n)`, `O(n log n)`, `O(n²)` and `O(2ⁿ)`, and how fast each one grows.
- Three rules for reading and writing Big-O.
- How the practice package `code/p07-algorithms` of this whole phase is organised.

## Before you start

- You finished [Lesson 06.12 — Checkpoint: Stage 1 complete](../06-rust-advanced/12-checkpoint-stage-1-complete.md). You know Rust well enough to read every example below.
- You know vectors and slices ([Lesson 04.7](../04-structs-enums-collections-errors/07-vectors.md), [Lesson 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md)), `HashSet` ([Lesson 04.9](../04-structs-enums-collections-errors/09-hash-maps.md)) and tests ([Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md)).

## New words

| Word | Meaning |
|---|---|
| **algorithm** | A precise list of steps that solves a problem and always stops. |
| **input size (`n`)** | The number that tells how big the input is: the length of a vector, the number of words in a text. By tradition it is called `n`. |
| **time complexity** | How the number of steps grows when `n` grows. |
| **space complexity** | How the extra memory grows when `n` grows (Lesson 07.2). |
| **Big-O notation** | A short way to write complexity: `O(n)` is read "order n" or "big-O of n". It keeps only the part that grows fastest. |
| **best case / worst case** | The input that makes the algorithm finish the fastest / the slowest, for a given `n`. |
| **linear / quadratic** | `O(n)`: double the input, double the work / `O(n²)`: double the input, four times the work. |
| **logarithm** (`log n`) | Here: how many times you can cut `n` in half before you reach 1. For 1,000 it is 9; for 1,000,000 it is 19. It grows very slowly. |
| **exponential** | `O(2ⁿ)`: each new item doubles the work. Unusable from about `n = 40`. |
| **module** | A named part of a crate, usually one file. Here: one module per topic. |
| **example (Cargo)** | A small program in the `examples/` folder of a package, run with `cargo run --example <name>`. |
| **`std`** | The Rust standard library. |

## 1. What an algorithm is

An **algorithm** is a precise list of steps that turns an **input** into an **output**. Four properties
make it an algorithm and not a wish:

1. It has a clear input and a clear output.
2. Every step is precise: a computer could do it without guessing.
3. It always stops (after a finite number of steps).
4. It gives the right answer for every valid input.

Here is an everyday algorithm: *find the tallest person in a room.* Look at the first person and
remember the height. Look at each next person. If this person is taller than the remembered height,
remember the new height. When nobody is left, the remembered height is the answer. You already wrote
this in Stage 1 as a `for` loop.

Many different algorithms can solve the same problem. They all give the right answer, but one may take
a second and another may take a year on a big input. A **data structure** is a way of organising data
in memory (a vector, a hash map, a tree). Algorithms and data structures go together: the right
structure often makes the algorithm much faster. This phase teaches both.

So we need a way to say "this algorithm is faster than that one". A stopwatch is a poor judge: the
result depends on the computer, on the other programs that run, and on the input you happened to
choose. We want a measure that is the same on every computer. That measure is **the number of steps as
a function of the input size `n`**.

## 2. The practice package of this phase

All code of Phase 07 lives in **one library package** with **one module per topic**. Create it once:

```bash
cd ~/projects/rust
cargo new --lib code/p07-algorithms
printf '/target\n' > code/p07-algorithms/.gitignore
cd code/p07-algorithms
mkdir examples
```

**What happens:**

1. `cargo new --lib` creates a package with `src/lib.rs` (Lesson 05.6 explained library packages).
   Inside Rust code the crate is called `p07_algorithms`: dashes become underscores.
2. The `.gitignore` line tells Git to ignore the `target` folder, where Cargo puts build results.
3. `mkdir examples` creates the folder for small runnable programs.

The package grows like this during the phase:

```text
code/p07-algorithms/
├── Cargo.toml
├── src/
│   ├── lib.rs          ← one `pub mod <topic>;` line per topic
│   ├── big_o.rs        ← this lesson
│   ├── analysis.rs     ← Lesson 07.2
│   └── ...             ← one file per topic in the next lessons (sorting.rs, graphs.rs, ...)
├── examples/           ← small programs: cargo run --example <name>
└── benches/            ← benchmarks (criterion), added by the lessons that need them
```

Three rules for the whole phase:

- **Unit tests live next to the code**, in a `#[cfg(test)] mod tests` at the bottom of each module
  (Lesson 05.6). `cargo test` runs all of them.
- **A library cannot be run**, so a runnable demonstration goes into the `examples/` folder. A file
  `examples/e01_steps.rs` is a normal program with its own `fn main`, and it reaches your library with
  `use p07_algorithms::big_o::...;`. You run it with `cargo run --example e01_steps`. (Examples are named
  `eNN_...` after the lesson, so they stay in order.)
- **Benchmarks live in `benches/`** and use `criterion`, which you met in
  [Lesson 05.12](../05-rust-intermediate/12-benchmarking-basics.md). A lesson that has a benchmark tells
  you exactly what to add to `Cargo.toml`.

The project at the end of the phase (Lesson 07.19) has its own package, `code/p07-route-finder`.

Open `src/lib.rs`, delete the sample code and write one line:

```rust
pub mod big_o;
```

`pub mod big_o;` tells Rust to load the module from the file `src/big_o.rs`. `pub` makes it visible
to the examples and to other crates. Every new topic in this phase adds one such line.

## 3. Counting steps

We will count **comparisons**, because comparisons are the main work in the three algorithms below.
Create `src/big_o.rs` and write the first algorithm, from the tallest-person story:

```rust
/// Returns the largest number in the slice, or `None` if the slice is empty.
pub fn largest(numbers: &[i32]) -> Option<i32> {
    if numbers.is_empty() {
        return None;
    }
    let mut best = numbers[0];
    for &n in &numbers[1..] {
        if n > best {
            best = n;
        }
    }
    Some(best)
}
```

**What happens:**

1. An empty slice has no largest number, so the answer is `None`.
2. `best` remembers the largest number seen so far. It starts as the first item.
3. `&numbers[1..]` is the slice without the first item. The loop looks at each of the other items once.
4. For `n` items the loop makes `n - 1` comparisons. If `n` doubles, the work doubles. There is no lucky
   input (every item must be looked at), so the best case and the worst case are the same.

Now a search. **Linear search** looks for a value from left to right and stops when it finds it. This
version also returns the number of comparisons it made, so we can study it:

```rust
/// Looks for `target` from left to right.
/// Returns the position (if found) and the number of comparisons made.
pub fn linear_search_counted(items: &[i32], target: i32) -> (Option<usize>, usize) {
    let mut comparisons = 0;
    for (index, &item) in items.iter().enumerate() {
        comparisons += 1;
        if item == target {
            return (Some(index), comparisons);
        }
    }
    (None, comparisons)
}
```

**What happens:** `items.iter().enumerate()` gives pairs `(index, item)`; the pattern `(index, &item)` copies
the `i32` out of the reference. Each pass of the loop is one comparison. On a match the function returns at
once. If the loop ends, the value is not there, and the answer is `None`.

The **best case** is when the target is the first item: 1 comparison, whatever `n` is. The **worst case**
is when the target is last or missing: `n` comparisons. We usually care about the worst case, because a
promise "never slower than this" is more useful than "sometimes faster".

The third algorithm checks whether any value appears twice, by comparing every pair of items:

```rust
/// Checks whether some value appears twice, by comparing every pair.
/// Returns the answer and the number of comparisons made.
pub fn has_duplicate_counted(items: &[i32]) -> (bool, usize) {
    let mut comparisons = 0;
    for i in 0..items.len() {
        for j in (i + 1)..items.len() {
            comparisons += 1;
            if items[i] == items[j] {
                return (true, comparisons);
            }
        }
    }
    (false, comparisons)
}
```

**What happens:** The outer loop picks the position `i`; the inner loop picks every position `j` *after* `i`,
so each pair is compared once. Two equal items stop the function with `true`. In the worst case (all items
different) the number of pairs is `(n-1) + (n-2) + ... + 1 = n(n-1)/2`.

Let us see the numbers. The file `examples/e01_steps.rs` counts the worst case for growing `n`:

```rust
use p07_algorithms::big_o::{has_duplicate_counted, linear_search_counted};

fn main() {
    println!(
        "{:>8} {:>16} {:>16}",
        "n", "search (worst)", "pairs (worst)"
    );
    for n in [10, 100, 1_000, 10_000] {
        let items: Vec<i32> = (0..n).collect(); // all different, 0..n
        let (_, search_steps) = linear_search_counted(&items, -1); // -1 is not there
        let (_, pair_steps) = has_duplicate_counted(&items);
        println!("{:>8} {:>16} {:>16}", n, search_steps, pair_steps);
    }
}
```

**What happens:** `(0..n).collect()` builds the vector `0, 1, ..., n-1` with all items different. The search
for `-1` never succeeds and the duplicate check never finds a duplicate, so both run through their worst
case. `{:>8}` prints a number right-aligned in 8 characters (Lesson 02.6).

Run it with `cargo run --example e01_steps`:

```text
       n   search (worst)    pairs (worst)
      10               10               45
     100              100             4950
    1000             1000           499500
   10000            10000         49995000
```

Look at the two columns. When `n` grows 10 times, the search needs 10 times more comparisons. The pair
check needs about 100 times more: from 45 to 4,950 to 499,500 to 49,995,000. This is the whole idea of
the lesson. The two algorithms are different **in the way their cost grows**.

## 4. Big-O notation

**Big-O notation** is the short way to write "how the cost grows with `n`". We say:

- `largest` and `linear_search_counted` are `O(n)` (read: "order n", or "linear").
- `has_duplicate_counted` is `O(n²)` ("order n squared", or "quadratic").

A precise definition, for the curious: *a cost `f(n)` is `O(g(n))` if, from some size on, `f(n)` is never
larger than some constant number times `g(n)`.* In practice you only need the idea: **keep the part that
grows fastest and throw the rest away.**

The common growth rates, from the best to the worst:

| Notation | Name | Example | If `n` doubles |
|---|---|---|---|
| `O(1)` | constant | read `v[5]` from a vector | the same |
| `O(log n)` | logarithmic | binary search (Lesson 07.10) | one more step |
| `O(n)` | linear | `largest`, `linear_search` | twice the work |
| `O(n log n)` | "n log n" | good sorting (Lesson 07.9) | a little more than twice |
| `O(n²)` | quadratic | compare all pairs | four times |
| `O(2ⁿ)` | exponential | try all subsets | the square (very bad) |

`log n` needs one more explanation. Take `n = 1000`. Cut it in half: 500, 250, 125, 62, 31, 15, 7, 3, 1.
That is 9 cuts. The number of cuts is `log n` (to be exact, `log₂ n`, but in Big-O the base does not
matter). Algorithms that cut the problem in half at each step are `O(log n)`. Here is a function that
counts the cuts:

```rust
/// How many times can `n` be cut in half (rounding down) before it reaches 1?
pub fn halvings(mut n: u64) -> u32 {
    let mut count = 0;
    while n > 1 {
        n /= 2;
        count += 1;
    }
    count
}
```

The file `examples/e01_growth.rs` prints how many steps each growth rate needs for several sizes:

```rust
use p07_algorithms::big_o::halvings;

fn main() {
    println!(
        "{:>10} {:>4} {:>6} {:>10} {:>12} {:>16}",
        "n", "1", "log n", "n", "n log n", "n squared"
    );
    for n in [10u64, 100, 1_000, 1_000_000] {
        let log_n = halvings(n);
        println!(
            "{:>10} {:>4} {:>6} {:>10} {:>12} {:>16}",
            n,
            1,
            log_n,
            n,
            n * u64::from(log_n),
            n * n
        );
    }

    println!();
    println!("2 to the power n:");
    for n in [10u32, 20, 30, 40, 50, 60] {
        println!("{:>4} {:>22}", n, 1u64 << n);
    }
}
```

**What happens:** `halvings(n)` is `log n`, and `n * u64::from(log_n)` is `n log n` (`u64::from` widens the
`u32` safely). `1u64 << n` is 2 to the power `n`: the bit `1` shifted left by `n` places. With `n = 64` it
would no longer fit into a `u64` (Lesson 02.2: overflow). Run it with `cargo run --example e01_growth`:

```text
         n    1  log n          n      n log n        n squared
        10    1      3         10           30              100
       100    1      6        100          600            10000
      1000    1      9       1000         9000          1000000
   1000000    1     19    1000000     19000000    1000000000000

2 to the power n:
  10                   1024
  20                1048576
  30             1073741824
  40          1099511627776
  50       1125899906842624
  60    1152921504606846976
```

Read the table by columns, as if each column were an algorithm working on a vector of one million
items:

- `O(log n)` needs **19** steps.
- `O(n)` needs 1,000,000 steps: a blink for a computer.
- `O(n log n)` needs 19,000,000 steps: still fast.
- `O(n²)` needs **1,000,000,000,000** steps. At a billion simple steps per second that is about 17 minutes.
- `O(2ⁿ)` at `n = 60` is about a billion billion (10¹⁸) steps. No computer can do it.

That is why Big-O matters. For small `n` nobody notices a difference. For large `n` the growth rate
decides whether the program finishes today or never.

## 5. Three rules for Big-O

**Rule 1: drop constant factors.** A loop that does 3 comparisons per item makes `3n` steps. A loop
that does 1 makes `n` steps. Both are `O(n)`. Constants depend on the computer and the compiler; the
growth shape does not. Two loops one after the other (`n + n = 2n`) are still `O(n)`.

**Rule 2: drop the smaller terms.** If an algorithm makes `n² + 5n + 100` steps, then for `n = 1000` that
is 1,000,000 + 5,000 + 100. The `n²` part is almost all of it, and the bigger `n` is, the more true that
becomes. So the algorithm is `O(n²)`.

**Rule 3: say which case you mean.** Unless told otherwise, Big-O describes the **worst case**. So
`linear_search_counted` is `O(n)` even though it may finish after one comparison. When the best or the
average case matters, say so: "best case `O(1)`, worst case `O(n)`".

Two warnings that beginners often need:

> ⚠️ **Common mistake:** `O(1)` does not mean "fast" and `O(n²)` does not mean "slow". Big-O only says
> how the cost *grows*. An `O(1)` operation that takes one second is slower than an `O(n)` loop over ten
> items. For small `n` the constant factors win; this is why very simple algorithms are often the best
> choice for small inputs. You will see this with real numbers in Lesson 07.2.

> ⚡ **Performance:** Big-O tells you which algorithm to *consider*. A measurement tells you which one
> is *faster on your data* ([Lesson 05.12](../05-rust-intermediate/12-benchmarking-basics.md)). Use both.

## 6. Tests: compare with the standard library on small inputs

An algorithm that is fast but wrong is worthless. A good way to test one: write the same function in a
very different, obviously correct way and compare the two on **many inputs**. For tiny inputs you can
even try **all** of them (an *exhaustive test*). Most bugs show up on small inputs. Add this at the bottom
of `src/big_o.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::HashSet;

    #[test]
    fn largest_of_empty_is_none() {
        assert_eq!(largest(&[]), None);
        assert_eq!(largest(&[4, 9, 2]), Some(9));
    }

    #[test]
    fn linear_search_counts_comparisons() {
        let v = [5, 8, 13, 21];
        assert_eq!(linear_search_counted(&v, 5), (Some(0), 1)); // best case
        assert_eq!(linear_search_counted(&v, 21), (Some(3), 4));
        assert_eq!(linear_search_counted(&v, 99), (None, 4)); // worst case
    }

    #[test]
    fn has_duplicate_matches_a_hash_set() {
        // every vector of length 3 with items from 0 to 3: 64 vectors
        for a in 0..4 {
            for b in 0..4 {
                for c in 0..4 {
                    let v = [a, b, c];
                    let unique: HashSet<i32> = v.iter().copied().collect();
                    let expected = unique.len() != v.len();
                    assert_eq!(has_duplicate_counted(&v).0, expected, "input {v:?}");
                }
            }
        }
    }

    #[test]
    fn duplicate_check_worst_case_is_n_times_n_minus_1_over_2() {
        let v: Vec<i32> = (0..100).collect(); // all different: no early exit
        assert_eq!(has_duplicate_counted(&v), (false, 100 * 99 / 2));
    }

    #[test]
    fn halvings_examples() {
        assert_eq!(halvings(1), 0);
        assert_eq!(halvings(8), 3);
        assert_eq!(halvings(1000), 9);
        assert_eq!(halvings(1_000_000), 19);
    }
}
```

**What happens:**

1. `largest_of_empty_is_none` tests the edge case: the empty slice.
2. `linear_search_counts_comparisons` checks the best case (1 comparison), a middle case and the worst
   case (4 comparisons).
3. `has_duplicate_matches_a_hash_set` visits all 64 vectors of length 3 over the values 0 to 3 with three
   nested loops. For each one it compares our `O(n²)` answer with a different method: put the items into a
   `HashSet` (a set keeps every value once) and see whether the set is smaller than the vector.
4. The last two tests fix the formula `n(n-1)/2` from section 3 (for `n = 100`) and the values of `log n`.

Run `cargo test`:

```text
running 5 tests
test big_o::tests::halvings_examples ... ok
test big_o::tests::largest_of_empty_is_none ... ok
test big_o::tests::duplicate_check_worst_case_is_n_times_n_minus_1_over_2 ... ok
test big_o::tests::linear_search_counts_comparisons ... ok
test big_o::tests::has_duplicate_matches_a_hash_set ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(The order of the lines can differ on your computer: tests run in parallel.)

> 🧪 **Testing:** "Compare with a slow-but-obviously-correct version on all small inputs" is the most
> useful testing trick of this phase. You will use it for sorting, searching, trees and graphs.

## 7. When to use the standard library instead

You wrote these algorithms to understand them. In real code use what `std` offers:

| You want | Use | Cost |
|---|---|---|
| the largest item | `v.iter().max()` | `O(n)` |
| the position of a value | `v.iter().position(\|x\| *x == target)` or `v.contains(&target)` | `O(n)` |
| "does this vector have duplicates?" | put the items into a `HashSet`, compare lengths | `O(n)` on average |

The standard library documents the cost of its collections in a table (see the sources). Learn to read it:
it is Big-O in daily life.

## Common mistakes

**1. Indexing an empty slice.** The first item of a slice is `numbers[0]`, but only if the slice has one.

```rust
fn first_or_panic(numbers: &[i32]) -> i32 {
    numbers[0]
}

fn main() {
    let empty: Vec<i32> = Vec::new();
    println!("{}", first_or_panic(&empty));
}
```

Run it with `cargo run --example e01_panic` (if you put it into `examples/`):

```text

thread 'main' (4624881) panicked at examples/e01_panic.rs:2:5:
index out of bounds: the len is 0 but the index is 0
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

(The number after `thread 'main'` differs on your computer.) That is why `largest` first checks
`numbers.is_empty()`. Always think about the empty input and the one-item input.

**2. Mixing up two different `n`.** If a function takes two vectors `a` and `b`, there are two sizes. A loop
over `a` with an inner loop over `b` is `O(a.len() * b.len())`, not `O(n²)`. Name your sizes.

## Try it yourself

1. Create the package (section 2) and write `pub mod big_o;` in `src/lib.rs`.
2. Type `largest`, `linear_search_counted`, `has_duplicate_counted`, `halvings` and the tests into
   `src/big_o.rs` yourself (typing finds details that copying hides). Make `cargo test`, `cargo clippy`
   and `cargo fmt` clean.
3. Create the two examples and run them. Change the sizes in `e01_steps.rs` to `[10, 20, 40, 80]` and check
   that the pair count grows about four times each time the size doubles.
4. Commit this step:

```bash
cd ~/projects/rust
git add code/p07-algorithms
git commit -m "feat(p07): add the algorithms package and the Big-O basics"
```

## Exercises

### Exercise 1 — Classify the loops (easy)

Give the time complexity of each fragment, for a vector `v` of length `n`. (The fragments are parts of
`main`; you do not need to run them.)

```rust
let mut total = 0;                  // (a)
for x in &v { total += x; }

let first = v[0];                   // (b)

for x in &v {                       // (c)
    for y in &v { println!("{x} {y}"); }
}

let mut k = v.len();                // (d)
while k > 1 { k /= 2; }
```

<details>
<summary>Hint</summary>

Count how many times the innermost statement runs as a function of `n`. For (d) ask how many times you
can halve `n`.

</details>

<details>
<summary>Answer</summary>

(a) `O(n)`: one addition per item. (b) `O(1)`: one step, whatever `n` is. (c) `O(n²)`: `n` times `n` prints.
(d) `O(log n)`: `k` is halved each time.

Why it works: constants are dropped (rule 1) and nested loops multiply their counts.

</details>

### Exercise 2 — The second largest (medium)

Write `pub fn second_largest(numbers: &[i32]) -> Option<i32>` that returns the second item in descending
order (equal items count separately: for `[5, 5]` the answer is `Some(5)`). Use **one pass** over the slice
and state the time complexity. Test it against a version that sorts a copy.

<details>
<summary>Hint</summary>

Keep two variables, `first` and `second`, both `Option<i32>`. A number that beats `first` pushes the old
`first` down into `second`; any other number may still beat `second`.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn second_largest(numbers: &[i32]) -> Option<i32> {
    let mut first: Option<i32> = None;
    let mut second: Option<i32> = None;
    for &n in numbers {
        match first {
            None => first = Some(n),
            Some(f) if n > f => {
                second = first;
                first = Some(n);
            }
            Some(_) => {
                if second.is_none_or(|s| n > s) {
                    second = Some(n);
                }
            }
        }
    }
    second
}
```

The test (inside `mod tests`):

```rust
    #[test]
    fn second_largest_matches_sorting() {
        for a in 0..4 {
            for b in 0..4 {
                for c in 0..4 {
                    let v = [a, b, c];
                    let mut sorted = v.to_vec();
                    sorted.sort_by(|x, y| y.cmp(x)); // largest first
                    assert_eq!(second_largest(&v), sorted.get(1).copied(), "input {v:?}");
                }
            }
        }
        assert_eq!(second_largest(&[]), None);
        assert_eq!(second_largest(&[7]), None);
    }
```

The complexity is `O(n)`: one pass, a constant number of steps per item, and `O(1)` extra memory (two
variables). The sorting version is `O(n log n)` (Lesson 07.9).

</details>

### Exercise 3 — Count the triples (hard)

Write `pub fn count_triples(n: u64) -> u64` that counts how many times the innermost line runs in the loops
`for i in 0..n`, `for j in (i + 1)..n`, `for k in (j + 1)..n`. Find the formula by hand, test it for `n` from 0
to 29, and give the Big-O.

<details>
<summary>Hint</summary>

Each run picks 3 different numbers out of `n` without order. There are `n(n-1)(n-2)/6` ways (for `n >= 3`;
below 3 the answer is 0).

</details>

<details>
<summary>Answer</summary>

```rust
pub fn count_triples(n: u64) -> u64 {
    let mut count = 0;
    for i in 0..n {
        for j in (i + 1)..n {
            for _k in (j + 1)..n {
                count += 1;
            }
        }
    }
    count
}
```

```rust
    #[test]
    fn triples_formula() {
        for n in 0..30u64 {
            let formula = if n < 3 { 0 } else { n * (n - 1) * (n - 2) / 6 };
            assert_eq!(count_triples(n), formula);
        }
    }
```

The count is about `n³/6`. Dropping the constant `1/6` and the smaller terms, the Big-O is `O(n³)`. Every
extra nested loop of this kind multiplies the growth by another `n`.

</details>

## Quiz

1. Why do we count steps instead of measuring seconds?

<details>
<summary>Answer</summary>

Seconds depend on the computer, the compiler settings and the other programs running. The number of steps
as a function of `n` is the same everywhere and shows how the cost *grows*.

</details>

2. What is the Big-O of `3n² + 20n + 7`?

<details>
<summary>Answer</summary>

`O(n²)`: drop the constant factor 3 and the smaller terms `20n` and `7`.

</details>

3. What are the best case and the worst case of linear search in `n` items?

<details>
<summary>Answer</summary>

Best: the target is first, 1 comparison, `O(1)`. Worst: the target is last or missing, `n` comparisons, `O(n)`.

</details>

4. Algorithm A makes `50n` steps, algorithm B makes `n²` steps. For which `n` is B faster? Which one is better for huge `n`?

<details>
<summary>Answer</summary>

`n² < 50n` means `n < 50`. For `n` below 50, B is faster; above 50, A wins and the gap grows without limit.
A is `O(n)`, B is `O(n²)`. So Big-O tells you the winner for large `n`, but for tiny `n` the "worse" algorithm can win.

</details>

## Summary

- An algorithm is a precise list of steps that solves a problem and always stops. Different algorithms for the
  same problem can differ enormously in cost.
- We measure an algorithm by counting steps as a function of the input size `n`, not by timing it.
- Big-O keeps only the fastest-growing part: `O(1)`, `O(log n)`, `O(n)`, `O(n log n)`, `O(n²)`, `O(2ⁿ)`.
- Rules: drop constants, drop smaller terms, and say which case (usually the worst) you mean.
- Big-O is about growth, not about the speed for one input; measure when it matters.
- `code/p07-algorithms` holds one module per topic, unit tests next to the code, examples in `examples/`,
  benchmarks in `benches/`.
- A good test compares your algorithm with an obviously correct one on all small inputs.

## Sources and further reading

- [std::collections — module documentation](https://doc.rust-lang.org/std/collections/index.html) — the "Performance" section lists the Big-O cost of every standard collection.
- [Iterator::max](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.max) and [Iterator::position](https://doc.rust-lang.org/std/iter/trait.Iterator.html#method.position) — the standard library versions of our first two algorithms.
- [The Cargo Book — Cargo targets: examples](https://doc.rust-lang.org/cargo/reference/cargo-targets.html#examples) — how the `examples/` folder works.
- [The Rust Programming Language — 11.3 Test Organization](https://doc.rust-lang.org/book/ch11-03-test-organization.html) — unit tests next to the code.
- [The Rust Programming Language — 7. Packages, Crates, and Modules](https://doc.rust-lang.org/book/ch07-00-managing-growing-projects-with-packages-crates-and-modules.html) — packages and modules.

---

[🏠 Start](../START-HERE.md) · [← Phase 07 — Algorithms and data structures](README.md) · [📋 Phase 07](README.md) · [07.2 Analyzing time and space →](02-analyzing-time-and-space.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
