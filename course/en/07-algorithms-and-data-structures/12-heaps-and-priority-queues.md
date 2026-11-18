<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.11 Trees and binary search trees](11-trees-and-binary-search-trees.md) · [📋 Phase 07](README.md) · [07.13 Graphs: BFS and DFS →](13-graphs-bfs-and-dfs.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/12-heaps-and-priority-queues.md) <!-- nav -->

# Lesson 07.12 — Heaps and priority queues

> **Phase 07 — Algorithms and data structures** · Lesson 12 of 20 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a priority queue is, and how a binary heap implements it with `push` and `pop` in O(log n).
- How a tree can live inside a plain vector (index arithmetic instead of pointers).
- How to write a max-heap and heapsort in Rust, and how to test them against the standard library.
- How to use `BinaryHeap` and `Reverse` from the standard library, and typical uses: schedulers, the k biggest values, merging sorted lists.

## Before you start

- You finished [Lesson 07.11 — Trees and binary search trees](11-trees-and-binary-search-trees.md) and [Lesson 07.9 — Merge sort and quicksort](09-merge-sort-and-quicksort.md).
- You know vectors ([Lesson 04.7](../04-structs-enums-collections-errors/07-vectors.md)), `Option` and tuples, and `#[derive]` for traits ([Lesson 05.2](../05-rust-intermediate/02-traits.md)).
- In `code/p07-algorithms` create `src/heap.rs` and add `pub mod heap;` to `src/lib.rs`.

## New words

| Word | Meaning |
|---|---|
| **priority queue** | A collection where you add items and always take out the item with the **highest priority** first, not the oldest one. |
| **heap** (the data structure) | A tree of values in which every parent is at least as big as its children (a *max-heap*) or at most as big (a *min-heap*). **It has nothing to do with the heap memory of Phase 03**, where `Box` and `Vec` keep their data; they only share the name. |
| **binary heap** | A heap in which every node has at most two children, and the tree is stored in a vector. |
| **heap property** | The rule of a max-heap: every parent is greater than or equal to both of its children. |
| **complete binary tree** | A binary tree where every level is full, except maybe the last one, which is filled from the left. |
| **sift up / sift down** | Moving an item up (swapping with its parent) or down (swapping with its bigger child) until the heap property holds again. |
| **heapify** | Turning an unordered vector into a heap, in O(n). |
| **heapsort** | A sorting algorithm that builds a heap and takes the biggest item out again and again. |
| **`Reverse`** | A wrapper type `std::cmp::Reverse<T>` that flips the order of `T`, so a max-heap behaves as a min-heap. |

## 1. Priority queues

A normal queue gives back items in the order they came (Lesson 07.5). A **priority queue** gives back the most important one first. Examples: an operating system picks the next
task to run, a hospital treats the most urgent patient, a map program (Lesson 07.14) always expands the closest place first.

You could keep a sorted vector, but then adding an item costs O(n). You could keep an unsorted vector, but then finding the biggest costs O(n). A **heap** makes both cheap:

| Operation | Sorted `Vec` | Unsorted `Vec` | Heap |
|---|---|---|---|
| add an item (`push`) | O(n) | O(1) | **O(log n)** |
| look at the biggest (`peek`) | O(1) | O(n) | **O(1)** |
| remove the biggest (`pop`) | O(1) | O(n) | **O(log n)** |

> ⚠️ **Not the same heap:** In Phase 03 the *heap* was the area of memory for data that lives as long as you need (`Box`, `Vec`, `String`). The *heap* of this
> lesson is a data structure, a kind of tree. The names are a historical accident. When we say "heap" in this lesson, we mean the data structure.

## 2. A tree inside a vector

A **binary heap** is a *complete* binary tree: all levels are full except the last, which fills from the left. Because there are no gaps, the tree can be stored in a plain `Vec`, level by level, with no `Box` and no
pointers. The position in the vector tells where the node is in the tree:

