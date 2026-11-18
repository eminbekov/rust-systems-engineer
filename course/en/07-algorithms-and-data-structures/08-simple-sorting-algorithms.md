<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.7 Recursion and the call stack](07-recursion.md) · [📋 Phase 07](README.md) · [07.9 Merge sort and quicksort →](09-merge-sort-and-quicksort.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/08-simple-sorting-algorithms.md) <!-- nav -->

# Lesson 07.8 — Simple sorting algorithms

> **Phase 07 — Algorithms and data structures** · Lesson 8 of 20 · ⏱ about 3 hours (2 sittings)

## What you will learn

- What sorting is, and three properties of a sorting algorithm: *in place*, *stable* and *adaptive*.
- How bubble sort, selection sort and insertion sort work, with a picture of each on a tiny input.
- How to write them as generic Rust functions (`fn insertion_sort<T: Ord>(v: &mut [T])`).
- How to test a sorting function against the standard library on many inputs, and how to count the steps it makes.
- Why all three are slow for big inputs (O(n²)), and when to use `sort` and `sort_unstable` instead.

## Before you start

- You finished [Lesson 07.7 — Recursion and the call stack](07-recursion.md). You also know how to read Big-O (Big-O notation: a way to describe how the work grows when the input grows) from [Lesson 07.1](01-algorithms-and-big-o.md) and [Lesson 07.2](02-analyzing-time-and-space.md).
- You know generics and trait bounds ([Lesson 05.1](../05-rust-intermediate/01-generics.md)), and tests ([Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md)).
- You have the package `code/p07-algorithms` from Lessons 07.1 to 07.7, with the test helper `src/testutil.rs` (the pseudo-random generator `Rng`) from [Lesson 07.3](03-arrays-and-vectors-under-the-hood.md).

## New words

| Word | Meaning |
|---|---|
| **sorting** | Putting the items of a list in order, for example numbers from small to large. |
| **comparison sort** | A sorting algorithm that learns the order only by comparing two items at a time (`a < b`). All algorithms in this lesson are comparison sorts. |
| **in place** | An algorithm that rearranges the items inside the same slice and needs only a few extra variables, not a second copy of the data. |
| **stable** | A sort is stable if items that compare as equal keep their original order. |
| **adaptive** | A sort is adaptive if it is faster when the input is already (almost) sorted. |
| **pass** | One full walk over the slice (or over a part of it) inside a sorting algorithm. |
| **swap** | Exchanging the places of two items. The slice method is `v.swap(i, j)`. |
| **inversion** | A pair of items that is in the wrong order: the left one is bigger than the right one. A sorted slice has 0 inversions. |
| **invariant** | A statement that is true every time the loop repeats. It helps you to prove that an algorithm is right. |
| **`Ord`** | The standard trait for types that have a total order: any two values can be compared. Numbers, `String` and `char` implement it. |
| **`std`** | Short for "standard": the name of the Rust standard library, for example in `std::collections`. "Compare with `std`" means "compare with the standard library's version". |

## 1. What is sorting, and why do we care?

A **sorted** list is easier to use: you can find an item faster (Lesson 07.10 shows how), see
duplicates next to each other, find the smallest and the biggest item at the two ends, and merge two lists
quickly. Many other algorithms start with "sort the data first".

In this lesson we write three simple sorts. They are not what you use in real programs (the standard
library has better ones), but they teach the important ideas, and one of them, insertion sort, is a real
building block.

Three properties help you to compare sorting algorithms:

| Property | Question it answers |
|---|---|
| **in place** | Does it need a second copy of the data, or only a few variables? |
| **stable** | If two items are equal, does the one that came first stay first? |
| **adaptive** | Does it get faster when the data is nearly sorted? |

**Why stability matters.** Imagine a list of students already sorted by name. You now sort it by grade. If
the sort is stable, students with the same grade stay in alphabetical order. If it is not stable, they may
come out in any order.

### Where the code goes

All code in Phase 07 lives in the library package `code/p07-algorithms`, one module per topic. This
lesson adds the module `sorting`. Create the file `src/sorting.rs` (it can be empty for now) and add one
line to `src/lib.rs`:

