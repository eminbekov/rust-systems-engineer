<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.8 Simple sorting algorithms](08-simple-sorting-algorithms.md) · [📋 Phase 07](README.md) · [07.10 Binary search →](10-binary-search.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/09-merge-sort-and-quicksort.md) <!-- nav -->

# Lesson 07.9 — Merge sort and quicksort

> **Phase 07 — Algorithms and data structures** · Lesson 9 of 20 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- The idea of *divide and conquer*, and why splitting the work in halves gives O(n log n).
- How merge sort works: split, sort the halves, merge two sorted lists.
- How quicksort works: choose a pivot, partition around it, repeat. Why a bad pivot makes it slow, and how to avoid that.
- How to test both against the standard library, measure them with `criterion`, and see why `sort` and `sort_unstable` are still faster.

## Before you start

- You finished [Lesson 07.8 — Simple sorting algorithms](08-simple-sorting-algorithms.md). This lesson continues in the same file, `src/sorting.rs` of `code/p07-algorithms`, and reuses `insertion_sort` and the test helpers `random_vec`, `check` and `sorted_tags` from that lesson.
- You know recursion and the call stack ([Lesson 07.7](07-recursion.md)) and slices ([Lesson 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md)).
- You know `criterion` from [Lesson 05.12](../05-rust-intermediate/12-benchmarking-basics.md). Make sure `Cargo.toml` has `criterion = "0.8"` in `[dev-dependencies]` (`cargo add --dev criterion@0.8`).

## New words

| Word | Meaning |
|---|---|
| **divide and conquer** | A way to solve a problem: split it into smaller parts of the same kind, solve each part, and combine the answers. |
| **logarithm** (log₂ n) | How many times you can halve `n` until you reach 1. log₂ 8 = 3, log₂ 1,000,000 ≈ 20. It grows very slowly. |
| **O(n log n)** | "n log n" time: about `n` steps for each of about log₂ n levels. Much faster than O(n²) for big `n`. |
| **merge** | Combining two sorted lists into one sorted list by repeatedly taking the smaller front item. |
| **scratch buffer** | A temporary block of memory an algorithm uses for its work and then forgets. |
| **pivot** | The item that quicksort uses as the dividing line: smaller items go to its left, bigger ones to its right. |
| **partition** | To rearrange a slice around a pivot, so that it falls into groups (smaller, equal, bigger). |
| **median** | The middle value of a group after sorting it. The median of 5, 9 and 4 is 5. |
| **median-of-three** | A way to choose the pivot: take the median of the first, middle and last item. |
| **recursion depth** | How many calls are waiting on the call stack at the same moment (Lesson 07.7). |
| **quickselect** | An algorithm that finds the k-th smallest item with one partition step at a time, without sorting everything. |

## 1. Divide and conquer, and why n log n

The simple sorts of Lesson 07.8 are slow because they compare almost every pair of items: about `n²/2`
comparisons. The better idea is **divide and conquer**: split the slice into two halves, sort each half
(with the same algorithm: this is recursion), and combine the two sorted halves cheaply.

Look at how the slice is cut in halves. Each line is a *level*:

```text
level 0:            [5 2 8 1 9 3 7 4]           1 slice  of 8
level 1:        [5 2 8 1]     [9 3 7 4]         2 slices of 4
level 2:      [5 2] [8 1]   [9 3] [7 4]         4 slices of 2
level 3:     [5][2] [8][1] [9][3] [7][4]        8 slices of 1   (nothing left to sort)
```

Halving 8 items takes 3 steps, because 2³ = 8. We say log₂ 8 = 3. For a million items, 20 halvings are
enough (2²⁰ ≈ one million). If the work to combine the parts on one level is about `n` steps in total, then the
whole algorithm needs about `n × log₂ n` steps: **O(n log n)**. For n = 1,000,000 that is about 20
million steps, compared with 500 billion for an O(n²) sort.

## 2. Merge sort

Merge sort does the real work while coming *back up* the levels. The key operation is **merging** two
sorted lists: look at the front item of each list, take the smaller one, and repeat. Each item is looked at
once, so merging `n` items costs O(n).

Merging `[1, 4, 6]` and `[2, 3, 9]`:

