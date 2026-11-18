<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.14 Shortest paths and topological sort](14-shortest-paths-and-topological-sort.md) · [📋 Phase 07](README.md) · [07.16 Dynamic programming →](16-dynamic-programming.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/15-two-pointers-and-sliding-window.md) <!-- nav -->

# Lesson 07.15 — Two pointers and sliding window

> **Phase 07 — Algorithms and data structures** · Lesson 15 of 20 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- The *two pointers* technique: two indexes that walk through the data, instead of two nested loops.
- Two shapes of it: pointers that move towards each other, and pointers that move in the same direction.
- The *sliding window*: a block of neighbouring items with a running sum.
- Why these loops are O(n), not O(n²), and when the technique does **not** work.
- How to test a clever algorithm against a slow, obviously correct one on thousands of inputs.

## Before you start

- You finished [Lesson 07.14 — Shortest paths and topological sort](14-shortest-paths-and-topological-sort.md).
- You know Big-O notation and how to count the work of a loop ([Lesson 07.1](01-algorithms-and-big-o.md), [Lesson 07.2](02-analyzing-time-and-space.md)).
- You have the library package `code/p07-algorithms` from the earlier lessons (one module per topic, tests next to the code, benchmarks in `benches/`). This lesson adds the module `src/two_pointers.rs` (and `pub mod two_pointers;` in `src/lib.rs`) and one benchmark. `criterion` was introduced in [Lesson 05.12](../05-rust-intermediate/12-benchmarking-basics.md).

## New words

| Word | Meaning |
|---|---|
| **two pointers** | A technique with two indexes (positions) into the same data. Each step moves one of them, and they never move back, so the work is O(n). |
| **pointer** (in this lesson) | An index: a plain `usize` number that says "I am at position 3". It is not a memory address and not a Rust reference. |
| **sliding window** | A block of neighbouring items, described by a left and a right index. The right index moves forward to add an item, the left index moves forward to drop an item. |
| **in place** | Changing the data you were given, using only a few extra variables (O(1) extra space), instead of building a new vector. |
| **invariant** | A sentence about a loop that is true at the start of every step, for example "everything before `write` is already finished". Invariants are how you convince yourself that a loop is correct. |
| **amortized** | The cost averaged over the whole run: one step may be expensive, but the total is small. |
| **pseudo-random** | Numbers that look random but come from a fixed formula and a starting number (the *seed*). The same seed gives the same numbers, so a failing test can be repeated. |

## 1. Pointers from both ends

**The problem.** A slice of numbers is **sorted** from small to large. Find two *different* positions whose numbers add up to `target`.

Trying every pair needs two nested loops: O(n²). For 100,000 numbers that is about five billion pairs. The sorted order allows something much better. Put `left` at the first (smallest) item and `right` at the last (largest) item, and look at the sum:

- **Too small:** the number at `left` is the smallest one left, and even with the largest partner the sum is too small. It cannot be in any answer. Drop it: `left += 1`.
- **Too big:** the number at `right` cannot be in any answer, for the same reason. Drop it: `right -= 1`.
- **Equal:** done.

Every step throws one number away for good, so there are at most `n - 1` steps.

```text
target = 13          index:  0  1  2  3  4   5
                     value:  1  3  4  6  9  11

step 1   L=0 R=5     1 + 11 = 12   too small  -> L moves right
         [1] 3  4  6  9 [11]
step 2   L=1 R=5     3 + 11 = 14   too big    -> R moves left
          1 [3] 4  6  9 [11]
step 3   L=1 R=4     3 +  9 = 12   too small  -> L moves right
          1 [3] 4  6 [9] 11
step 4   L=2 R=4     4 +  9 = 13   found!
          1  3 [4] 6 [9] 11
```

```rust
/// Looks for two different positions `i < j` of a SORTED slice with
/// `sorted[i] + sorted[j] == target`. Time O(n), extra space O(1).
pub fn pair_with_sum(sorted: &[i32], target: i32) -> Option<(usize, usize)> {
    if sorted.len() < 2 {
        return None;
    }
    let target = i64::from(target);
    let mut left = 0;
    let mut right = sorted.len() - 1;
    while left < right {
        // i64, so that two big i32 numbers cannot overflow
        let sum = i64::from(sorted[left]) + i64::from(sorted[right]);
        if sum == target {
            return Some((left, right));
        } else if sum < target {
            left += 1; // we need a bigger sum: take a bigger left number
        } else {
            right -= 1; // we need a smaller sum: take a smaller right number
        }
    }
    None
}
```

