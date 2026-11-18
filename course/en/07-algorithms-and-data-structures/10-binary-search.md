<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.9 Merge sort and quicksort](09-merge-sort-and-quicksort.md) · [📋 Phase 07](README.md) · [07.11 Trees and binary search trees →](11-trees-and-binary-search-trees.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/10-binary-search.md) <!-- nav -->

# Lesson 07.10 — Binary search

> **Phase 07 — Algorithms and data structures** · Lesson 10 of 20 · ⏱ about 135 minutes

## What you will learn

- How binary search finds an item in a sorted slice in O(log n) steps, and why the slice must be sorted.
- How to write it without off-by-one errors: a half-open range, a loop invariant and the safe middle index.
- How `partition_point`, "lower bound" and "upper bound" solve many problems with one idea, including searching for an *answer* instead of an item.
- How to test it against the standard library, how much faster it is than a linear scan, and which standard methods to use.

## Before you start

- You finished [Lesson 07.9 — Merge sort and quicksort](09-merge-sort-and-quicksort.md) and [Lesson 07.8](08-simple-sorting-algorithms.md).
- You know `Option` and `Result` ([Lesson 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)) and closures ([Lesson 05.4](../05-rust-intermediate/04-closures.md)).
- In `code/p07-algorithms` create the file `src/searching.rs` and add the line `pub mod searching;` to `src/lib.rs`. The benchmark uses `criterion` as in the earlier lessons.

## New words

| Word | Meaning |
|---|---|
| **linear search** | Looking at the items one by one from the start until you find the one you want. O(n). |
| **binary search** | Finding an item in a *sorted* slice by looking at the middle item and throwing away the half that cannot contain the target. O(log n). |
| **sorted** | Arranged so that every item is not bigger than the next one. Binary search gives wrong answers on unsorted data. |
| **half-open range** | A range `lo..hi` that includes `lo` but not `hi`, like Rust's `lo..hi`. It is empty when `lo == hi`. |
| **loop invariant** | A fact that is true before and after every turn of a loop. For binary search: "if the target is present, it is inside `v[lo..hi]`". |
| **lower bound** | The index of the first item that is **not smaller** than the target (the place where the target would start). |
| **upper bound** | The index of the first item that is **bigger** than the target (the place right after the last equal item). |
| **predicate** | A function that answers true or false for an item. |
| **monotone** | Changing in only one direction. A predicate is monotone on a slice if it is true for a prefix and false for the rest. |
| **overflow** | A number operation whose result does not fit into the type. In a debug build Rust panics. |

## 1. The idea

Think of the number-guessing game: "I am thinking of a number from 1 to 100." The best first guess is 50. If the
answer is "bigger", you have thrown away half of the numbers with one question. Next guess: 75. After about 7 questions you
always find the number, because 2⁷ = 128 > 100.

**Binary search** does the same with a sorted slice. Compare the target with the **middle** item:

- equal: found;
- the middle item is smaller than the target: the target can only be in the right half;
- the middle item is bigger: the target can only be in the left half.

Looking for `23` in a slice of ten items. `lo` and `hi` are the borders of the part that can still contain the target:

```text
index:   0   1   2   3    4    5    6    7    8    9
value:   2   5   8   12   16   23   38   56   72   91

target 56:
lo=0 hi=10  mid=5  v[5]=23 < 56   throw away the left half  -> lo=6
lo=6 hi=10  mid=8  v[8]=72 > 56   throw away the right half -> hi=8
lo=6 hi=8   mid=7  v[7]=56 = 56   found at index 7  (3 probes)
```