```text
left = [1 4 6]   right = [2 3 9]    out = []
1 vs 2  take 1                      out = [1]
4 vs 2  take 2                      out = [1 2]
4 vs 3  take 3                      out = [1 2 3]
4 vs 9  take 4                      out = [1 2 3 4]
6 vs 9  take 6                      out = [1 2 3 4 6]
left is empty: copy the rest        out = [1 2 3 4 6 9]
```

Going back up for our 8 items:

```text
level 3:     [5][2] [8][1] [9][3] [7][4]
merge:       [2 5]  [1 8]  [3 9]  [4 7]
merge:         [1 2 5 8]     [3 4 7 9]
merge:             [1 2 3 4 5 7 8 9]
```

Add this line near the top of `src/sorting.rs`, below the `//!` comment (we need it for quicksort too):

```rust
use std::cmp::Ordering;
```

and this code below `insertion_sort` (above the `tests` module):

```rust
/// Merge sort: split in two halves, sort each half, merge the sorted halves.
pub fn merge_sort<T: Ord + Clone>(v: &mut [T]) {
    let mut buf = v.to_vec(); // one scratch buffer, allocated once
    merge_sort_rec(v, &mut buf);
}

fn merge_sort_rec<T: Ord + Clone>(v: &mut [T], buf: &mut [T]) {
    let n = v.len();
    if n <= 1 {
        return;
    }
    let mid = n / 2;
    merge_sort_rec(&mut v[..mid], &mut buf[..mid]);
    merge_sort_rec(&mut v[mid..], &mut buf[mid..]);

    // Merge the sorted halves v[..mid] and v[mid..] into buf, then copy back.
    let (mut i, mut j, mut k) = (0, mid, 0);
    while i < mid && j < n {
        if v[j] < v[i] {
            buf[k] = v[j].clone();
            j += 1;
        } else {
            buf[k] = v[i].clone(); // on a tie take the left one: this keeps the sort stable
            i += 1;
        }
        k += 1;
    }
    while i < mid {
        buf[k] = v[i].clone();
        i += 1;
        k += 1;
    }
    while j < n {
        buf[k] = v[j].clone();
        j += 1;
        k += 1;
    }
    v.clone_from_slice(&buf[..n]);
}
```

**What happens:**

1. `merge_sort` is the public function. It makes **one** scratch buffer with the same length (`to_vec`
   copies the slice) and passes both to the recursive helper. Allocating once is much cheaper than allocating at every call.
2. The *base case* (Lesson 07.7) is `n <= 1`: an empty or one-item slice is already sorted.
3. Otherwise the slice is cut at `mid`. The two recursive calls get `&mut v[..mid]` and `&mut v[mid..]`
   (and the same parts of `buf`). They run one after the other, so the two mutable borrows never overlap.
4. The merge: `i` walks the left half `v[..mid]`, `j` walks the right half `v[mid..]`, `k` is the next free place in `buf`. The loop
   takes the smaller front item. Only if the right item is **strictly** smaller (`v[j] < v[i]`) is it taken first;
   on a tie the left one wins. That is what makes merge sort **stable**.
5. When one half is used up, the two small `while` loops copy the rest of the other half.
6. `clone_from_slice` copies the merged result from `buf` back into `v`. The `Clone` bound is needed
   because we copy items instead of moving them out of a slice (a slice cannot have holes).

**Cost.** There are about log₂ n levels, and on every level the merges together touch `n` items:
**O(n log n) in the best, average and worst case**. Extra space: the buffer, **O(n)**. It is **stable**.
It is not in place. The recursion depth is only about log₂ n, so the call stack stays small.

## 3. Quicksort

Quicksort does its real work while going *down* the levels. The idea:

1. Choose one item, the **pivot**.
2. **Partition** the slice: move all smaller items to the left of the pivot and all bigger items to its right.
   Now the pivot is in its final place.
3. Sort the left part and the right part with the same method.

There is nothing to merge afterwards: when the parts are sorted, the whole slice is sorted.

There are two traps, and our code avoids both:

- **Equal items.** If many items equal the pivot, the common two-way partition puts them all on one side
  and the parts become very uneven. We use a **three-way partition** with three groups: smaller, equal, bigger. The equal group is
  finished at once.
