<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.1 What an algorithm is, and Big-O notation](01-algorithms-and-big-o.md) · [📋 Phase 07](README.md) · [07.3 Arrays and vectors under the hood →](03-arrays-and-vectors-under-the-hood.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/02-analyzing-time-and-space.md) <!-- nav -->

# Lesson 07.2 — Analyzing time and space

> **Phase 07 — Algorithms and data structures** · Lesson 2 of 20 · ⏱ about 135 minutes

## What you will learn

- A recipe for finding the time complexity of code: which shapes of code cost what, and how to spot a hidden loop.
- The difference between best, worst, average and amortized cost.
- How to count the memory an algorithm needs (space complexity).
- The time/memory trade-off, with a real example and a real benchmark.
- Why a "slower" algorithm can win on small inputs.

## Before you start

- You finished [Lesson 07.1 — What an algorithm is, and Big-O notation](01-algorithms-and-big-o.md) and have the package `code/p07-algorithms`.
- You know `HashSet` ([Lesson 04.9](../04-structs-enums-collections-errors/09-hash-maps.md)) and the basics of `criterion` ([Lesson 05.12](../05-rust-intermediate/12-benchmarking-basics.md)).

## New words

| Word | Meaning |
|---|---|
| **average case** | The cost for a typical input, averaged over many inputs. |
| **amortized cost** | The cost per operation when you add up a long series of operations. A rare expensive operation is "paid" by the many cheap ones. |
| **auxiliary space** | The extra memory an algorithm needs on top of its input. When people say "space complexity", this is usually what they count. |
| **in place** | Working inside the input's own memory, with only a few extra variables (`O(1)` auxiliary space). |
| **trade-off** | Gaining one good thing by paying with another. Here: speed for memory. |
| **precomputation** | Work done once, before the questions come, so that every question becomes cheap. |
| **prefix sums** | A vector whose item `k` is the sum of the first `k` items of the input. It answers "what is the sum of this range?" in one subtraction. |
| **crossover point** | The input size at which one algorithm stops being faster and the other one takes over. |

## 1. A recipe for counting time

You do not need to count every step as in Lesson 07.1. Look at the **shape** of the code:

| Shape of the code | Time |
|---|---|
| a few simple statements, an index `v[i]` | `O(1)` |
| a loop over all `n` items, simple body | `O(n)` |
| a loop inside a loop, both over about `n` items | `O(n²)` |
| a loop that halves (or doubles) its counter | `O(log n)` |
| one block **after** another | the costs **add**: the biggest wins |
| one block **inside** another | the costs **multiply** |
| an `if` / `else` | the more expensive branch (worst case) |
| a call to a function | the cost of that function |

And one trap: **a method call can hide a loop.** `v.contains(&x)` looks like one step but it walks through
the vector: it is `O(n)`. `v.insert(0, x)` shifts every item: `O(n)`. `v.iter().sum()` is `O(n)`. Put such a
call inside a loop of `n` items and you get `O(n²)` without writing a second `for`. Here is a function with
all four kinds of parts (`examples/e02_shapes.rs`):

```rust
fn shapes(v: &[i64], other: &[i64]) -> i64 {
    let first = v[0]; // O(1)
    let total: i64 = v.iter().sum(); // O(n)

    let mut bigger_pairs = 0; // O(n²): n times n comparisons
    for a in v {
        for b in v {
            if a < b {
                bigger_pairs += 1;
            }
        }
    }

    let mut shared = 0; // looks like O(n), but `contains` is a hidden loop
    for x in v {
        if other.contains(x) {
            shared += 1;
        }
    }

    first + total + bigger_pairs + shared
}

fn main() {
    let v = [4, 8, 15, 16, 23, 42];
    let other = [8, 16, 32];
    println!("{}", shapes(&v, &other));
}
```

**What happens:** `v[0]` is `O(1)` and `v.iter().sum()` is `O(n)`. The double loop compares every item with every item:
`O(n²)`. The last loop looks like `O(n)`, but `other.contains(x)` is a loop over `other`, so with `m = other.len()` it is
`O(n * m)`. The parts follow one another, so the costs add: `O(1) + O(n) + O(n²) + O(n * m)`. Keep the biggest parts:
`O(n² + n * m)`, which is `O(n²)` when `other` is about as long as `v`. (The program prints `129`; the output is not the
point, reading the code like this is.)

## 2. Best, worst, average, amortized

The same function can cost different amounts on different inputs of the same size `n`.