```rust
pub mod sorting;
```

Small programs that use the library go into the folder `examples/` and run with
`cargo run --example <name>`, named `eNN_...` after the lesson, as in Lesson 07.1. (Benchmarks go into `benches/`; this lesson has none, the next one has.)

## 2. Bubble sort

**The idea.** Walk along the slice and compare each pair of neighbours. If the left one is bigger, swap
them. After one pass, the biggest item has "bubbled" to the end. Repeat for the rest of the slice.

**A picture** for `[4, 2, 5, 1]` (the item in brackets is the pair being compared; `|` marks the part that
is already final):

```text
start:   4  2  5  1
pass 1:  [4 2] 5  1   4 > 2  swap  ->  2  4  5  1
          2 [4  5] 1  4 < 5  keep
          2  4 [5  1] 5 > 1  swap  ->  2  4  1 | 5
pass 2:  [2 4] 1 | 5  keep
          2 [4  1] | 5 swap  ->  2  1 | 4  5
pass 3:  [2 1] | 4  5 swap  ->  1 | 2  4  5
result:  1  2  4  5
```

**The code.** Put this into `src/sorting.rs`:

```rust
//! Simple sorting algorithms: bubble, selection and insertion sort.

/// Bubble sort: repeatedly swaps neighbours that are in the wrong order.
pub fn bubble_sort<T: Ord>(v: &mut [T]) {
    // `end` is the index of the last item that is still not in its final place.
    for end in (1..v.len()).rev() {
        let mut swapped = false;
        for i in 0..end {
            if v[i] > v[i + 1] {
                v.swap(i, i + 1);
                swapped = true;
            }
        }
        if !swapped {
            break; // a whole pass without swaps: the slice is sorted
        }
    }
}
```

**What happens:**

1. `T: Ord` says: any type that can be compared with `<` and `>` works. `&mut [T]` is a mutable slice: the
   function changes the data where it lies, so it is *in place* and returns nothing.
2. `(1..v.len()).rev()` counts down from `len - 1` to `1`. For an empty or one-item slice the range is empty
   and the loop does nothing, so those inputs are safe.
3. The inner loop makes one pass over `v[0..=end]`, comparing `v[i]` and `v[i + 1]`. Because `i < end`,
   `i + 1` is at most `end`, which is a valid index.
4. After the pass with the largest `end`, the biggest item is at the last place, so the next pass can stop
   one item earlier. That is the **invariant**: *after a pass, the items after `end` are final*.
5. `swapped` is the speed-up. A pass without any swap proves every neighbour pair is in order, so we stop.
   This makes bubble sort *adaptive*: a sorted slice needs only one pass.

**Cost.** In the worst case (a reversed slice) the passes make `(n−1) + (n−2) + … + 1 = n(n−1)/2`
comparisons. For n = 1000 this is 499,500. We write it as **O(n²)** ("order n squared": doubling the input
makes the work about four times bigger). The best case (already sorted) is **O(n)**: one pass. It only
needs a few variables, so the extra space is **O(1)**. It is stable, because it swaps only when
`v[i] > v[i + 1]`, never for equal items.

## 3. Selection sort

**The idea.** Find the smallest item of the unsorted part and swap it to the front of that part. Then do the
same for the rest. The sorted part grows from the left.

```text
start:   4  2  5  1
i = 0:   smallest of [4 2 5 1] is 1 (index 3)  swap v[0], v[3]  ->  1 | 2  5  4
i = 1:   smallest of [2 5 4]   is 2 (index 1)  already there    ->  1  2 | 5  4
i = 2:   smallest of [5 4]     is 4 (index 3)  swap v[2], v[3]  ->  1  2  4 | 5
result:  1  2  4  5
```

```rust
/// Selection sort: finds the smallest item of the rest and puts it next in line.
pub fn selection_sort<T: Ord>(v: &mut [T]) {
    for i in 0..v.len() {
        let mut min = i;
        for j in (i + 1)..v.len() {
            if v[j] < v[min] {
                min = j;
            }
        }
        v.swap(i, min);
    }
}
```

**What happens:**