- **A bad pivot.** If the pivot is always the smallest or biggest item (for example the first item of an already sorted slice),
  the parts have sizes 0 and `n − 1`, and quicksort needs `n` levels: **O(n²)**. We choose the pivot with
  **median-of-three**: the median of the first, middle and last item. Sorted, reversed and equal inputs are then no problem.

Partitioning `[4, 7, 2, 4, 9, 1]` with the pivot `4`. Three markers split the slice: `[..lt]` is smaller than
the pivot, `[lt..i]` equals it, `[gt..]` is bigger, and `[i..gt]` is not yet looked at:

```text
start (pivot 4 at the front):  4 7 2 4 9 1      lt=0 i=1 gt=6
7 > 4  send it right           4 1 2 4 9 | 7    gt=5
1 < 4  send it left            1 4 2 4 9 | 7    lt=1 i=2
2 < 4  send it left            1 2 4 4 9 | 7    lt=2 i=3
4 = 4  leave it                1 2 4 4 9 | 7    i=4
9 > 4  send it right           1 2 4 4 | 9 7    gt=4, i == gt: stop
result:                        [1 2] [4 4] [9 7]   sort the two outer parts
```

Add this code below `merge_sort_rec`:

```rust
/// Slices shorter than this are finished with insertion sort.
const SMALL: usize = 16;

/// Quicksort: pick a pivot, move smaller items left and bigger items right, repeat on both sides.
pub fn quicksort<T: Ord>(mut v: &mut [T]) {
    while v.len() > SMALL {
        let pivot = median_of_three(v);
        let (lt, gt) = partition3(v, pivot);
        let (left, rest) = v.split_at_mut(lt);
        let right = &mut rest[gt - lt..];
        // Recurse into the smaller part, loop on the bigger one: the depth stays O(log n).
        if left.len() < right.len() {
            quicksort(left);
            v = right;
        } else {
            quicksort(right);
            v = left;
        }
    }
    insertion_sort(v);
}

/// Returns the index of the median of the first, middle and last item.
fn median_of_three<T: Ord>(v: &[T]) -> usize {
    let (a, b, c) = (0, v.len() / 2, v.len() - 1);
    if v[a] < v[b] {
        if v[b] < v[c] {
            b
        } else if v[a] < v[c] {
            c
        } else {
            a
        }
    } else if v[a] < v[c] {
        a
    } else if v[b] < v[c] {
        c
    } else {
        b
    }
}

/// Rearranges `v` around the pivot `v[pivot]`. Returns `(lt, gt)` such that
/// `v[..lt]` is smaller than the pivot, `v[lt..gt]` equals it, `v[gt..]` is bigger.
fn partition3<T: Ord>(v: &mut [T], pivot: usize) -> (usize, usize) {
    v.swap(0, pivot); // keep the pivot at the front while we work
    let (mut lt, mut i, mut gt) = (0, 1, v.len());
    while i < gt {
        match v[i].cmp(&v[lt]) {
            Ordering::Less => {
                v.swap(lt, i);
                lt += 1;
                i += 1;
            }
            Ordering::Greater => {
                gt -= 1;
                v.swap(i, gt);
            }
            Ordering::Equal => i += 1,
        }
    }
    (lt, gt)
}
```

**What happens:**

1. `partition3` first swaps the pivot to the front. Then it keeps this **invariant**: `v[..lt]` is smaller than the
   pivot, `v[lt..i]` equals it (so `v[lt]` is always a copy of the pivot value), `v[i..gt]` is unknown, `v[gt..]` is bigger.
2. For the next unknown item `v[i]`, a `match` on `cmp` (it returns an `Ordering`: `Less`, `Equal` or `Greater`) decides:
   *smaller*: swap it to `lt` and move both markers right; *bigger*: swap it to the end of the unknown part (`gt` moves
   left, `i` stays, because the item that arrived at `i` has not been looked at); *equal*: just move `i`.
3. When `i` meets `gt` nothing is unknown, and the function returns the two borders. Every item is looked at once: **O(n)**.
4. `median_of_three` compares three items and returns the *index* of the middle one. It never panics for
   `len >= 1`, because for a one-item slice all three indices are 0.