**What happens:**

1. A slice with fewer than two items has no pair. The early return also protects `sorted.len() - 1` on the next lines (see "Common mistakes").
2. `left` and `right` are `usize` indexes at the two ends. The loop runs while `left < right`, so the pointers never meet on the same item (we need two *different* positions).
3. The sum is computed in `i64`: two `i32` values near `i32::MAX` would overflow an `i32`, and a debug build would panic.
4. Too small moves `left`, too big moves `right`, equal returns the pair. If the pointers meet, every number was dropped without a match: `None`.

> 🔒 **Security:** Arithmetic on numbers you did not choose (file contents, user input) can overflow. Widening to a bigger type with `i64::from`, or using `checked_add`, costs almost nothing and removes a crash or a wrong answer.

**Time and space.** The gap between the pointers starts at `n - 1` and shrinks by one each step: at most `n - 1` steps with O(1) work each, so **O(n) time**. Only two indexes are stored: **O(1) space**. The slow version, which does not need sorted input, is:

```rust
/// The same question, answered with two nested loops. Time O(n * n).
pub fn pair_with_sum_slow(values: &[i32], target: i32) -> Option<(usize, usize)> {
    for i in 0..values.len() {
        for j in (i + 1)..values.len() {
            if i64::from(values[i]) + i64::from(values[j]) == i64::from(target) {
                return Some((i, j));
            }
        }
    }
    None
}
```

> 🧪 **Testing:** A clever algorithm is not trusted until it agrees with the slow one on thousands of small random inputs. The random numbers come from the *pseudo-random* generator `Rng` in `src/testutil.rs` ([Lesson 07.3](03-arrays-and-vectors-under-the-hood.md)). The tests of this lesson and of the next ones need numbers from a range that can be negative, so first add one method inside `impl Rng` in `src/testutil.rs`, below `below`:

```rust
    /// A number from `lo` to `hi`, both included. `lo` must not be bigger than `hi`.
    pub fn range(&mut self, lo: i64, hi: i64) -> i64 {
        lo + (self.next_u64() % (hi - lo + 1) as u64) as i64
    }
```

Then put this at the end of `src/two_pointers.rs`. The same `Rng` is used by all tests of this lesson.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    #[test]
    fn pair_small_examples() {
        let v = [1, 3, 4, 6, 9, 11];
        assert_eq!(pair_with_sum(&v, 13), Some((2, 4)));
        assert_eq!(pair_with_sum(&v, 100), None);
        assert_eq!(pair_with_sum(&[], 1), None);
        assert_eq!(pair_with_sum(&[5], 10), None);
        assert_eq!(pair_with_sum(&[i32::MAX, i32::MAX], i32::MAX), None);
    }

    #[test]
    fn pair_agrees_with_slow_version() {
        let mut rng = Rng::new(12345);
        for _ in 0..2000 {
            let len = rng.range(0, 12) as usize;
            let mut v: Vec<i32> = (0..len).map(|_| rng.range(-20, 20) as i32).collect();
            v.sort();
            let target = rng.range(-30, 30) as i32;
            let fast = pair_with_sum(&v, target);
            let slow = pair_with_sum_slow(&v, target);
            assert_eq!(
                fast.is_some(),
                slow.is_some(),
                "v = {v:?}, target = {target}"
            );
            if let Some((i, j)) = fast {
                assert!(i < j);
                assert_eq!(v[i] + v[j], target);
            }
        }
    }
}
```

**What happens:**

1. `range(lo, hi)` takes the remainder of a big random number divided by the size of the range, `hi - lo + 1`, and adds `lo`, so the result is a number from `lo` to `hi`. `Rng::new(12345)` starts the generator with the seed 12345: the same seed always gives the same sequence, so a failure can be repeated.
2. `pair_small_examples` checks the picture above and the borders: empty slice, one item, and the overflow case.
3. `pair_agrees_with_slow_version` builds 2,000 random sorted vectors of 0 to 12 numbers and asks both functions. When several pairs exist the two functions may return *different* pairs, so we compare only "is there an answer?", and then check that the fast pair really adds up. The text after the `assert_eq!` arguments shows the failing input.

```bash
cargo test two_pointers
```

```text
running 2 tests
test two_pointers::tests::pair_small_examples ... ok
test two_pointers::tests::pair_agrees_with_slow_version ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 63 filtered out; finished in 0.00s
```

(The order of the lines can differ: tests run in parallel. The 63 filtered-out tests come from the main text of Lessons 07.1 to 07.14; with the tests of the exercises your number is bigger. The lines `Running unittests` and `Doc-tests` are left out here and below.)

## 2. A reader and a writer

In the second shape both pointers start at the beginning and one runs ahead: the **reader** (`read`) looks at every item once, the **writer** (`write`) marks the end of the part that is already finished. A classic job is removing repeated neighbours from a sorted vector *in place*: `[1, 1, 2, 2, 2, 3]` becomes `[1, 2, 3]`.

```text
v = [1, 1, 2, 2, 2, 3]       W = write (last unique item), R = read