1. `min` is the index of the smallest item seen so far in the unsorted part; it starts as `i`.
2. The inner loop looks at every later item and remembers the index of a smaller one.
3. One `swap` per round puts the smallest item at position `i`. (If `min == i`, the swap changes nothing.)
4. **Invariant:** before round `i`, the items `v[0..i]` are the `i` smallest items, in order.

**Cost.** The inner loop always runs fully, whatever the input: exactly `n(n−1)/2` comparisons, so the time
is **O(n²) in the best, average and worst case**. It is not adaptive. It makes at most `n − 1` swaps, which
is its one good side: if moving an item is very expensive, selection sort moves the least. It is **not
stable**: the long-distance swap can jump an item over an equal one (the tests below prove it).

## 4. Insertion sort

**The idea.** This is how many people sort playing cards in their hand. Keep the left part sorted. Take the
next item and slide it to the left until it meets a smaller or equal item.

```text
start:   4 | 2  5  1          (the part before | is sorted)
i = 1:   take 2, slide left past 4         ->  2  4 | 5  1
i = 2:   take 5, 4 < 5, it stays           ->  2  4  5 | 1
i = 3:   take 1, slide past 5, 4 and 2     ->  1  2  4  5 |
result:  1  2  4  5
```

```rust
/// Insertion sort: grows a sorted prefix by sliding each new item into place.
pub fn insertion_sort<T: Ord>(v: &mut [T]) {
    for i in 1..v.len() {
        let mut j = i;
        while j > 0 && v[j - 1] > v[j] {
            v.swap(j - 1, j);
            j -= 1;
        }
    }
}
```

**What happens:**

1. `i` is the index of the item we insert. **Invariant:** before round `i`, `v[0..i]` is sorted.
2. `j` is where the item is now. While the left neighbour is bigger, we swap them and move `j` one step left.
3. The test `j > 0` comes **first**. With `&&` Rust stops at the first `false`, so `v[j - 1]` is never
   evaluated when `j == 0`. (Common mistake 2 shows what happens in the other order.)
4. The comparison is `>`, not `>=`, so the item stops in front of an equal one: equal items never pass each
   other, and the sort is **stable**.

**Cost.** The work depends on the input, because the `while` loop only runs as long as items are out of
order. Every swap removes exactly one *inversion* (one wrong-ordered pair), so the number of swaps equals the
number of inversions in the input:

- sorted input: 0 inversions, so about `n` comparisons: **O(n)**. Insertion sort is *adaptive*.
- reversed input: `n(n−1)/2` inversions: **O(n²)** (worst case).
- random input: about half of the pairs are inversions, so about `n²/4` swaps: **O(n²)** on average.

Space: **O(1)**, in place, stable.

| Algorithm | Best | Average | Worst | Extra space | Stable | Adaptive |
|---|---|---|---|---|---|---|
| bubble sort (with early exit) | O(n) | O(n²) | O(n²) | O(1) | yes | yes |
| selection sort | O(n²) | O(n²) | O(n²) | O(1) | no | no |
| insertion sort | O(n) | O(n²) | O(n²) | O(1) | yes | yes |

> 💡 **Tip:** Of the three, only insertion sort is used in real code. It is very fast for *short* slices
> (a dozen items) and for nearly sorted data, because it has almost no overhead. Lesson 07.9 uses it as the
> "finishing step" of faster algorithms. Bubble sort is for teaching only.

## 5. Testing a sort

A sorting function is easy to test well, because the standard library gives you the right answer to compare
with: `slice.sort()`. The plan:

1. **Edge cases:** empty slice, one item, already sorted, reversed.
2. **Many generated inputs:** compare your result with `std` on hundreds of inputs: different lengths, and
   values with many duplicates and with few duplicates.
3. **Stability:** a test that uses items that compare as equal but are different.

We need "random" numbers, but a test must give the same result on every run. The **pseudo-random number
generator** `Rng` that you wrote in `src/testutil.rs` in Lesson 07.3 does exactly that: its numbers look
random, but they are fully decided by the starting number, the *seed*. Every test of this phase uses it.