5. In `quicksort`, `split_at_mut(lt)` cuts the slice into two **non-overlapping** mutable parts. `rest[gt - lt..]` skips the
   equal group. Now `left` and `right` are the two parts to sort.
6. The `if` sorts the **smaller** part by recursion and then loops again on the **bigger** part. Each recursive call works on at most half of the items, so the recursion depth is at most
   about log₂ n, even for bad inputs. (If both parts were handled by recursion, a bad pivot could make the call stack
   `n` calls deep and the program would crash with a stack overflow, Lesson 07.7.)
7. A slice of at most 16 items is finished by the `insertion_sort` from Lesson 07.8. For short slices it is faster
   than more partitioning. The number 16 is a typical choice; real libraries tune it by measuring.

**Cost.** On average **O(n log n)**: each level partitions `n` items and good pivots make about log₂ n levels. With
median-of-three the worst case is rare on natural data, but it still exists (some special input can always defeat any simple pivot rule): **O(n²)**
in the worst case. Extra space: only the call stack, **O(log n)**. **Not stable**: partitioning swaps items over long distances.

| Algorithm | Best | Average | Worst | Extra space | Stable |
|---|---|---|---|---|---|
| merge sort | O(n log n) | O(n log n) | O(n log n) | O(n) | yes |
| quicksort (our version) | O(n) (all items equal) | O(n log n) | O(n²) | O(log n) | no |

## 4. Testing both

Add these tests **inside** the `tests` module from Lesson 07.8 (before its last closing brace). They use `random_vec`,
`check` and `sorted_tags` from that lesson:

```rust
    /// Like `check`, but with longer inputs, so that the pivot logic runs many times.
    fn check_long(sort: fn(&mut [i32])) {
        // The lengths around 16 test the switch to insertion sort.
        for len in [0, 1, 2, 15, 16, 17, 18, 31, 32, 33, 100, 1000, 5000] {
            for seed in 1..=3 {
                for max in [3, 50, 1_000_000] {
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
    fn merge_sort_and_quicksort_match_std() {
        check(merge_sort::<i32>);
        check(quicksort::<i32>);
        check_long(merge_sort::<i32>);
        check_long(quicksort::<i32>);
    }

    #[test]
    fn quicksort_handles_sorted_reversed_and_equal_input() {
        let n = 100_000;
        let sorted: Vec<i32> = (0..n).collect();
        let reversed: Vec<i32> = (0..n).rev().collect();
        let equal = vec![7; n as usize];
        for input in [sorted, reversed, equal] {
            let mut actual = input.clone();
            quicksort(&mut actual);
            let mut expected = input;
            expected.sort();
            assert_eq!(actual, expected);
        }
    }

    #[test]
    fn merge_sort_is_stable() {
        assert_eq!(sorted_tags(merge_sort), "cab");
    }
```

**What happens:**

1. `check_long` repeats the idea of `check` with longer inputs. The values `max = 3` (almost all equal) and
   `1_000_000` (almost all different) test the three-way partition from both sides. The lengths 15, 16, 17 and 18 are at the border where
   `quicksort` switches to `insertion_sort`. **Borders are where off-by-one bugs live.**
2. The test with 100,000 sorted, reversed and equal items is the "bad pivot" test. A quicksort with a poor pivot rule would be extremely
   slow here, or crash with a stack overflow.
3. `sorted_tags(merge_sort)` reuses the `Card` test of Lesson 07.8: with equal ranks, the order `c a b` proves stability.

```bash
cd ~/projects/rust/code/p07-algorithms
cargo test sorting
```

```text
running 7 tests
test sorting::tests::bubble_and_insertion_are_stable ... ok
test sorting::tests::edge_cases ... ok
test sorting::tests::merge_sort_is_stable ... ok
test sorting::tests::selection_sort_is_not_stable ... ok
test sorting::tests::all_sorts_match_std ... ok
test sorting::tests::merge_sort_and_quicksort_match_std ... ok
test sorting::tests::quicksort_handles_sorted_reversed_and_equal_input ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 26 filtered out; finished in 1.12s
```