- **Worst case**: the most expensive input. This is the default meaning of Big-O (Rule 3 of Lesson 07.1).
- **Best case**: the cheapest input. Rarely useful alone, but "best case `O(n)`" for a sorting algorithm
  tells you that sorted input is cheap, which can matter (Lesson 07.8).
- **Average case**: the typical input. For linear search with the target at a random position it is about
  `n / 2` comparisons, which is still `O(n)`: the constant `1/2` is dropped.
- **Amortized**: the cost per operation over a long series of operations. `Vec::push` is the classic example. Most
  pushes write one item, which is cheap. Sometimes the vector is full: it allocates a bigger block and copies all
  `n` items, which costs `O(n)`. But the vector doubles its capacity each time (Lesson 04.7), so these expensive
  pushes are rare: over `n` pushes the copying adds up to less than `2n` item moves. So one push costs `O(1)`
  *amortized*. Lesson 07.3 proves this with a counter.

> ⚠️ **Common mistake:** "Amortized `O(1)`" does not say that every single push is fast. One push can take
> `O(n)`. If a program must answer every request within a strict time limit (a game frame, a trading system),
> a rare slow operation can matter, and you reserve space in advance with `Vec::with_capacity`.

## 3. Space complexity

Time is not the only price. **Space complexity** says how the memory grows with `n`. We count the **auxiliary
space**: the extra memory the algorithm allocates, not the input it receives (the caller already owns that).

- `largest` from Lesson 07.1 uses a few variables: `O(1)` auxiliary space. It works **in place**.
- `v.iter().map(...).collect::<Vec<_>>()` builds a new vector with one item for each input item: `O(n)`.
- An algorithm that builds a table of `n` by `n` cells uses `O(n²)` memory. At `n = 100,000` that is ten
  billion cells: the program is killed long before the time matters.

To feel the numbers, run `examples/e02_space.rs`:

```rust
use std::collections::HashSet;
use std::mem::size_of;

fn main() {
    let n = 1000;
    let items: Vec<i64> = (0..n).collect();
    let set: HashSet<i64> = items.iter().copied().collect();

    println!("one i64 takes {} bytes", size_of::<i64>());
    println!(
        "Vec<i64> with {} items: about {} bytes of item data",
        items.len(),
        items.capacity() * size_of::<i64>()
    );
    println!(
        "HashSet<i64> with {} items has room for {} items",
        set.len(),
        set.capacity()
    );
}
```

```text
one i64 takes 8 bytes
Vec<i64> with 1000 items: about 8000 bytes of item data
HashSet<i64> with 1000 items has room for 1792 items
```

A vector of 1,000 `i64` numbers needs about 8,000 bytes, because each number takes 8 bytes. A hash set with the
same 1,000 numbers has room for 1,792 items (the set keeps some slots empty on purpose, so lookups stay fast; Lesson 07.6
explains why) and needs noticeably more memory than the vector. The exact figures depend on the
implementation; the lesson is that "`O(n)` memory" can still be a large `n` times a large constant.

## 4. The trade-off: time against memory

A question: *is there a pair of items in a slice whose sum is `target`?* For example, in `[8, 3, 5, 2]` with
target `10` the answer is yes (`8 + 2`). The first solution compares every pair:

```rust
/// Is there a pair of items (at two different positions) whose sum is `target`?
/// Compares every pair: O(n²) time, O(1) extra memory.
pub fn pair_sum_quadratic(items: &[i64], target: i64) -> bool {
    for i in 0..items.len() {
        for j in (i + 1)..items.len() {
            if items[i].checked_add(items[j]) == Some(target) {
                return true;
            }
        }
    }
    false
}
```

This is `O(n²)` time and `O(1)` space. (`checked_add` returns `None` instead of overflowing, see the security note
below.) The second solution trades memory for speed. For each item `x` it asks: *have I already seen the number
`target - x`?* A hash set answers in `O(1)` on average:

```rust
/// The same question. Remembers the items already seen in a hash set:
/// O(n) time on average, O(n) extra memory.
pub fn pair_sum_hashset(items: &[i64], target: i64) -> bool {
    let mut seen = HashSet::new();
    for &x in items {
        let found = target
            .checked_sub(x)
            .is_some_and(|needed| seen.contains(&needed));
        if found {
            return true;
        }
        seen.insert(x);
    }
    false
}
```

**What happens:**