```text
index:   0   1   2   3   4   5   6
value:  90  70  80  30  50  60  20

               90   (index 0)
            /        \
         70            80          (indexes 1 and 2)
        /  \          /  \
      30    50      60    20       (indexes 3, 4, 5, 6)

for the node at index i:
    parent         = (i - 1) / 2       (whole-number division)
    left child     = 2 * i + 1
    right child    = 2 * i + 2
```

Check: the node `80` has index 2. Its parent is `(2 - 1) / 2 = 0` (`90`), its children are the indexes 5 and 6 (`60` and `20`). This is a **max-heap**: every parent is at least as big as its children. Only the parent–child
relation is ordered; there is no order between `70` and `80`. This is a weaker rule than the BST (binary search tree) rule of Lesson 07.11, and that is why a heap is cheaper to maintain, and why it can give you only the biggest item, not search for any item.

The biggest item is always at index 0 (the root), so `peek` is O(1). Because the tree is complete, its height is about log₂ n, and both repairs below walk along one path: O(log n).

### Push: add at the end, sift up

Add the new item at the end (the next free place), where the shape stays complete. Then, while it is bigger than its parent, swap them. Pushing `85`:

```text
start:        [90, 70, 80, 30, 50, 60, 20]
append 85:    [90, 70, 80, 30, 50, 60, 20, 85]       85 is the child of index 3 (30)
85 > 30, swap:[90, 70, 80, 85, 50, 60, 20, 30]       now the child of index 1 (70)
85 > 70, swap:[90, 85, 80, 70, 50, 60, 20, 30]       now the child of index 0 (90)
85 < 90: stop.
```

### Pop: take the root, move the last item up, sift down

The biggest item is at index 0. Removing it leaves a hole. Fill the hole with the **last** item of the vector (the shape stays complete), then, while it is smaller than a child, swap it with its **bigger** child.
Popping from the heap above:

```text
start:           [90, 70, 80, 30, 50, 60, 20]
swap root/last:  [20, 70, 80, 30, 50, 60 | 90]       take 90 out: that is the result
sift down 20:    children 70 and 80, the bigger is 80:
                 [80, 70, 20, 30, 50, 60]
                 children of index 2: only 60 (index 5):
                 [80, 70, 60, 30, 50, 20]            20 has no children: stop.
```

## 3. A max-heap in Rust

Put this into `src/heap.rs`:

```rust
//! A binary max-heap in a vector, and heapsort.

/// Moves the item at index `i` up until its parent is not smaller.
fn sift_up<T: Ord>(v: &mut [T], mut i: usize) {
    while i > 0 {
        let parent = (i - 1) / 2;
        if v[i] <= v[parent] {
            break;
        }
        v.swap(i, parent);
        i = parent;
    }
}

/// Moves the item at index `i` down until it is not smaller than its children.
/// Only the first `len` items of `v` belong to the heap.
fn sift_down<T: Ord>(v: &mut [T], mut i: usize, len: usize) {
    loop {
        let (left, right) = (2 * i + 1, 2 * i + 2);
        let mut biggest = i;
        if left < len && v[left] > v[biggest] {
            biggest = left;
        }
        if right < len && v[right] > v[biggest] {
            biggest = right;
        }
        if biggest == i {
            break;
        }
        v.swap(i, biggest);
        i = biggest;
    }
}

/// A priority queue that always gives back the biggest item first.
#[derive(Debug)]
pub struct MaxHeap<T> {
    items: Vec<T>,
}

impl<T: Ord> MaxHeap<T> {
    pub fn new() -> Self {
        MaxHeap { items: Vec::new() }
    }

    /// Builds a heap from a vector in O(n).
    pub fn from_vec(mut items: Vec<T>) -> Self {
        let len = items.len();
        for i in (0..len / 2).rev() {
            sift_down(&mut items, i, len);
        }
        MaxHeap { items }
    }

    pub fn len(&self) -> usize {
        self.items.len()
    }

    pub fn is_empty(&self) -> bool {
        self.items.is_empty()
    }

    /// The biggest item, without removing it.
    pub fn peek(&self) -> Option<&T> {
        self.items.first()
    }

    pub fn push(&mut self, value: T) {
        self.items.push(value);
        let last = self.items.len() - 1;
        sift_up(&mut self.items, last);
    }

    /// Removes and returns the biggest item.
    pub fn pop(&mut self) -> Option<T> {
        if self.items.is_empty() {
            return None;
        }
        let last = self.items.len() - 1;
        self.items.swap(0, last); // the last item goes to the root ...
        let top = self.items.pop(); // ... and the old root leaves
        let len = self.items.len();
        sift_down(&mut self.items, 0, len); // repair the heap
        top
    }
}

impl<T: Ord> Default for MaxHeap<T> {
    fn default() -> Self {
        Self::new()
    }
}
```