R=1   v[R]=1 equals v[W]=1   -> skip
R=2   v[R]=2 differs         -> W=1, swap(W, R):   [1  2] 1  2  2  3
R=3   v[R]=2 equals v[W]=2   -> skip
R=4   v[R]=2 equals v[W]=2   -> skip
R=5   v[R]=3 differs         -> W=2, swap(W, R):   [1  2  3] 2  2  1
end   truncate to W+1 = 3 items                    [1, 2, 3]
```

```rust
/// Removes repeated neighbours from a SORTED vector, in place.
/// `write` is the last place of the unique part, `read` scans the rest.
pub fn dedup_sorted<T: PartialEq>(v: &mut Vec<T>) {
    if v.is_empty() {
        return;
    }
    let mut write = 0;
    for read in 1..v.len() {
        if v[read] != v[write] {
            write += 1;
            v.swap(write, read);
        }
    }
    v.truncate(write + 1);
}
```

**What happens:**

1. `write` starts at 0 because the first item is always unique.
2. For every `read` from 1: if the item differs from the last unique one, the unique part grows by one place and the new item is swapped into it. `swap` needs no `Clone` and no new vector; `T: PartialEq` is only needed for `!=`.
3. `truncate(write + 1)` cuts the leftovers after the unique part.
4. **Invariant:** at the start of each step, `v[..=write]` holds the unique items seen so far, in order. The loop keeps that sentence true; when `read` reaches the end, the sentence is the answer.

Time O(n), extra space O(1). The judge here is the standard library; add to `mod tests`:

```rust
    #[test]
    fn dedup_agrees_with_std() {
        let mut rng = Rng::new(777);
        for _ in 0..1000 {
            let len = rng.range(0, 15) as usize;
            let mut mine: Vec<i64> = (0..len).map(|_| rng.range(0, 6)).collect();
            mine.sort();
            let mut expected = mine.clone();
            expected.dedup();
            dedup_sorted(&mut mine);
            assert_eq!(mine, expected);
        }
    }
```

> 💡 **Tip:** In real code use `Vec::dedup` (removes *consecutive* equal items, so it works after sorting) or `Vec::retain` (keeps the items for which a closure returns `true`). Writing the loop yourself teaches the reader/writer pattern, which comes back in the partition step of quicksort ([Lesson 07.9](09-merge-sort-and-quicksort.md)) and in merging sorted lists.

## 3. A sliding window of fixed size

**The problem.** Given numbers and a size `k`, find the largest sum of `k` neighbouring numbers.

Computing every block's sum from scratch costs k additions for each of the n blocks: O(n · k). But two neighbouring blocks share `k - 1` numbers. When the block moves one step right, one number **enters** on the right and one **leaves** on the left, so we can *update* the sum:

```text
values = [2, 1, 5, 1, 3, 2],   k = 3

[2 1 5] 1 3 2      sum = 8
 2 [1 5 1] 3 2     sum = 8 + 1 (enters) - 2 (leaves) = 7
 2  1 [5 1 3] 2    sum = 7 + 3 - 1 = 9      <- best
 2  1  5 [1 3 2]   sum = 9 + 2 - 5 = 6