1. `seen` holds all items before the current one. At the start it is empty.
2. For the item `x` the partner we need is `target - x`. If `seen` contains it, we found a pair and return `true`.
3. Otherwise we remember `x` and go on. Each item is looked at once, and each lookup and insert is `O(1)`
   on average, so the time is `O(n)`. The set can grow to `n` items, so the space is `O(n)`.
4. The check happens *before* the insert, so an item is never paired with itself. `[5]` with target `10` is `false`,
   but `[5, 5]` is `true`.

| | time | auxiliary space |
|---|---|---|
| `pair_sum_quadratic` | `O(n²)` | `O(1)` |
| `pair_sum_hashset` | `O(n)` on average | `O(n)` |

There is no free lunch: one version is cheap in memory and expensive in time, the other is the opposite. Which is
better depends on `n`, on how much memory you have, and on how often you ask. A hash set also has a worst
case of `O(n)` per operation when many keys collide (Lesson 07.6), which is why we say "on average".

> 🔒 **Security:** `a + b` on `i64` panics in a debug build (and silently wraps in a release build) when the result
> does not fit. If the numbers come from outside, a wrong answer or a crash is an attack surface. That is why the code
> uses `checked_add` and `checked_sub` and why a test uses `i64::MAX` and `i64::MIN` (Lesson 02.2).

The tests for these functions come at the end of the next section, after the last piece of the module.

## 5. Precomputation: pay once, answer fast

Another trade of memory for time: *do work once so that every later question is cheap.* Suppose we have a long
slice and ask many times: "what is the sum of `items[from..to]`?" (`from..to` is a range: it includes `from` and
excludes `to`.) Computing each sum from scratch costs `O(to - from)` per question. With **prefix sums** we spend
`O(n)` time and `O(n)` memory once, and every question costs `O(1)`:

```rust
/// The sum of `items[from..to]`, computed from scratch: O(to - from) per question.
pub fn range_sum_naive(items: &[i64], from: usize, to: usize) -> i64 {
    items[from..to].iter().sum()
}

/// Prepared once in O(n) time and O(n) memory; then every range sum costs O(1).
pub struct PrefixSums {
    // sums[k] is the sum of the first k items, so sums[0] == 0 and sums.len() == n + 1
    sums: Vec<i64>,
}

impl PrefixSums {
    pub fn new(items: &[i64]) -> PrefixSums {
        let mut sums = Vec::with_capacity(items.len() + 1);
        sums.push(0);
        for &x in items {
            let last = sums[sums.len() - 1];
            sums.push(last + x);
        }
        PrefixSums { sums }
    }

    /// The sum of `items[from..to]`.
    pub fn range_sum(&self, from: usize, to: usize) -> i64 {
        self.sums[to] - self.sums[from]
    }
}
```

**What happens:**

1. `sums[k]` is the sum of the first `k` items. `sums[0]` is 0 (no items), so the vector has `n + 1` entries.
2. `new` builds it in one pass: each entry is the previous entry plus one more item.
3. The sum of `items[from..to]` is (the first `to` items) minus (the first `from` items): one subtraction.

```text
items:  [ 3, -1,  4,  1, -5 ]
sums:   [ 0,  3,  2,  6,  7,  2 ]        sums[k] = sum of the first k items
sum of items[1..4] = sums[4] - sums[1] = 7 - 3 = 4      (that is -1 + 4 + 1)
```