Each probe halves the part that is left. A slice of `n` items needs at most about log₂ n probes (log₂ means "how many
times can I halve", Lesson 07.9): 10 probes for 1,000 items, 20 for a million, 30 for a billion. A linear search needs up to `n`. We write the
cost as **O(log n)**, and the extra memory is **O(1)**.

> ⚠️ **Common mistake:** Binary search works **only on sorted data**. On an unsorted slice it does not crash; it just
> answers wrongly (Common mistakes below shows it).

## 2. Writing binary search

Add this to `src/searching.rs`. It has the same result type as the standard method: `Ok(index)` when found,
`Err(index)` with the place where the item could be inserted to keep the slice sorted when it is not:

```rust
//! Binary search and friends.

use std::cmp::Ordering;

/// Binary search in a sorted slice. Returns `Ok(index)` if `target` is found, or
/// `Err(index)` with the place where it could be inserted to keep the slice sorted.
pub fn binary_search<T: Ord>(v: &[T], target: &T) -> Result<usize, usize> {
    let (mut lo, mut hi) = (0, v.len()); // the answer is inside v[lo..hi]
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        match v[mid].cmp(target) {
            Ordering::Equal => return Ok(mid),
            Ordering::Less => lo = mid + 1,
            Ordering::Greater => hi = mid,
        }
    }
    Err(lo)
}
```

**What happens:**

1. `lo..hi` is a **half-open range**: `lo` is included and `hi` is not. Start with the whole slice, `0..len`.
   **Invariant:** if the target is in the slice, its index is in `lo..hi`.
2. The loop runs while the range is not empty (`lo < hi`).
3. `mid = lo + (hi - lo) / 2` is the middle index. It is always inside the range, so `v[mid]` is safe. (Why not
   `(lo + hi) / 2`? See Common mistakes.)
4. `cmp` returns an `Ordering` (Lesson 07.9). If `v[mid]` is smaller than the target, then `mid` and everything to its left cannot be the answer,
   so `lo = mid + 1`. If it is bigger, `mid` and everything right of it are out, so `hi = mid`. Either way the range gets smaller, so the
   loop must end.
5. When the loop ends, `lo == hi`: the empty range sits exactly at the place where the target would have to be. That is the
   insertion point, so we return `Err(lo)`. For an empty slice this gives `Err(0)` immediately.

**Duplicates.** If the slice has several equal items, this function returns *one* of them (the one the probes hit first), not necessarily the first.
The standard method behaves in the same way, and says that the choice may change between Rust versions. Section 3 shows how to find
the first or the last.

## 3. One idea, many uses: `partition_point`

Look again at the loop. We only ask one question per step: "Is the middle item still *before* the place I search for?" If yes, go right,
otherwise go left. This works for **any** yes/no question that is true for a prefix of the slice and false for the rest (a *monotone predicate*).
So we can write the loop once:

```rust
/// Returns the index of the first item for which `pred` is false.
/// `pred` must be true for a (maybe empty) prefix of the slice and false for the rest.
pub fn partition_point<T, F>(v: &[T], mut pred: F) -> usize
where
    F: FnMut(&T) -> bool,
{
    let (mut lo, mut hi) = (0, v.len());
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        if pred(&v[mid]) {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    lo
}

/// The index of the first item that is not smaller than `target`.
pub fn lower_bound<T: Ord>(v: &[T], target: &T) -> usize {
    partition_point(v, |x| x < target)
}

/// The index of the first item that is bigger than `target`.
pub fn upper_bound<T: Ord>(v: &[T], target: &T) -> usize {
    partition_point(v, |x| x <= target)
}
```

For `[1, 3, 3, 3, 5]` and the target `3`:

```text
index:        0  1  2  3  4  5
value:        1  3  3  3  5
x < 3:        T  F  F  F  F        lower_bound = 1  (first index where it is false)
x <= 3:       T  T  T  T  F        upper_bound = 4
```

**What happens:**

1. The predicate `x < target` is true for the items before the target and false from the first target on. The first
   `false` is at index 1, so `lower_bound` is 1.
2. The predicate `x <= target` is true up to and including the last `3`. The first `false` is at index 4, so `upper_bound` is 4.
3. From these two numbers you get a lot for free: `upper_bound - lower_bound` is **how many times** the target occurs (3); `lower_bound == upper_bound`
   means the target is absent; `lower_bound` is the place to insert a new item before its equals and `upper_bound` the place after them.
4. The standard library has the same function as a slice method, `v.partition_point(|x| ...)`, and our tests compare with it.

### Searching for an answer, not an item

The predicate does not have to look at a slice at all. Whenever you can say "small values are fine, from some point on they are too
big", you can binary search the *range of possible answers*. Example: the integer square root of `n`, the biggest `r` with `r * r <= n`:

```rust
/// The biggest `r` with `r * r <= n`.
pub fn integer_sqrt(n: u64) -> u64 {
    // Search the numbers 0..=min(n, u32::MAX); their squares fit into u64.
    let (mut lo, mut hi) = (0u64, n.min(u32::MAX as u64) + 1);
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        if mid * mid <= n {
            lo = mid + 1; // mid is small enough, look at bigger numbers
        } else {
            hi = mid;
        }
    }
    lo - 1 // `lo` is the first number whose square is too big
}
```

**What happens:**

1. The candidates are the numbers `0..=n`, but a square root of a `u64` is never bigger than `u32::MAX` (4,294,967,295), and its square still fits in a `u64`.
   So we search `0..=min(n, u32::MAX)`. Without this limit `mid * mid` could overflow.
2. The predicate "`mid * mid <= n`" is true for small numbers and false for big ones: it is monotone. This is the same loop as `partition_point`.
3. `lo` ends as the first number whose square is **too big**, so the answer is `lo - 1`. (`lo` is at least 1: for `n = 0` the number 0 satisfies the
   predicate, so `lo` becomes 1.)
4. It needs at most 32 rounds for any `u64`. Exercise 5 uses the same trick for a bigger problem.

## 4. Testing

The standard library gives us the correct answers, so we compare. There is one subtlety: with duplicates, `Ok(i)` may differ from the standard
result. So for `Ok` we check that `v[i]` equals the target, and for `Err` we require the exact same index. Add this to the end of
`src/searching.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    /// A sorted vector of pseudo-random numbers (the `Rng` of Lesson 07.3).
    fn sorted_random(seed: u64, len: usize, max: u64) -> Vec<i32> {
        let mut rng = Rng::new(seed);
        let mut v: Vec<i32> = (0..len).map(|_| (rng.next_u64() % max) as i32).collect();
        v.sort();
        v
    }

    #[test]
    fn edge_cases() {
        let empty: [i32; 0] = [];
        assert_eq!(binary_search(&empty, &5), Err(0));
        assert_eq!(binary_search(&[5], &5), Ok(0));
        assert_eq!(binary_search(&[5], &1), Err(0));
        assert_eq!(binary_search(&[5], &9), Err(1));
        let v = [1, 3, 5, 7];
        assert_eq!(binary_search(&v, &0), Err(0));
        assert_eq!(binary_search(&v, &8), Err(4));
        assert_eq!(binary_search(&v, &4), Err(2));
    }

    #[test]
    fn matches_std_on_many_inputs() {
        for len in 0..40 {
            for seed in 1..=5 {
                for max in [5, 30, 1000] {
                    let v = sorted_random(seed, len, max);
                    for target in -1..=(max as i32 + 1) {
                        let ours = binary_search(&v, &target);
                        let theirs = v.binary_search(&target);
                        match ours {
                            // With duplicates any matching index is allowed.
                            Ok(i) => assert_eq!(v[i], target),
                            Err(i) => assert_eq!(Err(i), theirs),
                        }
                        assert_eq!(ours.is_ok(), theirs.is_ok());
                        assert_eq!(lower_bound(&v, &target), v.partition_point(|x| *x < target));
                        assert_eq!(
                            upper_bound(&v, &target),
                            v.partition_point(|x| *x <= target)
                        );
                    }
                }
            }
        }
    }

    #[test]
    fn integer_sqrt_works() {
        for n in 0..10_000u64 {
            assert_eq!(integer_sqrt(n), n.isqrt(), "n = {n}");
        }
        assert_eq!(integer_sqrt(u64::MAX), 4_294_967_295);
        assert_eq!(integer_sqrt(1 << 62), 1 << 31);
    }
}
```

**What happens:**

1. `sorted_random` makes a sorted vector with the `Rng` generator of Lesson 07.3. A small `max` such as 5 means many duplicates.
2. `edge_cases` tests the borders: the empty slice, one item, and a target smaller than everything, bigger than everything and in a gap.
3. `matches_std_on_many_inputs` tries every target from `-1` to `max + 1` (including absent values on both sides) for 600 different slices,
   and compares `binary_search`, `lower_bound` and `upper_bound` with the standard library (`partition_point` is a standard slice method).
4. `integer_sqrt_works` compares with the standard `u64::isqrt` for the first 10,000 numbers and checks two huge inputs, including the largest `u64`, where a careless
   version overflows.

```bash
cd ~/projects/rust/code/p07-algorithms
cargo test searching
```

```text
running 3 tests
test searching::tests::edge_cases ... ok
test searching::tests::integer_sqrt_works ... ok
test searching::tests::matches_std_on_many_inputs ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 33 filtered out; finished in 0.05s
```

(The filtered-out tests are the 33 tests from the main text of Lessons 07.1 to 07.9; with the tests of the exercises your number is bigger.)

> 🧪 **Testing:** Binary search is the classic place for off-by-one errors, so test the **borders**: empty, one item, a target smaller than all, bigger than all, equal to the
> first, equal to the last, and in every gap. The loop above that tries *every* target around the data does this automatically.

## 5. How much faster is O(log n)?

Create `benches/search.rs` and register it in `Cargo.toml`:

```toml
[[bench]]
name = "search"
harness = false
```

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use p07_algorithms::searching::binary_search;
use std::hint::black_box;

fn bench_search(c: &mut Criterion) {
    let n = 1_000_000;
    let data: Vec<u32> = (0..n).map(|i| i * 2).collect(); // 0, 2, 4, ...: sorted
    let target = (n - 1) * 2; // the last item: the worst case for a linear scan

    let mut group = c.benchmark_group("find the last of 1_000_000");
    group.bench_function("linear scan", |b| {
        b.iter(|| data.iter().position(|&x| x == black_box(target)))
    });
    group.bench_function("our binary_search", |b| {
        b.iter(|| binary_search(black_box(&data), &black_box(target)))
    });
    group.bench_function("std binary_search", |b| {
        b.iter(|| black_box(&data).binary_search(&black_box(target)))
    });
    group.finish();
}

criterion_group!(benches, bench_search);
criterion_main!(benches);
```

```bash
cargo bench --bench search -- --warm-up-time 1 --measurement-time 3
```

Best estimates from one run on an Apple M4 Pro Mac, release build, 2026-11-18 (your numbers will be different):

| Method | Time for one search in 1,000,000 numbers |
|---|---|
| linear scan (`position`) | 336 µs |
| our `binary_search` | 13.5 ns |
| std `binary_search` | 10.6 ns |

**How to read it:**

1. 336 µs is 336,000 ns, so binary search is about **25,000 times faster** here: 20 probes against up to a million. And the gap grows with
   `n`: a thousand times more data costs binary search only 10 more probes.
2. The standard method is a little faster than ours, but of the same kind. Our version is a fine tool.
3. The last item is the easiest target for the processor to predict (it always goes right), so real lookups of random targets are slower per probe
   (the data does not fit into the processor's fast memory). The ratio to a linear scan is still huge.

> ⚡ **Performance:** Binary search has a price: the data must be sorted. Sorting costs O(n log n) once. A linear search costs O(n) every time.
> If you search more than a few times (about log n times), sorting first and searching in a sorted slice is cheaper. If the data changes
> all the time, a sorted `Vec` is awkward (inserting costs O(n)), and a hash table or a tree (Lesson 07.11) is the better choice.

## 6. What to use in real code

| Task | Use |
|---|---|
| find an item in a sorted slice | `v.binary_search(&x)` returns `Result<usize, usize>` |
| find by a key of a struct | `v.binary_search_by_key(&key, \|item\| item.key)` |
| any comparison | `v.binary_search_by(\|item\| item.cmp(&x))` |
| first / last position, count of equals, insertion place | `v.partition_point(\|item\| ...)` |
| insert and keep the vector sorted | `let i = v.partition_point(\|y\| *y < x); v.insert(i, x);` |
| many lookups, order does not matter | `HashMap` / `HashSet` (average O(1)) |
| lookups **and** sorted order, range queries | `BTreeMap` / `BTreeSet` (O(log n)) |

A sorted `Vec` with binary search is compact and fast to read, which is why it is often the best choice for data that is built once and searched many times.

## Common mistakes

**1. Searching in unsorted data.** The standard documentation says the result of an unsorted search is "unspecified and meaningless":

```rust
fn main() {
    let v = [5, 1, 4, 2, 3]; // NOT sorted
    println!("{:?}", v.binary_search(&4));
    println!("{:?}", v.binary_search(&1));
}
```

```text
Err(5)
Ok(1)
```

The first line says that `4` is not there, but it is at index 2. The second line is correct, by luck. There is no error and no panic. If you are unsure that the slice is
sorted, check it with `v.is_sorted()` in a test or a `debug_assert!`.

**2. The middle index `(lo + hi) / 2`.** For indices of a small type, the sum `lo + hi` can be bigger than the type can hold, even if both
numbers fit:

```rust
fn middle(lo: u8, hi: u8) -> u8 {
    (lo + hi) / 2
}

fn main() {
    println!("{}", middle(100, 120));
    println!("{}", middle(200, 250));
}
```

```text
110

thread 'main' (6504865) panicked at src/main.rs:2:5:
attempt to add with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

(The number after `thread 'main'` differs on your computer.) The correct middle is 225, but 200 + 250 = 450 does not fit into a `u8` (maximum 255). The form `lo + (hi - lo) / 2` never leaves the range, because
`hi - lo` is small. With `usize` the overflow needs a slice of more than 9 quintillion items, but the safe form costs nothing, so always use it.

> 🔒 **Security:** Overflow in index arithmetic is a classic source of real bugs: a wrong index can make a program read or write the wrong place. Rust turns it into a
> panic in debug builds, but a release build wraps around silently unless you enable overflow checks. Write the safe form from the start.

**3. A range that does not shrink.** If you write `lo = mid` instead of `lo = mid + 1`, then for `lo = 3, hi = 4` the middle is 3 and `lo` stays 3 forever: an infinite loop.
Check every branch: each one must make `hi - lo` smaller.

## Try it yourself

1. Add the three functions of Sections 2 and 3 and the tests of Section 4 to `src/searching.rs`. Run `cargo test searching`.
2. In `binary_search`, change `lo = mid + 1` to `lo = mid`. Which test hangs? Stop it with Ctrl+C and change it back. (Why does it hang? See mistake 3.)
3. Write a small `examples/` program that sorts ten numbers of your choice and calls `binary_search` for a present and for an absent value. Print the results.
4. Run `cargo fmt` and `cargo clippy --all-targets`, then commit: `git add code/p07-algorithms && git commit -m "feat(p07): binary search"`.

## Exercises

### Exercise 1 — Trace by hand (easy)

For `[2, 5, 8, 12, 16, 23, 38, 56, 72, 91]` run `binary_search` for the target `10` by hand. Write down `lo`, `hi`, `mid` and the decision at every step, and the final result.

<details>
<summary>Hint</summary>

Start with `lo = 0`, `hi = 10`. The middle is `lo + (hi - lo) / 2` with integer division. A value that is absent ends in `Err(insertion index)`.

</details>

<details>
<summary>Answer</summary>

```text
lo=0 hi=10  mid=5  v[5]=23 > 10   hi=5
lo=0 hi=5   mid=2  v[2]=8  < 10   lo=3
lo=3 hi=5   mid=4  v[4]=16 > 10   hi=4
lo=3 hi=4   mid=3  v[3]=12 > 10   hi=3
lo=3 hi=3   the range is empty: return Err(3)
```

4 probes (at the indexes 5, 2, 4 and 3). The result is `Err(3)`: `10` would be inserted at index 3, between `8` and `12`.

</details>

### Exercise 2 — Test counting (easy)

Write a test that checks `upper_bound(&v, &t) - lower_bound(&v, &t)` equals the number of times `t` occurs in `v` (count it with `iter().filter(...).count()`), for many sorted
vectors with many duplicates.

<details>
<summary>Hint</summary>

Reuse `sorted_random(seed, len, 8)`: only 8 different values, so every value repeats. Loop over the targets `0..8`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn count_with_bounds() {
        for len in 0..40 {
            for seed in 1..=3 {
                let v = sorted_random(seed, len, 8);
                for target in 0..8 {
                    let count = upper_bound(&v, &target) - lower_bound(&v, &target);
                    assert_eq!(count, v.iter().filter(|&&x| x == target).count());
                }
            }
        }
    }
```

Why it works: the items equal to the target lie between the first index that is not smaller than it and the first index that is bigger than it. The distance between the two indexes is their count. The binary
searches find that count in O(log n) instead of O(n).

</details>

### Exercise 3 — State the complexity (medium)

1. How many probes does `binary_search` need at most for 1,000,000,000 items?
2. You have an unsorted `Vec` of `n` items and must answer `k` membership questions. Compare: `k` linear scans, against sorting once and then using `k` binary searches. Give both costs as Big-O.
3. When is sorting first the better plan?

<details>
<summary>Hint</summary>

For (1): how many times can you halve a billion until you reach 1? For (2), sorting costs O(n log n), and a linear scan costs O(n).

</details>

<details>
<summary>Answer</summary>

1. About **30**: 2³⁰ = 1,073,741,824 is just above a billion.
2. `k` linear scans cost **O(k · n)**. Sorting once costs **O(n log n)**, and `k` binary searches cost **O(k log n)**, in total **O((n + k) log n)**.
3. When `k` is bigger than about `log n`: then `k · n` is bigger than `(n + k) log n`. For a million items, `log n` is about 20, so with more than a few dozen questions sorting wins. For a single
   question a linear scan is cheaper than sorting.

</details>

### Exercise 4 — Sorted insert (medium)

Write `insert_sorted<T: Ord>(v: &mut Vec<T>, item: T) -> bool` that inserts `item` into the sorted vector `v` so that it stays sorted, and does nothing and returns `false` if the
item is already in. Use `binary_search` of this lesson. Write a test.

<details>
<summary>Hint</summary>

`match binary_search(v, &item)`: for `Ok(_)` the item exists; for `Err(pos)` the insertion place is `pos`, and `Vec::insert(pos, item)` puts it there (it shifts the later items right, so it is O(n)).

</details>

<details>
<summary>Answer</summary>

```rust
/// Puts `item` into the sorted vector `v`. Returns false (and changes nothing) if it is already there.
pub fn insert_sorted<T: Ord>(v: &mut Vec<T>, item: T) -> bool {
    match binary_search(v, &item) {
        Ok(_) => false,
        Err(pos) => {
            v.insert(pos, item);
            true
        }
    }
}

    #[test]
    fn insert_sorted_keeps_order_and_skips_duplicates() {
        let mut v = Vec::new();
        let input = [5, 3, 8, 3, 1, 9, 5, 0];
        let results: Vec<bool> = input.iter().map(|&x| insert_sorted(&mut v, x)).collect();
        assert_eq!(v, [0, 1, 3, 5, 8, 9]);
        assert_eq!(results, [true, true, true, false, true, true, false, true]);
    }
```

(The function goes above the `tests` module, the test goes into it.) Why it works: `Err(pos)` is exactly the place that keeps the order. Finding the place is O(log n), but `insert` has to move
the items after it, so the whole operation is O(n). For many insertions a `BTreeSet` (Lesson 07.11) is better.

</details>

### Exercise 5 — The smallest ship (hard)

Packages with the weights `weights` must be shipped **in the given order**, one ship per day, and the ship carries as much as its `capacity`. All packages must be delivered within `days` days (a day's load is a run of
packages whose total weight is at most `capacity`). Write `min_capacity(weights: &[u32], days: u32) -> u64`, the smallest capacity that works. Test it with
`[1,2,3,4,5,6,7,8,9,10]` and 5 days (answer 15), and compare with a brute force that tries every capacity on small random inputs.

<details>
<summary>Hint</summary>

If capacity `c` works, every bigger capacity works too: a monotone predicate. Binary search `c` between the heaviest package (smaller cannot carry it) and the total weight. To test `c`, walk through the packages and start a new day whenever the next one does not fit.

</details>

<details>
<summary>Answer</summary>

```rust
/// Can a ship with this `capacity` deliver all packages, in order, within `days` days?
fn can_carry(weights: &[u32], days: u32, capacity: u64) -> bool {
    let (mut used_days, mut load) = (1, 0u64);
    for &w in weights {
        if load + w as u64 > capacity {
            used_days += 1; // this package starts a new day
            load = 0;
        }
        load += w as u64;
    }
    used_days <= days
}

/// The smallest ship capacity that carries all packages, in the given order, in at most `days` days.
pub fn min_capacity(weights: &[u32], days: u32) -> u64 {
    // The answer is at least the heaviest package and at most the total weight.
    let mut lo = weights.iter().copied().max().unwrap_or(0) as u64;
    let mut hi = weights.iter().map(|&w| w as u64).sum::<u64>() + 1;
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        if can_carry(weights, days, mid) {
            hi = mid; // mid works: look for something smaller
        } else {
            lo = mid + 1; // mid is too small
        }
    }
    lo
}
```

The test goes into the `tests` module. The brute force tries every capacity from the heaviest package up to the total weight and takes the first one that works:

```rust
    #[test]
    fn min_capacity_matches_brute_force() {
        assert_eq!(min_capacity(&[1, 2, 3, 4, 5, 6, 7, 8, 9, 10], 5), 15);
        let mut rng = Rng::new(77);
        for len in 1..9 {
            for days in 1..=len as u32 {
                let weights: Vec<u32> = (0..len).map(|_| rng.below(10) as u32 + 1).collect();
                let max = *weights.iter().max().unwrap() as u64;
                let total: u64 = weights.iter().map(|&w| w as u64).sum();
                let slow = (max..=total)
                    .find(|&c| can_carry(&weights, days, c))
                    .unwrap();
                assert_eq!(min_capacity(&weights, days), slow);
            }
        }
    }