**What happens:**

1. `sift_up` repeats: compute the parent index, stop if the item is not bigger than its parent (`<=`), otherwise swap and continue from the parent's place. At most one step per level: O(log n).
2. `sift_down` looks at the item and its two children (if they exist: `left < len`), finds the biggest of the three, and stops if that is the item itself. Otherwise it swaps with the bigger child and continues there.
   The parameter `len` says how many items at the front of the slice are part of the heap; `heap_sort` below needs this.
3. Both functions work on a slice `&mut [T]`, so they work for a `Vec` and for any other slice (the next section uses this).
4. `push` adds at the end and sifts the new item up. `pop` swaps the root with the last item, removes the last item (the old root, which is the result) with `Vec::pop`, and sifts the new root down. An empty heap returns `None`.
5. `from_vec` **heapifies**: it calls `sift_down` for every node that has children, from the last parent (`len / 2 - 1`) back to the root. Items near the bottom need little work, and there are many of them; the total is **O(n)**, better
   than `n` pushes, which would cost O(n log n).
6. `peek` is `items.first()`: the maximum is at index 0.
7. The `Default` implementation is there because Clippy asks for it when a type has a `new()` without arguments (Lesson 07.11).

## 4. Heapsort

A heap gives a neat sorting algorithm. Build a max-heap inside the slice itself. The biggest item is at the front: swap it with the last item of the slice, which is its final place, and shrink the heap by one. The new root is
probably wrong, so sift it down. Repeat. Sorting `[4, 1, 3, 2]`:

```text
start:                 4 1 3 2
build a max-heap:      4 2 3 1               (sift down at index 1: 1 and 2 swap)
swap root with end:    1 2 3 | 4             the part after | is final; repair the heap of 3 items:
                       3 2 1 | 4
swap root with end:    1 2 | 3 4             repair:
                       2 1 | 3 4
swap root with end:    1 | 2 3 4
done:                  1 2 3 4
```

Add the function below `MaxHeap`:

```rust
/// Heapsort: sorts in place in O(n log n), not stable.
pub fn heap_sort<T: Ord>(v: &mut [T]) {
    let n = v.len();
    for i in (0..n / 2).rev() {
        sift_down(v, i, n); // build a max-heap
    }
    for end in (1..n).rev() {
        v.swap(0, end); // the biggest item goes to its final place
        sift_down(v, 0, end); // repair the heap of the first `end` items
    }
}
```

**What happens:** the first loop is the same heapify as `from_vec`. The second loop makes `n − 1` rounds, each with one swap and one `sift_down` of cost O(log n), so the total is **O(n log n) in the best, average and worst case**.
Extra memory: **O(1)**, it is *in place*. It is **not stable**. Heapsort is the safe fallback of faster sorts: its worst case is guaranteed (this is what `sort_unstable` uses as an emergency exit, Lesson 07.9), but on typical data it
is slower than quicksort, because it jumps far around the vector and the processor cannot guess where to read next.

## 5. Testing