Now the tests, at the bottom of `src/analysis.rs`. The first one is exhaustive: it tries all 256 vectors of length 4
with items from -1 to 2 and every target from -3 to 4, and requires the two pair-sum functions to agree. The second
checks the "two different positions" rule, the third uses the extreme values `i64::MAX` and `i64::MIN`, and the
last compares `range_sum` with the naive sum for **every** range of an 8-item slice.

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn both_pair_sums_agree_on_all_small_inputs() {
        // all 256 vectors of length 4 with items from -1 to 2, and targets from -3 to 4
        for a in -1..=2 {
            for b in -1..=2 {
                for c in -1..=2 {
                    for d in -1..=2 {
                        let v = [a, b, c, d];
                        for target in -3..=4 {
                            assert_eq!(
                                pair_sum_quadratic(&v, target),
                                pair_sum_hashset(&v, target),
                                "items {v:?}, target {target}"
                            );
                        }
                    }
                }
            }
        }
    }

    #[test]
    fn pair_sum_needs_two_different_positions() {
        assert!(!pair_sum_quadratic(&[5], 10)); // 5 + 5 would reuse one item
        assert!(!pair_sum_hashset(&[5], 10));
        assert!(pair_sum_quadratic(&[5, 5], 10));
        assert!(pair_sum_hashset(&[5, 5], 10));
    }

    #[test]
    fn pair_sum_survives_extreme_values() {
        let v = [i64::MAX, i64::MAX, i64::MIN];
        assert!(!pair_sum_quadratic(&v, 0));
        assert!(!pair_sum_hashset(&v, 0));
        assert!(pair_sum_quadratic(&v, -1)); // MAX + MIN
        assert!(pair_sum_hashset(&v, -1));
    }

    #[test]
    fn prefix_sums_match_the_naive_sum_for_every_range() {
        let items = [3, -1, 4, 1, -5, 9, 2, 6];
        let prefix = PrefixSums::new(&items);
        for from in 0..=items.len() {
            for to in from..=items.len() {
                assert_eq!(
                    prefix.range_sum(from, to),
                    range_sum_naive(&items, from, to)
                );
            }
        }
    }
}
```

Run `cargo test analysis`:

```text
running 4 tests
test analysis::tests::prefix_sums_match_the_naive_sum_for_every_range ... ok
test analysis::tests::pair_sum_needs_two_different_positions ... ok
test analysis::tests::pair_sum_survives_extreme_values ... ok
test analysis::tests::both_pair_sums_agree_on_all_small_inputs ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 5 filtered out; finished in 0.00s
```

(The five tests of Lesson 07.1 are filtered out by the name `analysis`.) Use prefix sums when the data does not change and
the questions are many. With 100,000 questions on 1,000,000 items the naive way does up to 10¹¹ additions; prefix sums do
10⁶ plus 10⁵.

## 6. Measuring the trade-off

Big-O says `pair_sum_quadratic` loses for large `n`. Does it lose for *every* `n`? Let us measure. We use
`criterion` as in Lesson 05.12. Add it and register the benchmark:

```bash
cargo add --dev criterion@0.8
mkdir benches
```

Add to the end of `Cargo.toml`:

```toml
[[bench]]
name = "analysis"
harness = false
```

Create `benches/analysis.rs`:

```rust
use criterion::{BenchmarkId, Criterion, criterion_group, criterion_main};
use p07_algorithms::analysis::{pair_sum_hashset, pair_sum_quadratic};
use std::hint::black_box;

fn bench_pair_sum(c: &mut Criterion) {
    let mut group = c.benchmark_group("pair_sum (worst case)");
    for n in [8usize, 64, 512, 4096] {
        // the items are 0, 1, ..., n-1 and the target is -1: there is no pair,
        // so both functions must look at everything
        let items: Vec<i64> = (0..n as i64).collect();
        group.bench_with_input(BenchmarkId::new("quadratic", n), &items, |b, items| {
            b.iter(|| pair_sum_quadratic(black_box(items), black_box(-1)))
        });
        group.bench_with_input(BenchmarkId::new("hashset", n), &items, |b, items| {
            b.iter(|| pair_sum_hashset(black_box(items), black_box(-1)))
        });
    }
    group.finish();
}

criterion_group!(benches, bench_pair_sum);
criterion_main!(benches);
```

**What happens:**

1. For each size `n` the input is `0, 1, ..., n-1` and the target is `-1`. No pair of non-negative numbers adds up
   to -1, so both functions must look at everything: this is the worst case.
2. `BenchmarkId::new("quadratic", n)` gives each measurement a name with the size, so the report shows the series.
3. `black_box` hides the arguments from the optimizer (Lesson 05.12).

Run it (a short run, about one minute; the name after `--bench` is needed because the `criterion` options
would confuse the normal test runner):

```bash
cargo bench --bench analysis -- --warm-up-time 1 --measurement-time 3
```

```text
pair_sum (worst case)/quadratic/8
                        time:   [12.774 ns 12.833 ns 12.904 ns]
pair_sum (worst case)/hashset/8
                        time:   [250.53 ns 251.07 ns 251.69 ns]
pair_sum (worst case)/quadratic/64
                        time:   [871.47 ns 874.23 ns 877.10 ns]
pair_sum (worst case)/hashset/64
                        time:   [2.2221 µs 2.2289 µs 2.2380 µs]
pair_sum (worst case)/quadratic/512
                        time:   [52.690 µs 53.051 µs 53.540 µs]
pair_sum (worst case)/hashset/512
                        time:   [18.465 µs 18.536 µs 18.616 µs]
pair_sum (worst case)/quadratic/4096
                        time:   [3.2139 ms 3.2285 ms 3.2447 ms]