Add this to the end of `src/sorting.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;
    use std::cmp::Ordering;

    /// `len` pseudo-random numbers from 0 to `max - 1`. Same seed, same numbers.
    fn random_vec(seed: u64, len: usize, max: u64) -> Vec<i32> {
        let mut rng = Rng::new(seed);
        (0..len).map(|_| (rng.next_u64() % max) as i32).collect()
    }

    /// Runs one sorting function on many inputs and compares with the standard library.
    fn check(sort: fn(&mut [i32])) {
        for len in 0..=40 {
            for seed in 1..=5 {
                // `max` = 5 gives many duplicates, 1000 gives few.
                for max in [5, 1000] {
                    let mut actual = random_vec(seed, len, max);
                    let mut expected = actual.clone();
                    expected.sort();
                    sort(&mut actual);
                    assert_eq!(actual, expected, "len {len}, seed {seed}, max {max}");
                }
            }
        }
    }

    #[test]
    fn all_sorts_match_std() {
        check(bubble_sort::<i32>);
        check(selection_sort::<i32>);
        check(insertion_sort::<i32>);
    }

    #[test]
    fn edge_cases() {
        for case in [vec![], vec![7], vec![1, 2, 3, 4], vec![4, 3, 2, 1]] {
            let mut v = case.clone();
            insertion_sort(&mut v);
            let mut expected = case;
            expected.sort();
            assert_eq!(v, expected);
        }
    }

    /// A card that is compared by `rank` only. The `tag` shows the original order.
    #[derive(Debug, Clone, Copy, PartialEq, Eq)]
    struct Card {
        rank: u8,
        tag: char,
    }

    impl PartialOrd for Card {
        fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
            Some(self.cmp(other))
        }
    }

    impl Ord for Card {
        fn cmp(&self, other: &Self) -> Ordering {
            self.rank.cmp(&other.rank)
        }
    }

    /// Sorts the cards 2a, 2b, 1c and returns the tags in the sorted order.
    fn sorted_tags(sort: fn(&mut [Card])) -> String {
        let mut v = vec![
            Card { rank: 2, tag: 'a' },
            Card { rank: 2, tag: 'b' },
            Card { rank: 1, tag: 'c' },
        ];
        sort(&mut v);
        v.iter().map(|c| c.tag).collect()
    }

    #[test]
    fn bubble_and_insertion_are_stable() {
        assert_eq!(sorted_tags(bubble_sort), "cab");
        assert_eq!(sorted_tags(insertion_sort), "cab");
    }

    #[test]
    fn selection_sort_is_not_stable() {
        // The two cards of rank 2 swapped places: "b" now comes before "a".
        assert_eq!(sorted_tags(selection_sort), "cba");
    }
}
```

**What happens:**

1. `random_vec` builds a list of `len` numbers below `max` from a seed, with the `Rng` of Lesson 07.3. The same
   seed always gives the same list, so a failing test can be repeated.
2. `check` takes any function of the type `fn(&mut [i32])`. The call `check(bubble_sort::<i32>)` passes the
   generic function with `T` fixed to `i32` (the *turbofish* `::<i32>` from Lesson 05.1). The loops test
   lengths 0 to 40 with 5 seeds and 2 value ranges: 410 inputs per algorithm. With `max = 5` there are
   many equal items, which is where bugs hide.
3. `edge_cases` loops over four small inputs: empty, one item, sorted and reversed.
4. `Card` implements `Ord` by hand and compares only `rank`. Two cards with the same rank are "equal" for
   sorting, but their `tag` letters differ, so we can *see* whether their order changed. (`PartialOrd` and
   `Ord` come as a pair; Lesson 05.2 explained traits.)
5. The input `[2a, 2b, 1c]` sorted stably must give `c a b`. Bubble and insertion sort do. Selection sort
   swaps `v[0]` (the `2a`) with the `1c` at the end, so `2a` jumps behind `2b`: the result is `c b a`. We
   *assert the instability* to document it.

Run the tests from the package folder:

```bash
cd ~/projects/rust/code/p07-algorithms
cargo test sorting
```