The standard library has a `BinaryHeap`, so the model is ready. We also write a helper that checks the heap property directly, and call it after every operation. Add to the end of `src/heap.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;
    use std::collections::BinaryHeap;

    /// Is every parent at least as big as its children?
    fn is_heap<T: Ord>(v: &[T]) -> bool {
        (1..v.len()).all(|i| v[(i - 1) / 2] >= v[i])
    }

    /// `len` pseudo-random numbers from 0 to `max - 1` (as in Lesson 07.8).
    fn random_vec(seed: u64, len: usize, max: u64) -> Vec<i32> {
        let mut rng = Rng::new(seed);
        (0..len).map(|_| (rng.next_u64() % max) as i32).collect()
    }

    #[test]
    fn empty_heap() {
        let mut heap: MaxHeap<i32> = MaxHeap::new();
        assert!(heap.is_empty());
        assert_eq!(heap.peek(), None);
        assert_eq!(heap.pop(), None);
    }

    #[test]
    fn pops_in_decreasing_order() {
        let mut heap = MaxHeap::new();
        for value in [5, 3, 8, 1, 9, 2, 8] {
            heap.push(value);
            assert!(is_heap(&heap.items));
        }
        assert_eq!(heap.len(), 7);
        assert_eq!(heap.peek(), Some(&9));
        let mut out = Vec::new();
        while let Some(value) = heap.pop() {
            assert!(is_heap(&heap.items));
            out.push(value);
        }
        assert_eq!(out, [9, 8, 8, 5, 3, 2, 1]);
    }

    #[test]
    fn behaves_like_std_binary_heap() {
        for seed in 1..=5u64 {
            let mut rng = Rng::new(seed);
            let mut ours = MaxHeap::new();
            let mut model = BinaryHeap::new();
            for _ in 0..1000 {
                if rng.below(3) == 0 {
                    assert_eq!(ours.pop(), model.pop());
                } else {
                    let value = rng.below(100) as i32;
                    ours.push(value);
                    model.push(value);
                }
                assert_eq!(ours.peek(), model.peek());
                assert_eq!(ours.len(), model.len());
            }
        }
    }

    #[test]
    fn from_vec_builds_a_heap() {
        for len in 0..50 {
            let heap = MaxHeap::from_vec(random_vec(len as u64 + 1, len, 30));
            assert!(is_heap(&heap.items));
            assert_eq!(heap.len(), len);
        }
    }

    #[test]
    fn heap_sort_matches_std() {
        for len in 0..60 {
            for seed in 1..=4 {
                for max in [3, 1000] {
                    let mut actual = random_vec(seed, len, max);
                    let mut expected = actual.clone();
                    expected.sort();
                    heap_sort(&mut actual);
                    assert_eq!(actual, expected, "len {len}, seed {seed}, max {max}");
                }
            }
        }
    }
}
```

**What happens:**

1. `is_heap` checks the definition for every index `i ≥ 1`: its parent `(i - 1) / 2` must be at least as big. The tests call it after **every** push and pop, so a bug is found at the operation that caused it.
2. `pops_in_decreasing_order` pushes seven numbers (with a duplicate `8`), then pops all of them. A heap that always pops the maximum must give the numbers in decreasing order: it sorts. This is exactly why `pop` is the sorting step of heapsort.
3. `behaves_like_std_binary_heap` runs 1,000 random operations per seed, about one third pops and two thirds pushes of numbers below 100 (many duplicates), on our heap and on `std::collections::BinaryHeap`, and compares `pop`, `peek` and `len`
   every time (differential testing, Lesson 07.8).
4. `from_vec_builds_a_heap` checks heapify for every length from 0 to 49. `heap_sort_matches_std` compares heapsort with `sort` as in the earlier sorting lessons.

```bash
cd ~/projects/rust/code/p07-algorithms
cargo test heap
```

```text
running 5 tests
test heap::tests::empty_heap ... ok
test heap::tests::pops_in_decreasing_order ... ok
test heap::tests::from_vec_builds_a_heap ... ok
test heap::tests::behaves_like_std_binary_heap ... ok
test heap::tests::heap_sort_matches_std ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 41 filtered out; finished in 0.00s
```

(The 41 filtered-out tests come from the main text of Lessons 07.1 to 07.11; with the tests of the exercises your number is bigger.)