(The exact order of the lines can differ from run to run, because the tests run in parallel. The filtered-out tests are
those of Lessons 07.1 to 07.7; with the tests of the exercises your number is bigger.)

> 🧪 **Testing:** A test that passes does not prove the algorithm is right; it proves the algorithm is right *for the
> inputs you tried*. That is why the tests use many lengths, many value ranges and the border lengths. Phase 09 shows
> tools that generate inputs for you.

## 5. How fast are they really?

Create `benches/fast_sorts.rs` and register it in `Cargo.toml` like in Lesson 05.12:

```toml
[[bench]]
name = "fast_sorts"
harness = false
```

```rust
use criterion::{BatchSize, BenchmarkId, Criterion, criterion_group, criterion_main};
use p07_algorithms::sorting::{insertion_sort, merge_sort, quicksort};

type SortFn = fn(&mut [i32]);

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

fn bench_fast_sorts(c: &mut Criterion) {
    let n = 10_000;
    let random = pseudo_random(n);
    let sorted: Vec<i32> = (0..n as i32).collect();
    let inputs = [("random", random), ("sorted", sorted)];
    let sorts: [(&str, SortFn); 5] = [
        ("insertion", insertion_sort::<i32>),
        ("merge", merge_sort::<i32>),
        ("quick", quicksort::<i32>),
        ("std sort", <[i32]>::sort),
        ("std sort_unstable", <[i32]>::sort_unstable),
    ];

    let mut group = c.benchmark_group("sort 10_000");
    for (input_name, data) in &inputs {
        for (name, sort) in sorts {
            group.bench_with_input(BenchmarkId::new(name, input_name), data, |b, data| {
                b.iter_batched(
                    || data.clone(),
                    |mut v| {
                        sort(&mut v);
                        v
                    },
                    BatchSize::SmallInput,
                )
            });
        }
    }
    group.finish();
}

criterion_group!(benches, bench_fast_sorts);
criterion_main!(benches);
```

**What happens:** the same pattern as the benchmark of Lesson 05.12. `iter_batched` clones the input in an untimed
setup step, because a sort destroys its input. Every algorithm gets the same 10,000 numbers twice: once random and once
already sorted. `pseudo_random` uses a small xorshift formula (shifts and exclusive-or, like `Rng`): a benchmark
cannot use `Rng` from `src/testutil.rs`, because that module is compiled only for `cargo test` (`#[cfg(test)]`). Run it
(about two minutes):

```bash
cargo bench --bench fast_sorts -- --warm-up-time 1 --measurement-time 3
```

Best estimates from one run on an Apple M4 Pro Mac, release build, 2026-11-18 (your numbers will be different):

| Algorithm | random input | sorted input |
|---|---|---|
| insertion sort | 7.41 ms | 2.64 µs |
| merge sort (ours) | 354 µs | 96.3 µs |
| quicksort (ours) | 274 µs | 511 µs |
| `sort` (std) | 54.9 µs | 2.66 µs |
| `sort_unstable` (std) | 56.8 µs | 2.55 µs |

**How to read it:**

1. On random data, merge sort and quicksort are about **20 to 27 times faster** than insertion sort for only 10,000 items (7.41 ms against 0.35 and 0.27 ms).
   The gap grows with `n`, because O(n log n) beats O(n²).
2. Quicksort is a little faster than merge sort on random data: it does not copy items into a buffer.
3. On **sorted** input, insertion sort is the champion of our four (2.6 µs, a single pass: the O(n) best case of Lesson 07.8). Our merge sort and quicksort
   do not notice that the data is sorted, so they still do all their levels. Our quicksort is even slower on sorted input than on random input
   (511 µs); the swaps of the partition step move a lot of items around.
4. The standard library sorts are about **5 to 6 times faster** than ours on random data, and they handle sorted input as fast as insertion sort
   (2.6 µs): they detect an already ordered run in the data and stop. They have many more tricks (for example special handling of tiny
   slices), refined over years. Our versions teach the ideas; they do not compete.

> ⚡ **Performance:** An algorithm with a better growth rate (n log n) wins by a large factor already at 10,000 items.
> Only after that do details matter: the standard library's careful implementation is another factor of 5 faster than a textbook
> version. This is why "use the standard library" is the right default.