```

```rust
/// The largest sum of `k` neighbouring numbers. Time O(n), space O(1).
/// Returns `None` when `k` is 0 or bigger than the slice.
pub fn max_sum_of_k(values: &[i64], k: usize) -> Option<i64> {
    if k == 0 || k > values.len() {
        return None;
    }
    let mut window: i64 = values[..k].iter().sum();
    let mut best = window;
    for i in k..values.len() {
        window += values[i]; // the number that enters on the right
        window -= values[i - k]; // the number that leaves on the left
        best = best.max(window);
    }
    Some(best)
}

/// The same answer, recomputing every window. Time O(n * k).
pub fn max_sum_of_k_slow(values: &[i64], k: usize) -> Option<i64> {
    if k == 0 || k > values.len() {
        return None;
    }
    values.windows(k).map(|w| w.iter().sum::<i64>()).max()
}
```

**What happens:**

1. `k == 0` has no meaningful answer and `k > values.len()` does not fit: both give `None`. The guard also protects `windows(k)`, which panics for `k == 0`.
2. The first window is summed once (k additions). For every later `i`, `values[i]` enters and `values[i - k]` leaves: two operations per step, whatever `k` is. Total `k + (n - k)` = O(n); space O(1).
3. The slow version uses the standard method `slice::windows(k)`, which yields every block of `k` neighbours as a sub-slice. It is fine for small data, and it is the judge of the test:

```rust
    #[test]
    fn max_sum_agrees_with_slow_version() {
        let mut rng = Rng::new(99);
        for _ in 0..1000 {
            let len = rng.range(0, 20) as usize;
            let v: Vec<i64> = (0..len).map(|_| rng.range(-50, 50)).collect();
            let k = rng.range(0, 22) as usize;
            assert_eq!(
                max_sum_of_k(&v, k),
                max_sum_of_k_slow(&v, k),
                "v = {v:?}, k = {k}"
            );
        }
    }
```

`k` goes up to 22 on vectors of up to 19 numbers, so the test also covers `k == 0` and "k too big".

> ⚡ **Performance:** How much faster is O(n) than O(n · k)? Measure it. In `Cargo.toml` make sure `criterion = "0.8"` is under `[dev-dependencies]` and add:

```toml
[[bench]]
name = "windows"
harness = false
```

Create `benches/windows.rs`:

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use p07_algorithms::two_pointers::{max_sum_of_k, max_sum_of_k_slow};
use std::hint::black_box;

fn bench_windows(c: &mut Criterion) {
    let values: Vec<i64> = (0..50_000).map(|i| (i * 7919 % 1000) - 500).collect();
    let k = 500;
    let mut group = c.benchmark_group("max_sum_of_k");
    group.bench_function("slow", |b| {
        b.iter(|| max_sum_of_k_slow(black_box(&values), black_box(k)))
    });
    group.bench_function("sliding window", |b| {
        b.iter(|| max_sum_of_k(black_box(&values), black_box(k)))
    });
    group.finish();
}

criterion_group!(benches, bench_windows);
criterion_main!(benches);
```

```bash
cargo bench --bench windows -- --warm-up-time 1 --measurement-time 3
```

```text
max_sum_of_k/slow       time:   [1.7174 ms 1.7445 ms 1.7778 ms]
max_sum_of_k/sliding window
                        time:   [26.614 µs 27.514 µs 28.580 µs]
```

(Criterion also prints progress lines and a report about outliers; they are left out. These numbers come from an Apple M4 Mac. Numbers differ on every computer, but the ratio should be similar.)

**What happens:** `--bench windows` runs only this benchmark file; without it Cargo also hands `--warm-up-time` to the unit-test program, which does not know that option. For 50,000 numbers and `k = 500`, the sliding window needs about 27 µs and the slow version about 1.7 ms: roughly 60 times faster. The slow version is quicker than its 25 million additions suggest, because the compiler turns the inner sum into very fast vector instructions. The gap grows with `k`: the sliding window does not depend on `k` at all.

## 4. A sliding window that changes its size

**The problem.** A slice of numbers that are **not negative**, and a `target`. Find the length of the *shortest* block of neighbouring numbers whose sum is **at least** `target`.

The right edge moves forward and adds numbers. As soon as the sum reaches the target, record the length, then **shrink** from the left while the sum still reaches the target, recording each length. Then continue with the right edge.