## 6. The standard library: `BinaryHeap` and `Reverse`

In real code use `std::collections::BinaryHeap`. It is a max-heap with the same methods: `push`, `pop`, `peek`, `len`, `is_empty`, `BinaryHeap::from(vec)` (heapify) and `into_sorted_vec()`.
Its items must implement `Ord`. For a **min-heap** wrap the items in `Reverse`. A struct gets its order from `#[derive(PartialOrd, Ord)]`, which compares the fields **in the order they are written**.
Create `examples/e12_scheduler.rs`:

```rust
use std::cmp::Reverse;
use std::collections::BinaryHeap;

/// The derived `Ord` compares `priority` first, then `name`.
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
struct Task {
    priority: u8,
    name: String,
}

fn main() {
    let mut queue = BinaryHeap::new();
    for (priority, name) in [
        (2, "write report"),
        (9, "fix outage"),
        (5, "review code"),
        (9, "call customer"),
    ] {
        queue.push(Task {
            priority,
            name: name.to_string(),
        });
    }

    println!("next: {:?}", queue.peek().map(|task| &task.name));
    while let Some(task) = queue.pop() {
        println!("{} {}", task.priority, task.name);
    }

    // A min-heap: wrap the items in `Reverse`, which flips the order.
    let mut smallest_first = BinaryHeap::new();
    for n in [5, 1, 8, 3] {
        smallest_first.push(Reverse(n));
    }
    while let Some(Reverse(n)) = smallest_first.pop() {
        print!("{n} ");
    }
    println!();
}
```

```bash
cargo run --example e12_scheduler
```

```text
next: Some("fix outage")
9 fix outage
9 call customer
5 review code
2 write report
1 3 5 8 
```

**What happens:**

1. `Task` compares by `priority` first. The two tasks with priority 9 are compared by `name`: `"fix outage"` is greater than `"call customer"` because `f` comes after `c`, so it comes out first.
   If you want the older task first among equals, add a counter field and compare it in `Reverse`.
2. `queue.peek()` looks at the best task without taking it; `pop` removes the tasks one by one in priority order.
3. `Reverse(n)` is a tuple struct that flips the comparison. The max-heap now pops the smallest number first, and the pattern `Some(Reverse(n))` takes the number out of the wrapper.

You will use `BinaryHeap` with `Reverse` for Dijkstra's shortest path algorithm in Lesson 07.14, where the heap holds "the closest unvisited place".

## Common mistakes

**1. Forgetting that `BinaryHeap` is a max-heap.** You wanted the smallest number, but you get the biggest:

```rust
use std::collections::BinaryHeap;

fn main() {
    let mut heap = BinaryHeap::new();
    for n in [5, 1, 8, 3] {
        heap.push(n);
    }
    println!("first pop: {:?}", heap.pop());
}
```

```text
first pop: Some(8)
```

Wrap the numbers in `Reverse` (Section 6) to get `1` first.

**2. Reading a heap in order with `iter()`.** The iterator walks the internal vector, which is **not sorted**; only the first item is guaranteed to be the biggest:

```rust
use std::collections::BinaryHeap;

fn main() {
    let heap = BinaryHeap::from(vec![1, 5, 2, 8, 3]);
    println!("iter:   {:?}", heap.iter().collect::<Vec<_>>());
    println!("sorted: {:?}", heap.into_sorted_vec());
}
```

```text
iter:   [8, 5, 2, 1, 3]
sorted: [1, 2, 3, 5, 8]
```

To get the items in order, `pop` them one by one (which empties the heap), or call `into_sorted_vec()` (ascending order). Do not rely on the order of `iter()`.

**3. Putting `f64` into a heap.** Floating-point numbers have no total order (`NaN`, Lesson 07.8), so they do not implement `Ord`:

```rust
use std::collections::BinaryHeap;

fn main() {
    let mut heap = BinaryHeap::new();
    heap.push(2.5_f64);
    heap.push(0.5);
}
```