## 6. What to use in real code

- `sort()` is **stable**. The standard documentation says the current implementation is based on *driftsort*, which
  combines the fast average case of quicksort with the fast worst case of merge sort, and needs only linear time on sorted and reversed input.
- `sort_unstable()` is in place and not stable. The documentation says it is based on *ipnsort*, which combines the fast average case of quicksort
  with the fast worst case of heapsort (a sort that Lesson 07.12 explains), so O(n log n) holds for every input.
- To find only the k-th smallest item (the median, the top 10), you do not need a full sort:
  `v.select_nth_unstable(k)` runs in O(n). It is the standard library's quickselect (Exercise 4 builds a simple one).

You will not write a sorting algorithm in a real project. You will, however, meet "divide and conquer" and "partition" again in
trees, in searching and in many other algorithms.

## Common mistakes

**1. Two `&mut` slices of the same slice at once.** It looks natural to take both halves first and use them later:

```rust
pub fn sort_halves<T: Ord>(v: &mut [T]) {
    let mid = v.len() / 2;
    let left = &mut v[..mid];
    let right = &mut v[mid..];
    left.sort();
    right.sort();
}
```

```text
error[E0499]: cannot borrow `*v` as mutable more than once at a time
 --> src/lib.rs:4:22
  |
3 |     let left = &mut v[..mid];
  |                     - first mutable borrow occurs here
4 |     let right = &mut v[mid..];
  |                      ^ second mutable borrow occurs here
5 |     left.sort();
  |     ---- first borrow later used here
  |
  = help: use `.split_at_mut(position)` to obtain two mutable non-overlapping sub-slices
```

The compiler cannot see that `..mid` and `mid..` do not overlap. The help line gives the fix: `let (left, right) = v.split_at_mut(mid);`
is a safe function that proves the parts are separate. (Our `merge_sort_rec` is fine because each borrow ends before the next one begins.)

**2. A naive pivot.** The simplest quicksort uses the last item as the pivot and a two-way partition:

```rust
fn naive_quicksort<T: Ord>(v: &mut [T]) {
    if v.len() <= 1 {
        return;
    }
    let last = v.len() - 1; // the last item is the pivot
    let mut store = 0;
    for i in 0..last {
        if v[i] < v[last] {
            v.swap(i, store);
            store += 1;
        }
    }
    v.swap(store, last);
    naive_quicksort(&mut v[..store]);
    naive_quicksort(&mut v[store + 1..]);
}
```

It is correct, and it is fast on random data. But on an input that is **already sorted**, the pivot is always the biggest item,
so one part has `n − 1` items and the other has none. Timing it on `0..n` in a release build, with `Instant` (Lesson 05.12):

```text
n = 10000: 31.741333ms
n = 20000: 78.712625ms
n = 40000: 295.459708ms
```

Doubling `n` makes it about four times slower: that is O(n²). Sorted data is very common in real programs, so this "simple"
version is a trap. Median-of-three, the three-way partition and recursing into the smaller part are the three cheap fixes
that our `quicksort` uses.

## Try it yourself

1. Add `merge_sort`, `quicksort` and the helper functions to `src/sorting.rs`, and the new tests to the `tests` module. Run `cargo test`.
2. Add the benchmark and run it. Compare your numbers with the table; the ratios should look similar.
3. Change `SMALL` from 16 to 2 and run the benchmark again. Is `quicksort` faster or slower on random input? Why do you think that is?
4. Run `cargo fmt` and `cargo clippy --all-targets`, then commit: `git add code/p07-algorithms && git commit -m "feat(p07): merge sort and quicksort"`.

## Exercises

### Exercise 1 — Merge by hand (easy)

Merge the sorted lists `[2, 5, 8]` and `[1, 5, 9, 10]` like `merge_sort_rec` does. Write down each comparison. In which order do the two
`5`s come out (the first `5` is from the left list)? How many comparisons were made?

<details>
<summary>Hint</summary>

Take from the right list only if its front item is strictly smaller than the front item of the left list. When one list is empty, you stop comparing.

</details>

<details>
<summary>Answer</summary>