```text
running 4 tests
test sorting::tests::selection_sort_is_not_stable ... ok
test sorting::tests::bubble_and_insertion_are_stable ... ok
test sorting::tests::edge_cases ... ok
test sorting::tests::all_sorts_match_std ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 26 filtered out; finished in 0.01s
```

(The name `sorting` selects only the tests of this module. The 26 filtered-out tests are the ones from the main text of
Lessons 07.1 to 07.7; if you also wrote the tests of the exercises, your number is bigger.)

> 🧪 **Testing:** "Compare with a trusted slow or standard version on many generated inputs" is a very strong
> technique called **differential testing**. It finds the bugs that hand-written examples miss, like an
> off-by-one at the end of a slice. In Phase 09 you will meet its big brother, property-based testing.

## 6. Counting the steps

"O(n²)" is a claim. Let us check it. We want to count comparisons without changing our sorting functions.
The trick: sort a type whose comparison counts itself. Create `examples/e08_count_steps.rs`:

```rust
use p07_algorithms::sorting::{bubble_sort, insertion_sort, selection_sort};
use std::cell::Cell;
use std::cmp::Ordering;

/// A number that counts how many times it is compared.
struct Counted<'a> {
    value: i32,
    counter: &'a Cell<u64>,
}

impl PartialEq for Counted<'_> {
    fn eq(&self, other: &Self) -> bool {
        self.value == other.value
    }
}

impl Eq for Counted<'_> {}

impl PartialOrd for Counted<'_> {
    fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
        Some(self.cmp(other))
    }
}

impl Ord for Counted<'_> {
    fn cmp(&self, other: &Self) -> Ordering {
        self.counter.set(self.counter.get() + 1);
        self.value.cmp(&other.value)
    }
}

/// Sorts a copy of `data` with `sort` and returns the number of comparisons.
fn count<'a>(sort: fn(&mut [Counted<'a>]), data: &[i32], counter: &'a Cell<u64>) -> u64 {
    counter.set(0);
    let mut items: Vec<Counted> = data
        .iter()
        .map(|&value| Counted { value, counter })
        .collect();
    sort(&mut items);
    counter.get()
}

fn pseudo_random(len: usize) -> Vec<i32> {
    let mut x: u64 = 12345;
    (0..len)
        .map(|_| {
            x ^= x << 13;
            x ^= x >> 7;
            x ^= x << 17;
            (x % 1_000_000) as i32
        })
        .collect()
}

fn main() {
    let counter = Cell::new(0);
    println!(
        "{:>6} {:>12} {:>12} {:>12} {:>14}",
        "n", "bubble", "selection", "insertion", "insertion/sorted"
    );
    for n in [100, 200, 400, 800] {
        let data = pseudo_random(n);
        let sorted: Vec<i32> = (0..n as i32).collect();
        println!(
            "{:>6} {:>12} {:>12} {:>12} {:>14}",
            n,
            count(bubble_sort::<Counted>, &data, &counter),
            count(selection_sort::<Counted>, &data, &counter),
            count(insertion_sort::<Counted>, &data, &counter),
            count(insertion_sort::<Counted>, &sorted, &counter),
        );
    }
}
```

**What happens:**

1. `Counted` holds a number and a shared reference to a `Cell<u64>` (interior mutability, Lesson 06.4: a
   `Cell` can be changed through a shared reference). The lifetime `'a` says the counter must live at least as
   long as the numbers.
2. Every comparison goes through `Ord::cmp`, and our `cmp` adds one to the counter before it compares the
   values. The operators `<` and `>` call `partial_cmp`, which calls `cmp`, so every comparison inside
   `bubble_sort`, `selection_sort` and `insertion_sort` is counted without any change to them.
3. `count` wraps every number in `Counted`, runs the sort and returns the counter. The generic functions
   are used with `T = Counted`: this is what generics are good for.
4. `main` doubles `n` each time and prints the counts. The last column sorts an already sorted list.

```bash
cargo run --example e08_count_steps
```

```text
     n       bubble    selection    insertion insertion/sorted
   100         4845         4950         2376             99
   200        19600        19900         9981            199
   400        77784        79800        38651            399
   800       318654       319600       157250            799
```

**How to read it:**