```text
values = [2, 3, 1, 2, 4, 3],  target = 7

R=3   window [2 3 1 2]  sum 8 >= 7: length 4; drop 2 -> [3 1 2] sum 6, stop
R=4   window [3 1 2 4]  sum 10 >= 7: length 4; drop 3 -> [1 2 4] sum 7 >= 7:
                        length 3; drop 1 -> [2 4] sum 6, stop
R=5   window [2 4 3]    sum 9 >= 7: length 3; drop 2 -> [4 3] sum 7 >= 7:
                        length 2 (best); drop 4 -> [3] sum 3, stop
```

```rust
/// The length of the shortest block of neighbouring numbers whose sum is at
/// least `target`. The numbers are unsigned, so the sum only grows when the
/// window grows. Time O(n), space O(1).
pub fn shortest_window_with_sum_at_least(values: &[u32], target: u64) -> Option<usize> {
    if target == 0 {
        return Some(0); // the empty block is enough
    }
    let mut left = 0;
    let mut sum: u64 = 0;
    let mut best: Option<usize> = None;
    for right in 0..values.len() {
        sum += u64::from(values[right]);
        while sum >= target {
            let len = right - left + 1;
            best = Some(best.map_or(len, |b| b.min(len)));
            sum -= u64::from(values[left]);
            left += 1;
        }
    }
    best
}
```

**What happens:**

1. For `target == 0` the empty block already works, so the answer is length 0. With a positive target the `while` loop below always stops: when the window becomes empty, `sum` is 0, which is below the target.
2. The `for` loop moves `right` over every position and adds that number to the running `sum`.
3. The `while` loop shrinks the window while it is still big enough. Each time it records `right - left + 1`, removes `values[left]` from the sum and moves `left` forward.
4. `best.map_or(len, |b| b.min(len))` means: "if `best` is `None`, use `len`; otherwise the smaller of the two".

**Is it really O(n)?** A loop inside a loop looks like O(n²). Count the *moves* instead: `right` moves exactly n times. `left` only moves forward, never back and never past `right`, so it moves at most n times in the whole run. The inner loop does not start from zero; it continues where `left` stopped. At most 2n moves in total: O(n). This is an *amortized* argument: some outer steps shrink a lot, others not at all, and the total is small.

**When does it break?** Shrinking relies on one fact: removing a number never increases the sum, and adding a number never decreases it. This is true only for numbers that are not negative. With negative numbers, "big enough, so shrink" and "too small, so grow" stop being safe decisions, and the pointers would drop windows that should have been tried. That is why the function takes `u32` and not `i32`: the type states the requirement.

The judge builds on `windows` again: the answer is the first length for which some block reaches the target.

```rust
    /// The judge: the first length for which some block reaches the target.
    fn shortest_window_slow(values: &[u32], target: u64) -> Option<usize> {
        (1..=values.len()).find(|&len| {
            values
                .windows(len)
                .any(|w| w.iter().map(|&x| u64::from(x)).sum::<u64>() >= target)
        })
    }

    #[test]
    fn shortest_window_examples() {
        assert_eq!(
            shortest_window_with_sum_at_least(&[2, 3, 1, 2, 4, 3], 7),
            Some(2)
        );
        assert_eq!(shortest_window_with_sum_at_least(&[1, 1, 1], 10), None);
        assert_eq!(shortest_window_with_sum_at_least(&[], 1), None);
        assert_eq!(shortest_window_with_sum_at_least(&[9], 9), Some(1));
        assert_eq!(shortest_window_with_sum_at_least(&[1, 2], 0), Some(0));
    }

    #[test]
    fn shortest_window_agrees_with_slow_version() {
        let mut rng = Rng::new(4242);
        for _ in 0..2000 {
            let len = rng.range(0, 15) as usize;
            let v: Vec<u32> = (0..len).map(|_| rng.range(0, 9) as u32).collect();
            let target = rng.range(1, 40) as u64;
            assert_eq!(
                shortest_window_with_sum_at_least(&v, target),
                shortest_window_slow(&v, target),
                "v = {v:?}, target = {target}"
            );
        }
    }
```

The random numbers include zeros, a tricky case for windows. Add the test from Exercise 1 and run `cargo test two_pointers`: seven tests pass.