```text
2 vs 1  take 1     out = [1]
2 vs 5  take 2     out = [1 2]
5 vs 5  take 5 (left, tie)   out = [1 2 5]
8 vs 5  take 5 (right)       out = [1 2 5 5]
8 vs 9  take 8     out = [1 2 5 5 8]
left is empty: copy 9, 10    out = [1 2 5 5 8 9 10]
```

5 comparisons. On the tie `5 vs 5` the left `5` is taken first, so equal items keep their original order. This is the
stability rule of merge sort.

</details>

### Exercise 2 — Test the borders (easy)

Write a test for `quicksort` that checks the lengths 15, 16, 17, 18, 32 and 33 against the standard library. Why are these lengths interesting?

<details>
<summary>Hint</summary>

Use `random_vec(7, len, 100)` to make the input, clone it, sort the clone with `.sort()`, and compare. Think about the constant `SMALL`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn quicksort_boundaries() {
        for len in [15, 16, 17, 18, 32, 33] {
            let mut actual = random_vec(7, len, 100);
            let mut expected = actual.clone();
            expected.sort();
            quicksort(&mut actual);
            assert_eq!(actual, expected, "len {len}");
        }
    }
```

Why it works: with `SMALL = 16` the lengths up to 16 are sorted only by `insertion_sort`, the length 17 is the smallest one that
runs `partition3` at all, and 32 and 33 are where the parts of the first partition can land exactly at the border. Bugs
like "`<` instead of `<=`" hide at borders like these.

</details>

### Exercise 3 — State the complexity (medium)

Answer for each case with a Big-O and one sentence of reason:

1. The time of `merge_sort` on 1,000,000 items that are already sorted.
2. The extra memory of `merge_sort` and of `quicksort` for `n` items.
3. The recursion depth of `merge_sort_rec` for 1,000,000 items.
4. The time of the naive quicksort from "Common mistakes" for a list of `n` equal numbers.

<details>
<summary>Hint</summary>

Ask: how many levels are there, and how much work does one level do? For (4), what does the naive partition do when no item is smaller than the pivot?

</details>

<details>
<summary>Answer</summary>

1. **O(n log n)** (about 20 million steps). Merge sort does not look at the order of the input; every level merges all `n` items.
2. Merge sort: **O(n)** for the scratch buffer. Quicksort: **O(log n)** for the call stack, because our version recurses only into the smaller part.
3. About **20**: halving 1,000,000 items reaches 1 after 20 steps (2²⁰ = 1,048,576), so at most 20 calls wait on the stack at once.
4. **O(n²)**: no item is smaller than the pivot, so every partition leaves `n − 1` items on one side: `n + (n−1) + … + 1` steps. (Our `quicksort` with the three-way
   partition handles this input in O(n): all items fall into the "equal" group in a single pass.)

</details>

### Exercise 4 — Quickselect (hard)

Write `nth_smallest<T: Ord + Clone>(v: &mut [T], k: usize) -> T` (where `k = 0` is the smallest item) in `sorting.rs`, using
`median_of_three` and `partition3`, but **without** sorting everything: after each partition keep only the part that contains position `k`. Write a test
that compares the result with the sorted vector for every `k` and for many lengths. What is the average time complexity?

<details>
<summary>Hint</summary>

Keep two numbers `lo` and `hi` so that the answer is always inside `v[lo..hi]`. Partition `&mut v[lo..hi]`; the borders it returns
are relative to `lo`. If `k` is before the equal group, set `hi = lo + lt`. If `k` is after it, set `lo = lo + gt`. Otherwise the pivot value is the answer.

</details>

<details>
<summary>Answer</summary>

```rust
/// The k-th smallest item (k = 0 is the smallest), found with partitioning only.
pub fn nth_smallest<T: Ord + Clone>(v: &mut [T], k: usize) -> T {
    assert!(k < v.len(), "k must be smaller than the length");
    let (mut lo, mut hi) = (0, v.len()); // the answer is inside v[lo..hi]
    loop {
        let part = &mut v[lo..hi];
        let pivot = median_of_three(part);
        let (lt, gt) = partition3(part, pivot);
        let (lt, gt) = (lo + lt, lo + gt);
        if k < lt {
            hi = lt;
        } else if k >= gt {
            lo = gt;
        } else {
            return v[k].clone();
        }
    }
}
```

The test, inside the `tests` module:

```rust
    #[test]
    fn nth_smallest_matches_sorting() {
        for len in 1..60 {
            let data = random_vec(len as u64, len, 20);
            let mut sorted = data.clone();
            sorted.sort();
            for (k, expected) in sorted.iter().enumerate() {
                let mut copy = data.clone();
                assert_eq!(&nth_smallest(&mut copy, k), expected, "len {len}, k {k}");
            }
        }
    }