```text
error[E0277]: the trait bound `f64: Ord` is not satisfied
 --> src/main.rs:5:10
  |
5 |     heap.push(2.5_f64);
  |          ^^^^ ------- `f64` doesn't satisfy the trait bound
  |          |
  |          the trait `Ord` is not implemented for `f64`
```

The compiler says that `f64` is not `Ord`. Fix: wrap the number in a small struct that implements `Ord` with `f64::total_cmp`, or use integer priorities (for example, milliseconds instead of seconds as `f64`).

## Try it yourself

1. Add `heap.rs` step by step: the two sift functions, `MaxHeap`, `heap_sort` and the tests. Run `cargo test heap`.
2. In `sift_up`, change `<=` to `>=`. Run the tests. Which tests fail (two of them do), and what does the message of `pops_in_decreasing_order` say? Change it back.
3. Run `cargo run --example e12_scheduler`. Change `Task` so that `name` is written before `priority` in the struct. What is the order now, and why?
4. Run `cargo fmt`, `cargo clippy --all-targets`, and commit: `git add code/p07-algorithms && git commit -m "feat(p07): binary heap and heapsort"`.

## Exercises

### Exercise 1 — Trace a heap (easy)

Push `5, 3, 8, 1, 9` one by one into an empty max-heap and write down the vector after each push. Then do one `pop` and write down the vector.

<details>
<summary>Hint</summary>

After each push, compare the new last item with its parent `(i - 1) / 2` and swap while it is bigger. For `pop`, swap the first and the last items, remove the last, and sift the new first item down along its bigger child.

</details>

<details>
<summary>Answer</summary>

```text
push 5:  [5]
push 3:  [5, 3]
push 8:  [5, 3, 8] -> 8 > 5, swap:           [8, 3, 5]
push 1:  [8, 3, 5, 1]                        1 < 3, stays
push 9:  [8, 3, 5, 1, 9] -> 9 > 3, swap:     [8, 9, 5, 1, 3]
                         -> 9 > 8, swap:     [9, 8, 5, 1, 3]
pop:     swap first and last:                [3, 8, 5, 1 | 9]  -> 9 is the result
         sift down 3: bigger child is 8:     [8, 3, 5, 1]
         children of 3 (index 1): only 1:    1 < 3, stop
```

Result: after the pop the vector is `[8, 3, 5, 1]`.

</details>

### Exercise 2 — A property test (easy)

Write a test that mixes pushes and pops of 300 pseudo-random numbers (a pop after every third push) and checks `is_heap` after every operation.

<details>
<summary>Hint</summary>

Use `random_vec(5, 300, 50)` and `enumerate()`. After each `push`, if `i % 3 == 2`, call `heap.pop()`. Then `assert!(is_heap(&heap.items))`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn is_heap_holds_for_random_pushes_and_pops() {
        let mut heap = MaxHeap::new();
        for (i, value) in random_vec(5, 300, 50).into_iter().enumerate() {
            heap.push(value);
            if i % 3 == 2 {
                heap.pop();
            }
            assert!(is_heap(&heap.items));
        }
    }