```text
running 7 tests
test two_pointers::tests::max_sum_examples ... ok
test two_pointers::tests::pair_small_examples ... ok
test two_pointers::tests::shortest_window_examples ... ok
test two_pointers::tests::max_sum_agrees_with_slow_version ... ok
test two_pointers::tests::dedup_agrees_with_std ... ok
test two_pointers::tests::shortest_window_agrees_with_slow_version ... ok
test two_pointers::tests::pair_agrees_with_slow_version ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 63 filtered out; finished in 0.00s
```

## 5. A recipe, and what the standard library offers

| You see | Try |
|---|---|
| A **sorted** slice and a question about pairs or triples | Pointers from both ends (Section 1). |
| "Remove / move / group items **in place**" | Reader and writer (Section 2). |
| "Best block of **k** neighbours" | Fixed window with a running value (Section 3). |
| "Shortest / longest block" where adding an item can only help (or only hurt) | Variable window (Section 4). |

To design a solution, write down: what each pointer means, the invariant, when each pointer moves, and **why a pointer never has to go back**. If you cannot answer the last question, the technique probably does not apply.

| Task | Standard tool |
|---|---|
| All overlapping blocks of `k` neighbours | `slice.windows(k)` |
| Blocks that do not overlap | `slice.chunks(k)` |
| Remove consecutive duplicates / keep items by a rule | `Vec::dedup`, `Vec::retain` |
| Reverse in place | `slice.reverse()` |
| Find a position in a sorted slice | `binary_search`, `partition_point` ([Lesson 07.10](10-binary-search.md)) |

Use these when the job is exactly that. Write your own pointer loop when the window needs a running value (a sum, a count) that `windows` would recompute again and again.

## Common mistakes

**1. `len() - 1` on an empty slice.** `len()` is a `usize`, which cannot be negative. Without the guard at the top of `pair_with_sum`:

```rust
fn pair_with_sum(sorted: &[i32], target: i32) -> Option<(usize, usize)> {
    let mut left = 0;
    let mut right = sorted.len() - 1; // no check that the slice is not empty
    while left < right {
        let sum = sorted[left] + sorted[right];
        if sum == target {
            return Some((left, right));
        } else if sum < target {
            left += 1;
        } else {
            right -= 1;
        }
    }
    None
}

fn main() {
    let empty: Vec<i32> = Vec::new();
    println!("{:?}", pair_with_sum(&empty, 10));
}
```

```text
thread 'main' (4623313) panicked at src/main.rs:3:21:
attempt to subtract with overflow
```

**What it means:** `0 - 1` does not fit into `usize`, so a debug build panics. (A release build would wrap around to a huge number and fail later in a stranger way.) **Fix:** check `len() < 2` first, or use `checked_sub`. Always test the empty and the one-item input.

**2. Two pointers on data that is not sorted.** With the guard added, the same function compiles and runs, and gives a wrong answer:

```rust
fn main() {
    let unsorted = [5, 1, 9, 2];
    // 1 + 9 = 10 exists, but the data is not sorted
    println!("{:?}", pair_with_sum(&unsorted, 10));
}
```

```text
None
```

**What it means:** the pointers moved by the rules, but the rule "this number is too small to be in any answer" needs the sorted order, and no type says "sorted". **Fix:** sort first (O(n log n)), or document the requirement and add `debug_assert!(sorted.is_sorted());` at the start so tests catch the mistake.

**3. A branch that does not move a pointer, or a window on data with negative numbers.** If one path through the loop body changes neither pointer, the loop never ends: *every* path must make progress. And negative numbers break the "shrink from the left" logic of Section 4, so test a window with negative inputs before you trust it.

## Try it yourself

1. Run `cargo test two_pointers` and read the seven test names. Then break `pair_with_sum` on purpose (change `<` to `>`) and see which tests fail first, and what the message of the random test prints.
2. Change the seed of `Rng` in one test (for example `Rng::new(12345)` to `Rng::new(1)`). The tests still pass, on different inputs. Why is a fixed seed better than a truly random one for a test?
3. Run the quality check and make one commit:

```bash
cargo fmt && cargo clippy --all-targets -- -D warnings && cargo test
git add code/p07-algorithms
git commit -m "feat(p07): two pointers and sliding window"
```