pair_sum (worst case)/hashset/4096
                        time:   [149.49 µs 149.97 µs 150.47 µs]
```

(Criterion prints more lines, such as warm-up progress and outlier counts; they are left out. The numbers come from
an Apple M4 Pro with Rust 1.100; **your numbers will be different**, look at the ratios.)

**How to read it:**

1. **Quadratic grows by about 60 times** each time `n` grows 8 times (874 ns, 53 µs, 3.2 ms). That is close to
   `8² = 64`: the signature of `O(n²)`.
2. **The hash set grows by about 8 times** (2.2 µs, 18.5 µs, 150 µs). That is the signature of `O(n)`.
3. **For `n = 8` and `n = 64` the "bad" algorithm wins**: 13 ns against 251 ns. Creating a hash set, hashing every
   number and allocating memory has a large constant cost. For tiny inputs the double loop is faster.
4. The **crossover point** lies between 64 and 512 on this computer. At `n = 4096` the hash set is 21 times faster.

> ⚡ **Performance:** Big-O tells you what happens when `n` grows. It does not tell you where the crossover is. If your
> `n` is always small (say, under 50), the simple algorithm may be the best one. If `n` can be large, the growth rate
> wins. Measure with realistic sizes ([Lesson 05.12](../05-rust-intermediate/12-benchmarking-basics.md)).

## Common mistakes

**1. Forgetting the hidden loop.** `for x in &a { if b.contains(x) { ... } }` is `O(a.len() * b.len())`, not
`O(n)`. Fix: put `b` into a `HashSet` first; then the loop is `O(a.len() + b.len())`.

**2. Counting only time.** An algorithm that is fast but needs `O(n²)` memory can crash the machine. Always ask both
questions: how long, and how much memory.

## Try it yourself

1. Add `pub mod analysis;` to `src/lib.rs` and type the code of sections 4 and 5 into `src/analysis.rs`.
2. Run `cargo test`, `cargo clippy` and `cargo fmt`.
3. Add the benchmark of section 6, run it, and write the three most surprising numbers in a comment at the top of
   `benches/analysis.rs`.
4. Commit: `git add code/p07-algorithms && git commit -m "feat(p07): analysis module, time/space trade-offs"`.

## Exercises

### Exercise 1 — Time and space (easy)

Give the time and the auxiliary space of each piece of code (`v` has `n` items):

```rust
let doubled: Vec<i64> = v.iter().map(|x| x * 2).collect();   // (a)

let mut count = 0;                                             // (b)
for x in &v { if v.contains(&(x + 1)) { count += 1; } }

let mut k = 1;                                                 // (c)
while k < v.len() { k *= 2; }
```

<details>
<summary>Hint</summary>

Ask: how many times does the loop run, what does each pass cost (look for hidden loops), and what is allocated?

</details>

<details>
<summary>Answer</summary>

(a) time `O(n)`, space `O(n)`: the new vector. (b) time `O(n²)` because `contains` is `O(n)` inside a loop of `n`;
space `O(1)`. (c) time `O(log n)`: `k` doubles until it reaches `n`; space `O(1)`.

</details>

### Exercise 2 — Edge cases of prefix sums (easy)

Write a test for `PrefixSums` with an empty slice, a one-item slice and an empty range (`from == to`).

<details>
<summary>Hint</summary>

For an empty input the vector `sums` still has one entry, `[0]`, so `range_sum(0, 0)` works and is 0.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn prefix_sums_edge_cases() {
        let empty = PrefixSums::new(&[]);
        assert_eq!(empty.range_sum(0, 0), 0);
        let one = PrefixSums::new(&[7]);
        assert_eq!(one.range_sum(0, 1), 7);
        assert_eq!(one.range_sum(1, 1), 0); // an empty range
    }
```

Why it works: `new(&[])` builds `sums = [0]`; `range_sum(0, 0)` is `0 - 0`. For `[7]`, `sums = [0, 7]`, and the range `1..1`
is `7 - 7 = 0`.

</details>

### Exercise 3 — Duplicates in O(n) (medium)

In Lesson 07.1 `has_duplicate_counted` compared every pair. Write `has_duplicate_hashset(items: &[i32]) -> bool` with a
`HashSet`, test it against `has_duplicate_counted` on all vectors of length 4 over 0 to 3, and give the time and space of both.

<details>
<summary>Hint</summary>