```

Why it works: a property such as "every parent is at least as big as its children" is checked at every step, with a small value range (0 to 49), so there are many duplicates. If `sift_down` ever picked the wrong child, `is_heap` would fail right at that operation.

</details>

### Exercise 3 — State the complexity (medium)

1. What do `n` calls of `push` cost, compared with `from_vec` on `n` items?
2. What does it cost to get the items in sorted order by `n` calls of `pop`?
3. You have a stream of `n` numbers and want the `k` biggest. A min-heap that keeps at most `k` items (when it grows to `k + 1`, pop the smallest) solves it: what are the time and the extra memory, in terms of `n` and `k`?

<details>
<summary>Hint</summary>

For (3), each of the `n` numbers is pushed once and popped at most once, and the heap has at most `k + 1` items, so each operation costs O(log k).

</details>

<details>
<summary>Answer</summary>

1. `n` pushes cost **O(n log n)**; `from_vec` costs **O(n)**, because most items sit near the bottom and move only a few steps.
2. **O(n log n)**: `n` pops of O(log n) each. This is heapsort with an extra copy.
3. Time **O(n log k)**, extra memory **O(k)**. If `k` is much smaller than `n`, this is better than sorting everything (O(n log n), O(n) memory), and it works on a stream you cannot keep in memory.

</details>

### Exercise 4 — The k biggest (medium)

Write `k_largest(values: &[i32], k: usize) -> Vec<i32>` that returns the `k` biggest values, biggest first (fewer if there are fewer than `k` values), using a `BinaryHeap` of `Reverse` values as in Exercise 3. Write a test that compares it with sorting.

<details>
<summary>Hint</summary>

Push `Reverse(value)` for each value; when `heap.len() > k`, `pop` (this drops the smallest). At the end, turn the heap into a vector of plain numbers with `into_iter().map(|Reverse(v)| v)`, then sort it in decreasing order. Add `use std::cmp::Reverse; use std::collections::BinaryHeap;` at the top of the file.

</details>

<details>
<summary>Answer</summary>

```rust
/// The `k` biggest values of `values`, biggest first.
pub fn k_largest(values: &[i32], k: usize) -> Vec<i32> {
    // A min-heap holding the k biggest values seen so far; its root is the smallest of them.
    let mut heap = BinaryHeap::with_capacity(k + 1);
    for &value in values {
        heap.push(Reverse(value));
        if heap.len() > k {
            heap.pop(); // drop the smallest: it cannot be among the k biggest
        }
    }
    let mut result: Vec<i32> = heap.into_iter().map(|Reverse(v)| v).collect();
    result.sort_by(|a, b| b.cmp(a));
    result
}
```

The test, inside the `tests` module:

```rust
    #[test]
    fn k_largest_matches_sorting() {
        for len in 0..40 {
            let values = random_vec(len as u64 + 3, len, 20);
            let mut sorted = values.clone();
            sorted.sort_by(|a, b| b.cmp(a));
            for k in 0..=len + 2 {
                let expected: Vec<i32> = sorted.iter().take(k).copied().collect();
                assert_eq!(k_largest(&values, k), expected, "len {len}, k {k}");
            }
        }
    }
```

Why it works: the heap always contains the `k` biggest values seen so far, and the smallest of them is at the root, ready to be thrown out when a bigger value arrives. The test covers `k = 0` and `k` bigger than the length, where `take(k)` gives fewer items.

</details>

### Exercise 5 — Merge sorted lists (hard)

Write `merge_sorted(lists: &[Vec<i32>]) -> Vec<i32>` that merges `k` sorted lists into one sorted vector using a min-heap that holds at most one entry per list. What is the time complexity, for `N` values in total? Test it against `concat` plus `sort`.

<details>
<summary>Hint</summary>

Put `Reverse((value, list_index, position))` into the heap for the first item of each non-empty list. Pop the smallest entry, push its value to the result, and push the next item of the same list (`lists[list_index].get(position + 1)`), if there is one. Tuples compare field by field, so the smallest value comes first.

</details>

<details>
<summary>Answer</summary>

```rust
/// Merges sorted lists into one sorted vector in O(N log k) for k lists with N values in total.
pub fn merge_sorted(lists: &[Vec<i32>]) -> Vec<i32> {
    // Each entry is (value, which list, position in that list); the smallest comes out first.
    let mut heap = BinaryHeap::new();
    for (list_index, list) in lists.iter().enumerate() {
        if let Some(&first) = list.first() {
            heap.push(Reverse((first, list_index, 0)));
        }
    }
    let mut merged = Vec::new();
    while let Some(Reverse((value, list_index, position))) = heap.pop() {
        merged.push(value);
        if let Some(&next) = lists[list_index].get(position + 1) {
            heap.push(Reverse((next, list_index, position + 1)));
        }
    }
    merged
}
```

The test, inside the `tests` module:

```rust
    #[test]
    fn merge_sorted_matches_sorting() {
        let lists = vec![vec![1, 4, 9], vec![], vec![2, 2, 8, 10], vec![3]];
        assert_eq!(merge_sorted(&lists), [1, 2, 2, 3, 4, 8, 9, 10]);
        for seed in 1..=5u64 {
            let lists: Vec<Vec<i32>> = (0..6)
                .map(|i| {
                    let mut v = random_vec(seed * 10 + i, (i * 3) as usize, 100);
                    v.sort();
                    v
                })
                .collect();
            let mut expected: Vec<i32> = lists.concat();
            expected.sort();
            assert_eq!(merge_sorted(&lists), expected);
        }
    }