1. When `n` doubles, the counts of all three sorts grow about **four times** (4950, 19900, 79800, 319600 for
   selection sort): this is the signature of O(n²).
2. Selection sort needs exactly `n(n−1)/2` comparisons (for 100: 100·99/2 = 4950), whatever the data.
3. Insertion sort needs about half as many on random data (2376 for n = 100), because on average it slides
   an item only halfway.
4. On sorted input insertion sort needs only `n − 1` comparisons (99, 199, …): this is the **O(n)** best case.

> ⚡ **Performance:** For n = 1,000,000 insertion sort on random data makes about n²/4 = 2.5·10¹¹ comparisons,
> minutes of work. An O(n log n) sort needs about 2·10⁷ comparisons, a small fraction of a second. This is the
> difference an algorithm makes, and no compiler setting can close it. Lesson 07.9 measures the real times.

## 7. What to use in real code

The standard library has two sorts for slices and `Vec`:

| Method | Stable | Extra memory | Use it when |
|---|---|---|---|
| `v.sort()` | yes | may allocate a buffer | the default; you need equal items to keep their order |
| `v.sort_unstable()` | no | none (in place) | the default for plain numbers and other types where equal items are indistinguishable; usually a bit faster |

Variants: `sort_by(|a, b| …)` takes a comparison closure that returns an `Ordering`; `sort_by_key(|x| …)` sorts
by a key computed from each item; `sort_unstable_by` and `sort_unstable_by_key` are the unstable twins.
`is_sorted()` checks in O(n) whether a slice is sorted. For floating-point numbers use
`sort_by(|a, b| a.total_cmp(b))`, because `f32` and `f64` do not implement `Ord` (the value NaN, "not a number",
has no place in an order).

Both are O(n log n) in the worst case. Write your own simple sort only to learn, or in the very rare case
that you must sort tiny slices in a hot loop and have measured that it helps.

## Common mistakes

**1. Swapping with tuples of indexed items.** In a generic function the items may not be `Copy`, so you
cannot take them out of the slice:

```rust
pub fn swap_pair<T: Ord>(v: &mut [T], i: usize, j: usize) {
    (v[i], v[j]) = (v[j], v[i]);
}
```

```text
error[E0508]: cannot move out of type `[T]`, a non-copy slice
 --> src/lib.rs:2:21
  |
2 |     (v[i], v[j]) = (v[j], v[i]);
  |                     ^^^^
  |                     |
  |                     cannot move out of here
  |                     move occurs because `v[_]` has type `T`, which does not implement the `Copy` trait
```

The compiler says: you cannot move a `T` out of a slice and leave a hole. The fix is the slice method
`v.swap(i, j)`, which exchanges the two items safely and without any `Copy`.

**2. Checking `j > 0` too late.** Writing the two conditions of the insertion sort loop in the wrong order:

```rust
while v[j - 1] > v[j] && j > 0 {
```

With `j` of type `usize` (an unsigned number), `j - 1` when `j == 0` is an overflow. A test with the reversed
input `[3, 2, 1]` fails in a debug build:

```text
thread 'reversed' panicked at src/lib.rs:4:17:
attempt to subtract with overflow
```

Fix: test `j > 0` first. `&&` does not evaluate the right side when the left side is `false`.

**3. Using `>=` in insertion sort.** `v[j - 1] >= v[j]` also swaps equal items. The result is still sorted,
but the sort is no longer stable, and it does extra work. Use the strict `>`.

## Try it yourself

1. Add the module, the three functions and the tests in the order of this lesson. Run `cargo test`.
2. In `insertion_sort` change `>` to `>=` and run the tests. Which test fails, and why? Change it back.
3. Run `cargo run --example e08_count_steps` with `[1000, 2000]` instead of the four numbers. Does the count still
   grow four times when `n` doubles?
4. Run `cargo fmt` and `cargo clippy --all-targets`, then commit:
   `git add code/p07-algorithms && git commit -m "feat(p07): bubble, selection and insertion sort"`.

## Exercises

### Exercise 1 — Trace by hand (easy)

Write down the list after each round of insertion sort for `[3, 1, 4, 1, 5]`. How many swaps are made in
total, and how many inversions does the input have?