`HashSet::insert` returns `false` if the value was already in the set. That is the duplicate signal.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn has_duplicate_hashset(items: &[i32]) -> bool {
    let mut seen = HashSet::new();
    for &x in items {
        if !seen.insert(x) {
            return true; // `insert` returns false when the value was already there
        }
    }
    false
}
```

```rust
    #[test]
    fn hashset_duplicates_agree_with_the_quadratic_version() {
        for a in 0..4 {
            for b in 0..4 {
                for c in 0..4 {
                    for d in 0..4 {
                        let v = [a, b, c, d];
                        assert_eq!(has_duplicate_hashset(&v), has_duplicate_counted(&v).0);
                    }
                }
            }
        }
    }
```

Time `O(n)` on average and space `O(n)`; the pair version is `O(n²)` time and `O(1)` space: the same trade-off as in section 4.

</details>

### Exercise 4 — The best window (hard)

Write `best_window_sum(items: &[i64], k: usize) -> Option<i64>`: the largest sum of `k` consecutive items, or `None` if
`k` is 0 or larger than the length. Use `PrefixSums`. Give time and space. Can the space be `O(1)`?

<details>
<summary>Hint</summary>

The window that starts at `start` covers `start..start + k`. Try every `start` from 0 to `len - k` and take the largest
`range_sum`.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn best_window_sum(items: &[i64], k: usize) -> Option<i64> {
    if k == 0 || k > items.len() {
        return None;
    }
    let prefix = PrefixSums::new(items);
    let mut best = prefix.range_sum(0, k);
    for start in 1..=(items.len() - k) {
        best = best.max(prefix.range_sum(start, start + k));
    }
    Some(best)
}
```

Time `O(n)` (one pass to build the sums, then `n - k + 1` constant-time queries) and space `O(n)`. Yes, `O(1)` space is
possible: keep a running sum, add the item that enters the window and subtract the one that leaves. This is the
*sliding window* technique of Lesson 07.15.

</details>

## Quiz

1. What is the time of a loop over `n` items that calls `v.contains(&x)` in every pass?

<details>
<summary>Answer</summary>

`O(n²)` (more exactly `O(n * v.len())`): `contains` is a loop over the vector, hidden inside the loop.

</details>

2. What does "amortized `O(1)`" mean for `Vec::push`?

<details>
<summary>Answer</summary>

Over a long series of pushes the average cost of one push is constant. A single push can be slow (when the vector must grow
and copy its items), but this happens so rarely that it does not change the average.

</details>

3. Why can `pair_sum_quadratic` beat `pair_sum_hashset` for 8 items?

<details>
<summary>Answer</summary>

Big-O hides constants. Building a hash set (allocation, hashing) costs far more per step than comparing two numbers, so for
tiny inputs the cheap double loop wins. The hash set wins only after the crossover point.

</details>

## Summary

- Read code by its shape: loops multiply, sequential blocks add, an `if` costs its worst branch, a call costs the
  function it calls. Watch for hidden loops (`contains`, `insert(0, ..)`, `sum`).
- Cases: worst (the default), best, average, and amortized (the cost per operation over a long series).
- Space complexity counts the auxiliary memory; "in place" means `O(1)` of it.
- Time and memory trade against each other: a hash set made `O(n²)` into `O(n)` at the price of `O(n)` memory;
  prefix sums spend `O(n)` once to make queries `O(1)`.
- On small inputs the simple algorithm can win. The crossover point must be measured.
- Test fast algorithms against slow, obviously correct ones, including extreme values (overflow).

## Sources and further reading

- [std::collections — Performance](https://doc.rust-lang.org/std/collections/index.html#performance) — the cost of each operation of the standard collections.
- [Vec — Guarantees](https://doc.rust-lang.org/std/vec/struct.Vec.html#guarantees) — what the documentation promises about the capacity and the growth of a vector (amortized `O(1)` push).
- [HashSet::insert](https://doc.rust-lang.org/std/collections/struct.HashSet.html#method.insert) — returns whether the value was newly inserted.
- [i64::checked_add](https://doc.rust-lang.org/std/primitive.i64.html#method.checked_add) — overflow-safe addition.
- [Criterion.rs documentation — comparing functions and input sizes](https://bheisler.github.io/criterion.rs/book/user_guide/comparing_functions.html) — how `bench_with_input` and `BenchmarkId` work.

---

[🏠 Start](../START-HERE.md) · [← 07.1 What an algorithm is, and Big-O notation](01-algorithms-and-big-o.md) · [📋 Phase 07](README.md) · [07.3 Arrays and vectors under the hood →](03-arrays-and-vectors-under-the-hood.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