## Exercises

### Exercise 1 — Test `max_sum_of_k` (easy)

Write the test `max_sum_examples` for `max_sum_of_k`. Cover `k` equal to 1, `k` equal to the length, `k` bigger than the length, `k == 0`, and a slice of only negative numbers. Start from `[2, 1, 5, 1, 3, 2]`.

<details>
<summary>Hint</summary>

Compute the expected answers by hand first. For `[2, 1, 5, 1, 3, 2]`: `k = 1` gives the biggest single number (5); `k = 6` gives the sum of everything (14). For `[-4, -1, -3]` and `k = 2` the blocks sum to `-5` and `-4`, so the best is `-4`: a result can be negative.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn max_sum_examples() {
        let v = [2, 1, 5, 1, 3, 2];
        assert_eq!(max_sum_of_k(&v, 3), Some(9));
        assert_eq!(max_sum_of_k(&v, 1), Some(5));
        assert_eq!(max_sum_of_k(&v, 6), Some(14));
        assert_eq!(max_sum_of_k(&v, 7), None);
        assert_eq!(max_sum_of_k(&v, 0), None);
        assert_eq!(max_sum_of_k(&[-4, -1, -3], 2), Some(-4));
    }
```

Why it works: the test covers the normal case, both borders of `k`, the invalid values and negative sums. A common bug is to start `best` at 0, which would wrongly give `Some(0)` for the all-negative slice; this test catches it.

</details>

### Exercise 2 — State the complexity (medium)

Give the time complexity (in terms of `n = values.len()`) and the extra space of each function, with one sentence of reason.

```rust
fn a(values: &[i32]) -> usize {
    let mut count = 0;
    for i in 0..values.len() {
        for j in 0..values.len() {
            if values[i] < values[j] {
                count += 1;
            }
        }
    }
    count
}

fn b(values: &[i32]) -> usize {
    let mut left = 0;
    let mut right = values.len();
    let mut steps = 0;
    while left < right {
        if values[left] % 2 == 0 {
            left += 1;
        } else {
            right -= 1;
        }
        steps += 1;
    }
    steps
}

fn c(values: &[i32], k: usize) -> i32 {
    let mut best = i32::MIN;
    for start in 0..=values.len().saturating_sub(k) {
        let sum: i32 = values[start..(start + k).min(values.len())].iter().sum();
        best = best.max(sum);
    }
    best
}
```

<details>
<summary>Hint</summary>

For `a`: count the loops. For `b`: how big is the gap `right - left` at the start, and how much does it shrink in each step? For `c`: how many starting positions are there and how long is each inner sum?

</details>

<details>
<summary>Answer</summary>

- `a`: time **O(n²)**, space O(1). Two nested loops over all `n` items: n · n comparisons.
- `b`: time **O(n)**, space O(1). The gap starts at `n` and shrinks by exactly one in every step (either `left` grows or `right` shrinks), so the loop runs `n` times.
- `c`: time **O(n · k)**, space O(1). About `n - k + 1` starting positions, and each inner `sum` adds up to `k` numbers. The sliding window of Section 3 removes the factor `k`.

</details>

### Exercise 3 — Three numbers that add up to a target (hard)

Write `pub fn has_triple_with_sum(values: &[i32], target: i64) -> bool` that says whether three numbers at three different positions add up to `target`. Use sorting plus two pointers, state the time complexity, and test it against a triple loop on random inputs.

<details>
<summary>Hint</summary>

Sort a copy of the input. For each `first` position, the other two numbers must add up to `target - values[first]` and must lie **after** `first`: that is Section 1 on the part of the slice after `first`.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn has_triple_with_sum(values: &[i32], target: i64) -> bool {
    let mut v = values.to_vec();
    v.sort();
    if v.len() < 3 {
        return false;
    }
    for first in 0..v.len() - 2 {
        let need = target - i64::from(v[first]);
        let mut left = first + 1;
        let mut right = v.len() - 1;
        while left < right {
            let sum = i64::from(v[left]) + i64::from(v[right]);
            if sum == need {
                return true;
            } else if sum < need {
                left += 1;
            } else {
                right -= 1;
            }
        }
    }
    false
}
```

Test (in the `tests` module):