<details>
<summary>Hint</summary>

An inversion is a pair `(left, right)` with `left > right`, where `left` stands earlier in the list. Count the
pairs. Each swap of insertion sort fixes exactly one such pair.

</details>

<details>
<summary>Answer</summary>

```text
start:   3  1  4  1  5
i = 1:   1  3  4  1  5     (1 slides past 3: 1 swap)
i = 2:   1  3  4  1  5     (4 stays)
i = 3:   1  1  3  4  5     (the second 1 slides past 4 and 3: 2 swaps, stops in front of the first 1)
i = 4:   1  1  3  4  5     (5 stays)
```

Total: 3 swaps. Inversions: (3, 1), (3, 1) and (4, 1) are the three wrong pairs, so 3. The two numbers are
equal, as the lesson says. The second `1` stops in front of the first `1` because the comparison is strict, so
the equal ones keep their order.

</details>

### Exercise 2 — Test with strings (easy)

Write a test that sorts a `Vec<String>` with `insertion_sort` (it is generic, so it must work), compares the
result with the standard library's `sort`, and also checks `is_sorted()`.

<details>
<summary>Hint</summary>

Clone the vector before sorting it, sort the clone with `.sort()`, and compare the two with `assert_eq!`.
Put the test in the `tests` module.

</details>

<details>
<summary>Answer</summary>

```rust
#[test]
fn sorts_words() {
    let mut words = vec![
        "pear".to_string(),
        "apple".to_string(),
        "fig".to_string(),
        "apple".to_string(),
    ];
    let mut expected = words.clone();
    expected.sort();

    insertion_sort(&mut words);

    assert_eq!(words, expected);
    assert!(words.is_sorted());
}
```

Why it works: `String` implements `Ord`, so `T = String` satisfies the bound. Strings are compared letter by
letter, so `"apple" < "fig" < "pear"`. The duplicate `"apple"` checks that equal items survive.

</details>

### Exercise 3 — State the complexity (medium)

Give the time complexity (as Big-O, in terms of `n`) of each case, and say roughly how many comparisons it
makes:

1. insertion sort on an already sorted list of 1,000,000 numbers;
2. selection sort on an already sorted list of 1,000,000 numbers;
3. bubble sort (with early exit) on a reversed list of 1,000 numbers: how many comparisons and how many swaps?

<details>
<summary>Hint</summary>

For (1) and (2) think about what each inner loop does when nothing needs moving. For (3) add up the lengths
of the passes: `999 + 998 + … + 1`.

</details>

<details>
<summary>Answer</summary>

1. **O(n)**: the `while` loop checks one comparison per item and stops at once. About 999,999 comparisons.
2. **O(n²)**: selection sort does not notice that the data is sorted; it always makes `n(n−1)/2`, about
   5·10¹¹ (500 billion) comparisons.
3. **O(n²)**: 999 + 998 + … + 1 = 499,500 comparisons. In a reversed list every compared pair is out of order,
   so there are also 499,500 swaps.

Why it works: the best case of an adaptive algorithm depends on the data; for a non-adaptive one it does not.

</details>

### Exercise 4 — Swaps equal inversions (hard)

Write `inversions<T: Ord>(v: &[T]) -> usize` that counts the wrong-ordered pairs with two nested loops, and
`insertion_sort_counting_swaps<T: Ord>(v: &mut [T]) -> usize` that sorts and returns the number of swaps it
made. Write a test that checks they are equal for many generated inputs, and a test that a reversed list of
1000 numbers has 499,500 inversions. What is the time complexity of `inversions`?

<details>
<summary>Hint</summary>

For each `i`, count the `j > i` with `v[i] > v[j]`. For the swap counter, add one in the `while` loop after
each `swap`. Reuse the `random_vec` helper from the `tests` module (use a small `max` such as 20 to get duplicates).

</details>

<details>
<summary>Answer</summary>