```

Why it works: the loop is the `partition_point` loop applied to the range of capacities, with the predicate "this capacity is *not* enough". One `can_carry` check costs O(n), and
there are about log₂(total weight) rounds, so the whole function is **O(n log W)** where `W` is the total weight. A brute force that tries all capacities would cost O(n · W).

</details>

## Quiz

1. Why must the slice be sorted for binary search?

<details>
<summary>Answer</summary>

Because one comparison with the middle item must tell you which half can be thrown away. That is only true if everything on the left is not bigger and everything on the right is not smaller.

</details>

2. What does `Err(i)` mean in the result of `binary_search`?

<details>
<summary>Answer</summary>

The item is not in the slice, and `i` is the index where it could be inserted to keep the slice sorted.

</details>

3. How do you find how many times a value occurs in a sorted slice in O(log n)?

<details>
<summary>Answer</summary>

`upper_bound - lower_bound`, or with the standard library `v.partition_point(|x| x <= &t) - v.partition_point(|x| x < &t)`.

</details>

4. Why is `lo + (hi - lo) / 2` better than `(lo + hi) / 2`?

<details>
<summary>Answer</summary>

`lo + hi` can overflow the number type even when both numbers fit. `hi - lo` never does, so the result stays in range.

</details>

5. Name a problem where you binary search for an answer, not for an item in a slice.

<details>
<summary>Answer</summary>

For example the integer square root, or the smallest ship capacity of Exercise 5. It works whenever "the answer is good enough" is false for small values and true from some value on (or the other way round).

</details>

## Summary

- Binary search looks at the middle of a **sorted** slice and discards half. It needs O(log n) probes and O(1) memory: 20 probes for a million items.
- Use a half-open range `lo..hi`, the invariant "the target, if present, is in `lo..hi`", the middle `lo + (hi - lo) / 2`, and make sure every branch shrinks the range.
- `Err(i)` of the result is the insertion point that keeps the slice sorted. With duplicates, `Ok(i)` may be any matching index.
- `partition_point` (first index where a monotone predicate turns false) gives the lower bound, the upper bound, the number of equal items and the place to insert.
- The same loop can search a range of *answers* (integer square root, smallest capacity) when the check is monotone.
- It is about 25,000 times faster than a linear scan on a million numbers, but the data must be sorted first.
- In real code use `binary_search`, `binary_search_by_key` and `partition_point`, and a hash table or a tree when the data changes often.

## Sources and further reading

- [Rust standard library — slice::binary_search](https://doc.rust-lang.org/std/primitive.slice.html#method.binary_search) — the standard method with its guarantees and examples.
- [Rust standard library — slice::partition_point](https://doc.rust-lang.org/std/primitive.slice.html#method.partition_point) — the general tool, with the sorted-insert example.
- [Rust standard library — slice::binary_search_by_key](https://doc.rust-lang.org/std/primitive.slice.html#method.binary_search_by_key) — search by a key of a struct.
- [Rust standard library — u64::isqrt](https://doc.rust-lang.org/std/primitive.u64.html#method.isqrt) — the integer square root from the standard library.

---

[🏠 Start](../START-HERE.md) · [← 07.9 Merge sort and quicksort](09-merge-sort-and-quicksort.md) · [📋 Phase 07](README.md) · [07.11 Trees and binary search trees →](11-trees-and-binary-search-trees.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