```rust
    fn has_triple_slow(v: &[i32], target: i64) -> bool {
        let n = v.len();
        (0..n).any(|i| {
            (i + 1..n).any(|j| {
                (j + 1..n).any(|k| i64::from(v[i]) + i64::from(v[j]) + i64::from(v[k]) == target)
            })
        })
    }

    #[test]
    fn triple_agrees_with_slow() {
        let mut rng = Rng::new(9);
        for _ in 0..1000 {
            let len = rng.range(0, 8) as usize;
            let v: Vec<i32> = (0..len).map(|_| rng.range(-10, 10) as i32).collect();
            let target = rng.range(-15, 15);
            assert_eq!(
                has_triple_with_sum(&v, target),
                has_triple_slow(&v, target),
                "{v:?} {target}"
            );
        }
    }
```

Time: sorting is O(n log n), then n passes of O(n) each: **O(n²)** in total, better than the O(n³) triple loop. Extra space O(n) for the sorted copy. The test passes.

</details>

## Quiz

1. Why does `pair_with_sum` need a sorted slice?

<details>
<summary>Answer</summary>

The decision "this number is too small (or too big) to be in any answer" relies on the order. In a sorted slice, if `values[left] + values[right]` is too small, `values[left]` plus any other candidate (all at most `values[right]`) is also too small, so `left` can be dropped for good. In an unsorted slice that argument is false (Common mistake 2).

</details>

2. The shrinking `while` loop sits inside the `for` loop in `shortest_window_with_sum_at_least`. Why is the total time still O(n)?

<details>
<summary>Answer</summary>

The pointers never move back. `right` moves forward n times; `left` moves forward at most n times in the whole run, because it never passes `right`. The inner loop continues from where `left` was, so there are at most 2n pointer moves in total. The cost is amortized over the run.

</details>

3. What are the time and space of `max_sum_of_k`, compared with the version that sums each `windows(k)` block?

<details>
<summary>Answer</summary>

Sliding window: O(n) time, O(1) space. The `windows(k)` version: O(n · k) time, O(1) space. Same answer; the first updates the sum with one addition and one subtraction per step.

</details>

4. Why does `shortest_window_with_sum_at_least` take `&[u32]` and not `&[i32]`?

<details>
<summary>Answer</summary>

Shrinking is correct only if adding items never decreases the sum and removing items never increases it. That is true for non-negative numbers only. The unsigned type makes the requirement part of the signature.

</details>

## Summary

- Two pointers replace a second loop with a second index. Because the pointers never move back, the work is O(n), usually with O(1) extra space.
- Opposite ends: on a sorted slice, a sum that is too small moves the left pointer, a sum that is too big moves the right one. Same direction: a reader and a writer, for in-place filtering.
- A sliding window keeps a running value: add the item that enters, subtract the item that leaves. A variable window shrinks from the left while its condition still holds.
- A loop inside a loop is not always O(n²): count the total moves of the pointers (amortized analysis).
- The technique depends on a property (sorted data, non-negative numbers). Put it in the type or the doc comment, and test the borders: empty, one item, `k` too big.
- Test a fast algorithm against a slow, simple one on seeded random inputs. In real code prefer `windows`, `chunks`, `dedup`, `retain` and `reverse` when they do the job.

## Sources and further reading

- [The Rust Standard Library — slice](https://doc.rust-lang.org/std/primitive.slice.html) — `windows`, `chunks`, `swap`, `reverse`, `is_sorted`, `partition_point`.
- [The Rust Standard Library — Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html) — `dedup`, `retain`, `truncate`.
- [The Rust Standard Library — Option](https://doc.rust-lang.org/std/option/enum.Option.html) — `map_or`, used for the best length.
- [The Rust Programming Language — 13.2 Processing a Series of Items with Iterators](https://doc.rust-lang.org/book/ch13-02-iterators.html) — the official chapter on iterators, which `windows` and `chunks` produce.
- [Criterion.rs User Guide](https://bheisler.github.io/criterion.rs/book/) — the benchmark library used here.

---

[🏠 Start](../START-HERE.md) · [← 07.14 Shortest paths and topological sort](14-shortest-paths-and-topological-sort.md) · [📋 Phase 07](README.md) · [07.16 Dynamic programming →](16-dynamic-programming.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