```rust
pub fn inversions<T: Ord>(v: &[T]) -> usize {
    let mut count = 0;
    for i in 0..v.len() {
        for j in (i + 1)..v.len() {
            if v[i] > v[j] {
                count += 1;
            }
        }
    }
    count
}

pub fn insertion_sort_counting_swaps<T: Ord>(v: &mut [T]) -> usize {
    let mut swaps = 0;
    for i in 1..v.len() {
        let mut j = i;
        while j > 0 && v[j - 1] > v[j] {
            v.swap(j - 1, j);
            swaps += 1;
            j -= 1;
        }
    }
    swaps
}

#[test]
fn swaps_equal_inversions() {
    assert_eq!(inversions(&[3, 1, 4, 1, 5]), 3);
    for len in 0..30 {
        for seed in 1..=5 {
            let v = random_vec(seed, len, 20);
            let expected = inversions(&v);
            let mut w = v.clone();
            assert_eq!(insertion_sort_counting_swaps(&mut w), expected);
        }
    }
    let reversed: Vec<i32> = (0..1000).rev().collect();
    assert_eq!(inversions(&reversed), 499_500);
}
```

Why it works: each swap of two neighbours fixes exactly one wrong pair and creates no new one, so the number
of swaps cannot be anything but the number of inversions. `inversions` is **O(n²)** because of the two nested
loops over about `n²/2` pairs. (A faster way, O(n log n), appears after the merge sort of Lesson 07.9.)

</details>

## Quiz

1. What does "stable" mean for a sorting algorithm?

<details>
<summary>Answer</summary>

Items that compare as equal keep the same relative order as in the input. Bubble sort and insertion sort are
stable. Selection sort is not.

</details>

2. Why does insertion sort take O(n) time on a sorted list but selection sort O(n²)?

<details>
<summary>Answer</summary>

Insertion sort stops sliding an item as soon as its left neighbour is not bigger, so on sorted data each item
costs one comparison. Selection sort always scans the whole unsorted part to find the smallest item, whatever
the data looks like.

</details>

3. When `n` becomes 10 times bigger, how much more work does an O(n²) algorithm do?

<details>
<summary>Answer</summary>

About 100 times more (10² = 100).

</details>

4. Which method should you use to sort a `Vec<u32>` in a real program?

<details>
<summary>Answer</summary>

`sort_unstable()` (equal numbers are indistinguishable, so stability does not matter), or `sort()` if you
prefer the stable one. Never a hand-written O(n²) sort.

</details>

## Summary

- Sorting arranges items in order; a comparison sort learns the order only by comparing pairs.
- Three properties compare algorithms: in place (little extra memory), stable (equal items keep their order)
  and adaptive (faster on nearly sorted data).
- Bubble, selection and insertion sort are all O(n²) in the worst case and O(1) in extra space.
- Insertion sort is the best of them: stable, adaptive (O(n) on sorted data), and fast for short slices.
  Its swap count equals the number of inversions.
- Test a sort against `std` on many generated inputs, with many duplicates, and add edge cases and a
  stability test.
- Counting comparisons shows that doubling `n` makes an O(n²) sort do about four times more work.
- In real code, use `sort`, `sort_unstable`, `sort_by_key` and friends.

## Sources and further reading

- [Rust standard library — slice::sort](https://doc.rust-lang.org/std/primitive.slice.html#method.sort) — the stable sort: guarantees and cost.
- [Rust standard library — slice::sort_unstable](https://doc.rust-lang.org/std/primitive.slice.html#method.sort_unstable) — the unstable sort, and when it is the better choice.
- [Rust standard library — slice::sort_by_key](https://doc.rust-lang.org/std/primitive.slice.html#method.sort_by_key) — sorting by a computed key.
- [Rust standard library — std::cmp::Ord](https://doc.rust-lang.org/std/cmp/trait.Ord.html) — the trait behind `T: Ord`, and the rules a correct order must follow.
- [The Rust Programming Language — Writing automated tests](https://doc.rust-lang.org/book/ch11-00-testing.html) — a reminder of the testing tools used here.

---

[🏠 Start](../START-HERE.md) · [← 07.7 Recursion and the call stack](07-recursion.md) · [📋 Phase 07](README.md) · [07.9 Merge sort and quicksort →](09-merge-sort-and-quicksort.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