```

Why it works: after `partition3`, the positions `lt..gt` hold the pivot value in their final sorted places. If `k` falls
there, we are done. If not, the answer is in one side only, so we throw away the other one. With good pivots the part is about
half as big each time: `n + n/2 + n/4 + … ≈ 2n`, so the average time is **O(n)**, less than the O(n log n) of a full sort.

</details>

## Quiz

1. What is the main difference between how merge sort and quicksort do their work?

<details>
<summary>Answer</summary>

Merge sort splits blindly in halves and does its work while merging on the way back up. Quicksort does its work
while going down (partitioning around a pivot), and has nothing to combine afterwards.

</details>

2. Why is merge sort stable and our quicksort not?

<details>
<summary>Answer</summary>

When merging, a tie takes the left item first, so equal items keep their order. Partitioning swaps items over long
distances and can jump an item over an equal one.

</details>

3. What input makes a quicksort with a bad pivot rule take O(n²) time?

<details>
<summary>Answer</summary>

One where the pivot is always the smallest or biggest item, such as an already sorted or reversed list with the first or last item as the
pivot. The parts have sizes 0 and `n − 1`.

</details>

4. Why does `quicksort` recurse into the smaller part and loop on the bigger one?

<details>
<summary>Answer</summary>

Each recursive call then works on at most half of the items, so the call stack never grows deeper than about log₂ n, even if the pivots are bad.

</details>

5. Which standard-library method gives you the median of a big slice without sorting it completely?

<details>
<summary>Answer</summary>

`select_nth_unstable(k)` with `k = len / 2`. It runs in O(n).

</details>

## Summary

- Divide and conquer splits a problem into smaller copies of itself. Halving gives about log₂ n levels, so a sort that does O(n) work per level costs O(n log n).
- Merge sort: sort both halves, merge them by repeatedly taking the smaller front item. O(n log n) always, O(n) extra space, stable.
- Quicksort: choose a pivot, partition around it, sort both sides. O(n log n) on average, O(n²) in the worst case, O(log n) extra space, not stable.
- A good quicksort needs a good pivot (median-of-three), a three-way partition for equal items, recursion into the smaller part only, and insertion sort for short slices.
- Test with border lengths, many duplicates and sorted, reversed and equal inputs, and compare with `std`.
- The measured result: at 10,000 items both are over 20 times faster than insertion sort, and the standard `sort` is again 5 times faster than our versions.
- Use `sort`, `sort_unstable` and `select_nth_unstable` in real code.

## Sources and further reading

- [Rust standard library — slice::sort](https://doc.rust-lang.org/std/primitive.slice.html#method.sort) — the stable sort, with a description of the current implementation.
- [Rust standard library — slice::sort_unstable](https://doc.rust-lang.org/std/primitive.slice.html#method.sort_unstable) — the unstable sort and its guarantees.
- [Rust standard library — slice::select_nth_unstable](https://doc.rust-lang.org/std/primitive.slice.html#method.select_nth_unstable) — finding the k-th item in O(n).
- [Rust standard library — slice::split_at_mut](https://doc.rust-lang.org/std/primitive.slice.html#method.split_at_mut) — two non-overlapping mutable slices.
- [The Rustonomicon — Splitting borrows](https://doc.rust-lang.org/nomicon/borrow-splitting.html) — why the compiler allows `split_at_mut` and how it is built.

---

[🏠 Start](../START-HERE.md) · [← 07.8 Simple sorting algorithms](08-simple-sorting-algorithms.md) · [📋 Phase 07](README.md) · [07.10 Binary search →](10-binary-search.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