```

Why it works: the heap holds the front item of each list, so its smallest entry is the smallest value not yet used. Every value is pushed once and popped once, in a heap of at most `k` entries: **O(N log k)**. This is the merge step of Lesson 07.9 for `k` lists instead of two, and it is how external sorting merges big sorted files.

</details>

## Quiz

1. What is the heap property of a max-heap?

<details>
<summary>Answer</summary>

Every parent is greater than or equal to both of its children. So the biggest item is at the root.

</details>

2. For the node at index `i` in a vector-based binary heap, where are its parent and its children?

<details>
<summary>Answer</summary>

Parent at `(i - 1) / 2`, children at `2 * i + 1` and `2 * i + 2`.

</details>

3. Why is `pop` O(log n)?

<details>
<summary>Answer</summary>

It swaps the root with the last item and sifts it down. The sift goes down one level at each step, and a complete binary tree of `n` items has about log₂ n levels.

</details>

4. How do you get a min-heap from `BinaryHeap`?

<details>
<summary>Answer</summary>

Wrap the items in `std::cmp::Reverse`: `heap.push(Reverse(x))`, and read them back with `Some(Reverse(x)) = heap.pop()`.

</details>

5. Is the heap from this lesson the same as the heap memory of Phase 03?

<details>
<summary>Answer</summary>

No. This heap is a data structure (a tree stored in a vector, where parents are bigger than children). The heap memory is the area where a program keeps data that outlives a function call. Only the name is shared.

</details>

## Summary

- A priority queue gives back the item with the highest priority first. A binary heap makes `push` and `pop` O(log n) and `peek` O(1).
- The heap is a complete binary tree stored in a vector: the parent of `i` is `(i - 1) / 2`, the children are `2i + 1` and `2i + 2`. No pointers are needed.
- `push` adds at the end and sifts up; `pop` moves the last item to the root and sifts down; heapify builds a heap in O(n).
- Heapsort builds a heap in the slice and repeatedly moves the root to the end: O(n log n) always, O(1) extra memory, not stable.
- A heap orders only parents against children, so it cannot search or iterate in order; `iter()` of a `BinaryHeap` is not sorted.
- Use `std::collections::BinaryHeap` (a max-heap), `Reverse` for a min-heap, and `#[derive(Ord)]` with the most important field first.
- Typical uses: schedulers, the k biggest values in O(n log k), merging k sorted lists in O(N log k), and Dijkstra's algorithm (Lesson 07.14).

## Sources and further reading

- [Rust standard library — std::collections::BinaryHeap](https://doc.rust-lang.org/std/collections/struct.BinaryHeap.html) — the max-heap: methods, a Dijkstra example and the cost of each operation.
- [Rust standard library — std::cmp::Reverse](https://doc.rust-lang.org/std/cmp/struct.Reverse.html) — the wrapper that flips the order.
- [Rust standard library — Ord and derive](https://doc.rust-lang.org/std/cmp/trait.Ord.html) — how the order of fields decides a derived `Ord`.
- [Rust standard library — slice::select_nth_unstable](https://doc.rust-lang.org/std/primitive.slice.html#method.select_nth_unstable) — when you need only the k-th item and not a heap or a full sort.

---

[🏠 Start](../START-HERE.md) · [← 07.11 Trees and binary search trees](11-trees-and-binary-search-trees.md) · [📋 Phase 07](README.md) · [07.13 Graphs: BFS and DFS →](13-graphs-bfs-and-dfs.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
