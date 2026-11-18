<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.2 Analyzing time and space](02-analyzing-time-and-space.md) · [📋 Phase 07](README.md) · [07.4 Linked lists →](04-linked-lists.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/03-arrays-and-vectors-under-the-hood.md) <!-- nav -->

# Lesson 07.3 — Arrays and vectors under the hood

> **Phase 07 — Algorithms and data structures** · Lesson 3 of 20 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How an array sits in memory, and why reading `v[i]` costs the same for every `i`.
- What a `Vec` really is: three numbers on the stack and one block of items on the heap.
- Why `push` is amortized `O(1)` while `insert(0, x)` and `remove(0)` are `O(n)`.
- How to build a small growable array yourself, count its moves, and test it against `Vec` on thousands of random operations.

## Before you start

- You finished [Lesson 07.2 — Analyzing time and space](02-analyzing-time-and-space.md).
- You know vectors ([Lesson 04.7](../04-structs-enums-collections-errors/07-vectors.md), especially length and capacity), the stack and the heap ([Lesson 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)) and slices ([Lesson 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md)).

## New words

| Word | Meaning |
|---|---|
| **contiguous** | Placed one right after another, with no gaps. The items of an array are contiguous. |
| **address** | The number of a byte in memory. Like a house number: it says where something is. |
| **random access** | Reading any item directly by its position, in the same time for every position. |
| **reallocation** | Getting a bigger block of memory, copying the items into it and giving the old block back. |
| **shifting** | Moving a group of items one place to the left or right to open or close a gap. |
| **pseudo-random numbers** | Numbers that look random but come from a fixed formula. The same starting value (the *seed*) gives the same numbers, so a failing test can be repeated. |
| **RAM** (Random Access Memory) | The fast working memory of the computer. |

## 1. Memory is a long row of numbered boxes

The memory (RAM) of a computer is like a very long street where every byte has a house number, its
**address**. An **array** stores its items **contiguously**: item 0, then item 1 right after it, then item 2,
with no gaps and with every item the same size. Because of that, the address of item `i` is a simple
calculation:

```text
address of item i  =  address of item 0  +  i * (size of one item)
```

```text
address:   1000   1004   1008   1012
          ┌──────┬──────┬──────┬──────┐
 [i32; 4] │  10  │  20  │  30  │  40  │      each i32 takes 4 bytes
          └──────┴──────┴──────┴──────┘
 index:      0      1      2      3        item 3 is at 1000 + 3 * 4 = 1012
```

One multiplication and one addition, whatever `i` is. That is why `v[i]` is `O(1)`: this is called **random
access**. The same trick explains why every item must have the same size, and why you cannot put a
`String` of unknown length directly into an array: you would not know where item `i` begins. (A `Vec<String>`
stores the small fixed-size handles of the strings, not their text.)

Run `examples/e03_layout.rs` to see the real numbers:

```rust
use std::mem::size_of;

fn main() {
    let numbers: [i32; 4] = [10, 20, 30, 40];
    let first = &numbers[0] as *const i32 as usize;
    let second = &numbers[1] as *const i32 as usize;
    println!("bytes between neighbours: {}", second - first);
    println!("[i32; 4]  takes {} bytes", size_of::<[i32; 4]>());
    println!(
        "&[i32]    takes {} bytes (address + length)",
        size_of::<&[i32]>()
    );
    println!(
        "Vec<i32>  takes {} bytes (address + length + capacity)",
        size_of::<Vec<i32>>()
    );
}
```

**What happens:**

1. `&numbers[0] as *const i32 as usize` turns a reference into a plain address number (a *raw pointer*,
   Lesson 06.6, converted to an integer). We only print it; we never read through it, so no `unsafe` is needed.
2. The two addresses differ by the size of one `i32`.
3. `size_of` tells how many bytes a type takes on the stack.

```text
bytes between neighbours: 4
[i32; 4]  takes 16 bytes
&[i32]    takes 16 bytes (address + length)
Vec<i32>  takes 24 bytes (address + length + capacity)
```

A slice reference `&[i32]` is two numbers: the address of the first item and the length. A `Vec<i32>` is three:
the address of its block on the heap, the length, and the capacity. The items themselves live in one contiguous
block on the heap:

```text
stack                          heap (one contiguous block of 8 slots)
┌─────────────────┐            ┌───┬───┬───┬───┬───┬───┬───┬───┐
│ ptr      ●──────┼───────────▶│ 0 │ 1 │ 2 │ 3 │ 4 │ ? │ ? │ ? │
│ len      5      │            └───┴───┴───┴───┴───┴───┴───┴───┘
│ capacity 8      │             ◀── len = 5 ──▶◀─ spare room ─▶
└─────────────────┘
```

## 2. What each operation costs

Look at the picture and the costs follow:

| Operation | Cost | Why |
|---|---|---|
| `v[i]`, `v.get(i)`, `v.len()` | `O(1)` | one address calculation |
| `v.push(x)` | amortized `O(1)` | write after the last item; sometimes reallocate |
| `v.pop()` | `O(1)` | forget the last item |
| `v.insert(i, x)` | `O(n - i)` | the items after `i` shift one place right |
| `v.remove(i)` | `O(n - i)` | the items after `i` shift one place left |
| `v.contains(&x)` | `O(n)` | look at the items one by one |
| `v.swap_remove(i)` | `O(1)` | put the last item into the gap (the order changes) |

Inserting or removing at the **front** is the worst case: all `n` items shift. Inserting or removing at the
**end** is the best case: nothing shifts.

## 3. Build a growable array

The real `Vec` uses uninitialized memory and `unsafe` code. We can build the same *idea* in safe Rust with a
fixed-size box of slots, `Box<[Option<T>]>`: a slot holds `Some(item)` or `None` (empty). Create
`src/dyn_array.rs` and add `pub mod dyn_array;` to `src/lib.rs`. First the struct and the simple methods:

```rust
//! Lesson 07.3: a growable array, built from a fixed-size box of slots.

/// A growable array. `slots` is a fixed-size block of memory; the first `len`
/// slots hold `Some(item)`, the rest hold `None`.
pub struct DynArray<T> {
    slots: Box<[Option<T>]>,
    len: usize,
    moves: usize, // how many items were moved so far (growing and shifting)
}

impl<T> DynArray<T> {
    pub fn new() -> DynArray<T> {
        DynArray {
            slots: Box::new([]),
            len: 0,
            moves: 0,
        }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }

    pub fn capacity(&self) -> usize {
        self.slots.len()
    }

    pub fn moves(&self) -> usize {
        self.moves
    }

    /// Reading by position: one address calculation, O(1).
    pub fn get(&self, index: usize) -> Option<&T> {
        if index < self.len {
            self.slots[index].as_ref()
        } else {
            None
        }
    }
```

**What happens:**

1. `slots` is the block (its length is the **capacity**); `len` counts the filled slots at the front; `moves` counts
   how many items have been moved, so that we can measure the cost later.
2. `Box::new([])` is an empty block: capacity 0, nothing allocated yet, like `Vec::new()`.
3. `get` checks `index < self.len` first. Slots at `len` and after are empty, and reading them is not allowed.

`push` and `pop` work at the end:

```rust
    /// Adds an item at the end: O(1), except when the array is full (then O(n)).
    pub fn push(&mut self, value: T) {
        if self.len == self.slots.len() {
            self.grow();
        }
        self.slots[self.len] = Some(value);
        self.len += 1;
    }

    /// Removes the last item: O(1).
    pub fn pop(&mut self) -> Option<T> {
        if self.len == 0 {
            return None;
        }
        self.len -= 1;
        self.slots[self.len].take()
    }
```

**What happens:**

1. `push` first checks whether the block is full (`len` equals the capacity). If so, it calls `grow` (below).
   Then it writes into the first empty slot and increases `len`. That is `O(1)`.
2. `pop` decreases `len` and `take()`s the item out of the slot, leaving `None` behind. `Option::take` replaces
   the value with `None` and returns the old value: it is how you move an item out of a slot you only borrow.

`insert` and `remove` must shift items:

```rust
    /// Inserts at `index`, shifting the items after it to the right: O(n - index).
    pub fn insert(&mut self, index: usize, value: T) {
        assert!(index <= self.len, "insertion index is out of bounds");
        if self.len == self.slots.len() {
            self.grow();
        }
        for i in (index..self.len).rev() {
            self.slots[i + 1] = self.slots[i].take();
            self.moves += 1;
        }
        self.slots[index] = Some(value);
        self.len += 1;
    }

    /// Removes the item at `index`, shifting the items after it to the left: O(n - index).
    pub fn remove(&mut self, index: usize) -> T {
        assert!(index < self.len, "removal index is out of bounds");
        let value = self.slots[index].take().expect("slot below len is filled");
        for i in index..(self.len - 1) {
            self.slots[i] = self.slots[i + 1].take();
            self.moves += 1;
        }
        self.len -= 1;
        value
    }
```

**What happens:**

1. `insert` makes room: the loop walks from the **last** item down to `index` and moves each item one slot to the
   right. It goes from the end so that no item is overwritten before it is moved. Then it writes the new item.
2. `remove` takes the item out and walks upward from `index`, moving every later item one slot to the left to close
   the gap.
3. Both count each move in `moves`. With `index = 0` they move all `len` items: `O(n)`.

Finally `grow`, and the `Default` implementation that Clippy asks for when a type has `new()` without arguments:

```rust
    /// Doubles the capacity (the first capacity is 4) and moves all items: O(n).
    fn grow(&mut self) {
        let new_capacity = if self.slots.is_empty() {
            4
        } else {
            self.slots.len() * 2
        };
        let mut bigger: Box<[Option<T>]> = (0..new_capacity).map(|_| None).collect();
        for i in 0..self.len {
            bigger[i] = self.slots[i].take();
            self.moves += 1;
        }
        self.slots = bigger;
    }
}

impl<T> Default for DynArray<T> {
    fn default() -> DynArray<T> {
        DynArray::new()
    }
}
```

**What happens:**

1. The new capacity is 4 for the first allocation and then doubles: 4, 8, 16, 32...
2. A new block of `None`s is created. (`collect` can build a `Box<[T]>` directly.)
3. Every item is moved into the new block, and the old block is dropped when `self.slots = bigger` replaces it. This is
   the **reallocation**, `O(n)`, and each move is counted.

### Why `push` is amortized O(1)

Let us push many items and look at `moves`. The example `examples/e03_amortized.rs`:

```rust
use p07_algorithms::dyn_array::DynArray;

fn main() {
    println!(
        "{:>9} {:>10} {:>14}",
        "pushes", "item moves", "moves per push"
    );
    for n in [10, 1_000, 100_000, 1_000_000] {
        let mut a = DynArray::new();
        for i in 0..n {
            a.push(i);
        }
        println!(
            "{:>9} {:>10} {:>14.2}",
            n,
            a.moves(),
            a.moves() as f64 / n as f64
        );
    }

    println!();
    let mut b = DynArray::new();
    for i in 0..2_000 {
        b.insert(0, i); // always at the front
    }
    println!("2000 inserts at the front: {} item moves", b.moves());
}
```

```text
   pushes item moves moves per push
       10         12           1.20
     1000       1020           1.02
   100000     131068           1.31
  1000000    1048572           1.05

2000 inserts at the front: 2001044 item moves
```

Reading the output:

- However many items we push, the number of moves stays near **one move per push** (between 1.0 and 1.3). The
  average cost of one push is constant.
- Why: growing happens at sizes 4, 8, 16, ..., and it moves 4, 8, 16, ... items. The last growth moves fewer than
  `n` items, the one before it half as many, and so on, so all growths together move fewer than `2n` items (the table
  shows 1.0 to 1.3 moves per push, never 2). Even with the writes of the pushes themselves that is fewer than `3n` steps
  for `n` pushes: **amortized O(1)**. (If the block grew by a fixed number of slots instead of doubling, the total would
  be quadratic. Doubling is the key.)
- Inserting 2,000 items at the **front** needs about **two million** moves: roughly `n²/2`. That is the price of
  shifting.

## 4. Tests, with pseudo-random numbers

A growable array has many corners: growing, empty, full, insert at the ends. The best test is the one from Lesson 07.1,
but with random inputs: do the same random operations on `DynArray` and on `Vec` (the obviously correct reference) and
compare after every step. For that we need random numbers. Rather than add a crate, we use a tiny
**pseudo-random** generator. Create `src/testutil.rs`, and add `#[cfg(test)] mod testutil;` to `src/lib.rs`
(`#[cfg(test)]` means "only when testing"):

```rust
//! A tiny pseudo-random number generator for tests (the "xorshift64*" recipe).
//! The same seed always gives the same numbers, so a failing test can be repeated.
//! Not for security: never use it for passwords or keys.

pub struct Rng(u64);

impl Rng {
    pub fn new(seed: u64) -> Rng {
        Rng(seed.max(1)) // the state must never be 0
    }

    pub fn next_u64(&mut self) -> u64 {
        let mut x = self.0;
        x ^= x >> 12;
        x ^= x << 25;
        x ^= x >> 27;
        self.0 = x;
        x.wrapping_mul(0x2545_F491_4F6C_DD1D)
    }

    /// A number from 0 to n - 1. `n` must be greater than 0.
    pub fn below(&mut self, n: usize) -> usize {
        (self.next_u64() % n as u64) as usize
    }
}
```

**What happens:**

1. `Rng` holds one `u64`, its state. Each call to `next_u64` scrambles the state with shifts (`>>`, `<<`) and
   the exclusive-or operator `^=`, and multiplies by a large constant with `wrapping_mul` (which wraps around on overflow
   instead of panicking). You do not need to understand why the formula works. You need to know that it gives numbers
   that look random, and that **the same seed always gives the same sequence**, so a failing test can be repeated exactly.
2. `below(n)` returns a number from 0 to `n - 1` using the remainder `%`.

> 🔒 **Security:** This generator is predictable. Never use it for passwords, tokens or keys. For those you need a
> cryptographically secure generator (Phase 10).

Now the tests, at the bottom of `src/dyn_array.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    #[test]
    fn capacity_doubles() {
        let mut a = DynArray::new();
        assert!(a.is_empty());
        let mut seen = Vec::new();
        for i in 0..20 {
            a.push(i);
            if seen.last() != Some(&a.capacity()) {
                seen.push(a.capacity());
            }
        }
        assert_eq!(seen, vec![4, 8, 16, 32]);
    }

    #[test]
    fn pushes_move_fewer_than_two_items_each() {
        for n in 1..=300 {
            let mut a = DynArray::new();
            for i in 0..n {
                a.push(i);
            }
            assert!(a.moves() < 2 * n, "n = {n}, moves = {}", a.moves());
        }
    }

    #[test]
    fn behaves_like_vec_on_random_operations() {
        let mut rng = Rng::new(2024);
        let mut mine: DynArray<u64> = DynArray::new();
        let mut reference: Vec<u64> = Vec::new();
        for step in 0..5000 {
            match rng.below(5) {
                0 | 1 => {
                    let x = rng.next_u64() % 100;
                    mine.push(x);
                    reference.push(x);
                }
                2 => assert_eq!(mine.pop(), reference.pop()),
                3 => {
                    let i = rng.below(reference.len() + 1);
                    let x = rng.next_u64() % 100;
                    mine.insert(i, x);
                    reference.insert(i, x);
                }
                _ => {
                    if !reference.is_empty() {
                        let i = rng.below(reference.len());
                        assert_eq!(mine.remove(i), reference.remove(i));
                    }
                }
            }
            assert_eq!(mine.len(), reference.len(), "step {step}");
            for (i, expected) in reference.iter().enumerate() {
                assert_eq!(mine.get(i), Some(expected), "step {step}, index {i}");
            }
        }
    }
}
```

**What happens:**

1. `capacity_doubles` pushes 20 items and records every new capacity: 4, 8, 16, 32.
2. `pushes_move_fewer_than_two_items_each` turns the amortized argument into a test: for every `n` from 1 to 300,
   the moves stay below `2n`.
3. `behaves_like_vec_on_random_operations` is the main test. It runs 5,000 random steps (the `match rng.below(5)` picks
   push, push, pop, insert or remove), does each on both `DynArray` and `Vec`, and after every step compares the whole
   content. If you break `insert` or `grow`, this test finds it within a few steps, and the message shows the step number.

```text
running 3 tests
test dyn_array::tests::capacity_doubles ... ok
test dyn_array::tests::pushes_move_fewer_than_two_items_each ... ok
test dyn_array::tests::behaves_like_vec_on_random_operations ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 9 filtered out; finished in 0.04s
```

(The order of the lines can differ on your computer.)

## 5. Measuring: push against insert at the front

Now the real `Vec`. We build a vector with `push` and with `insert(0, ..)` for three sizes, each four times bigger than
the last. Add the benchmark program to `Cargo.toml` (`criterion` is already there from Lesson 07.2):

```toml
[[bench]]
name = "vectors"
harness = false
```

`benches/vectors.rs`:

```rust
use criterion::{BenchmarkId, Criterion, criterion_group, criterion_main};
use std::hint::black_box;

fn push_back(n: usize) -> Vec<usize> {
    let mut v = Vec::new();
    for i in 0..n {
        v.push(i);
    }
    v
}

fn insert_front(n: usize) -> Vec<usize> {
    let mut v = Vec::new();
    for i in 0..n {
        v.insert(0, i);
    }
    v
}

fn bench_building(c: &mut Criterion) {
    let mut group = c.benchmark_group("build a vector");
    for n in [2_500usize, 10_000, 40_000] {
        group.bench_with_input(BenchmarkId::new("push at the end", n), &n, |b, &n| {
            b.iter(|| push_back(black_box(n)))
        });
        group.bench_with_input(BenchmarkId::new("insert at the front", n), &n, |b, &n| {
            b.iter(|| insert_front(black_box(n)))
        });
    }
    group.finish();
}

criterion_group!(benches, bench_building);
criterion_main!(benches);
```

Run `cargo bench --bench vectors -- --warm-up-time 1 --measurement-time 3`:

```text
build a vector/push at the end/2500
                        time:   [1.5042 µs 1.5097 µs 1.5156 µs]
build a vector/insert at the front/2500
                        time:   [231.56 µs 233.57 µs 236.28 µs]
build a vector/push at the end/10000
                        time:   [5.3047 µs 5.4668 µs 5.6396 µs]
build a vector/insert at the front/10000
                        time:   [3.8321 ms 3.8632 ms 3.9034 ms]
build a vector/push at the end/40000
                        time:   [40.976 µs 41.214 µs 41.469 µs]
build a vector/insert at the front/40000
                        time:   [92.547 ms 93.003 ms 93.683 ms]
```

(Some lines of the criterion report are left out. The numbers come from an Apple M4 Pro; **yours will differ**.)

**How to read it:**

1. `push` grows roughly in step with `n`: 1.5 µs, 5.5 µs, 41 µs. (The last jump is a little larger than 4 times;
   big blocks of memory cost a bit more, and the exact reason depends on the memory system of the computer.)
2. `insert(0, ..)` grows much faster: 0.23 ms, 3.9 ms, 93 ms. Every time `n` quadruples, the time grows about 16
   times or more: that is `O(n²)` for `n` inserts.
3. At 40,000 items building with `insert(0, ..)` is **more than 2,000 times slower** than with `push`.

> ⚡ **Performance:** The shifting is done by a very fast memory copy, so `insert(0, ..)` is fine for 100 items. It becomes
> a problem when `n` is large. If you need to add or remove at **both ends**, use `VecDeque` (Lesson 07.5), where both
> are `O(1)`.

## 6. When to use the standard library instead

Use `Vec` unless you have a reason not to; it has everything you built here and more.

- Know the size? `Vec::with_capacity(n)` avoids every reallocation (Lesson 04.7).
- Order does not matter and you remove from the middle? `v.swap_remove(i)` is `O(1)`.
- Removing many items by a condition? `v.retain(|x| ...)` does it in one `O(n)` pass. Calling `remove` in a loop is `O(n²)`.
- Adding or removing at the front? `std::collections::VecDeque`.
- Need to read by position fast, and the length is fixed at compile time? A plain array `[T; N]`: it lives on the stack
  and needs no allocation.

## Common mistakes

**1. Inserting beyond the end.** `insert` accepts positions `0..=len`, nothing more. Put this into
`examples/e03_insert_panic.rs` and run `cargo run --example e03_insert_panic`:

```rust
fn main() {
    let mut v = vec![10, 20, 30];
    v.insert(5, 99);
    println!("{v:?}");
}
```

```text

thread 'main' (4733772) panicked at examples/e03_insert_panic.rs:3:7:
insertion index (is 5) should be <= len (is 3)
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

(The number after `thread 'main'` differs on your computer.) The message is exact: the index (5) must be at most the
length (3). Our `DynArray::insert` checks the same rule with `assert!`.

**2. Removing items in a loop.** `while v.contains(&x) { let i = v.iter().position(...); v.remove(i); }` shifts the
vector again and again. Use `v.retain(|item| *item != x)`.

## Try it yourself

1. Type `testutil.rs` and `dyn_array.rs` yourself and run `cargo test dyn_array`.
2. Break `grow` on purpose (for example, start its loop at `1` instead of `0`), see which test fails and what it prints, and fix it.
3. Change the first capacity from 4 to 1 and run `examples/e03_amortized.rs`. Do the moves per push change?
4. Run the benchmark. Then change `insert_front` to insert at the **end** (`v.insert(v.len(), i)`). What happens to the numbers?
5. Commit: `git add code/p07-algorithms && git commit -m "feat(p07): DynArray, a growable array"`.

## Exercises

### Exercise 1 — What does it cost? (easy)

Give the time complexity for a `Vec` of `n` items: (a) read the middle item; (b) `push` when the vector is full;
(c) `remove(0)`; (d) `pop()`; (e) `contains`. (f) Which of these is `O(1)` only on average over many calls?

<details>
<summary>Hint</summary>

Think of the picture in section 1: which operations touch one slot, and which move many?

</details>

<details>
<summary>Answer</summary>

(a) `O(1)`. (b) `O(n)`: reallocation copies all items. (c) `O(n)`: shifts all other items. (d) `O(1)`. (e) `O(n)`.
(f) `push`: it is `O(1)` *amortized*, but the single push that triggers a reallocation costs `O(n)`.

</details>

### Exercise 2 — Swap remove (medium)

Add `pub fn swap_remove(&mut self, index: usize) -> T` to `DynArray`. It removes the item at `index` by moving the last
item into its place, in `O(1)`. Write a test that compares it with `Vec::swap_remove` on random operations.

<details>
<summary>Hint</summary>

Decrease `len` first. Then swap slot `index` with slot `len` (`self.slots.swap(a, b)` swaps two slots of a slice) and
`take` the item that is now in slot `len`. It also works when `index` is the last item (then it swaps a slot with itself).

</details>

<details>
<summary>Answer</summary>

Add inside an `impl<T> DynArray<T>` block:

```rust
    /// Removes the item at `index` by swapping it with the last item: O(1).
    /// The order of the items changes.
    pub fn swap_remove(&mut self, index: usize) -> T {
        assert!(index < self.len, "removal index is out of bounds");
        self.len -= 1;
        self.slots.swap(index, self.len);
        self.slots[self.len]
            .take()
            .expect("slot below len is filled")
    }
```

And the test:

```rust
    #[test]
    fn swap_remove_matches_vec() {
        let mut rng = Rng::new(7);
        let mut mine: DynArray<u64> = DynArray::new();
        let mut reference: Vec<u64> = Vec::new();
        for _ in 0..2000 {
            if reference.is_empty() || rng.below(3) > 0 {
                let x = rng.next_u64() % 50;
                mine.push(x);
                reference.push(x);
            } else {
                let i = rng.below(reference.len());
                assert_eq!(mine.swap_remove(i), reference.swap_remove(i));
            }
            assert_eq!(mine.len(), reference.len());
            for (i, expected) in reference.iter().enumerate() {
                assert_eq!(mine.get(i), Some(expected));
            }
        }
    }
```

Why it works: only two slots are touched, so the cost does not depend on `n`. The price is that the order of the remaining
items changes, and `Vec::swap_remove` does exactly the same, so the comparison holds.

</details>

### Exercise 3 — Remove all in one pass (hard)

Add `pub fn remove_all(&mut self, value: &T) where T: PartialEq` that removes every item equal to `value` in **one** pass,
`O(n)` (not one `remove` per item). Test it against `Vec::retain` on 300 random vectors with small values (so there are
many equal items). State the time and the extra space.

<details>
<summary>Hint</summary>

Use two positions: `read` visits every item, `write` is where the next kept item goes. Items to keep are moved from `read` to
`write` (when they differ), and `write` advances. At the end `len = write`. Remember to set the removed slots to `None`.

</details>

<details>
<summary>Answer</summary>

```rust
    /// Removes every item equal to `value` in one pass: O(n).
    pub fn remove_all(&mut self, value: &T)
    where
        T: PartialEq,
    {
        let mut write = 0;
        for read in 0..self.len {
            if self.slots[read].as_ref() == Some(value) {
                self.slots[read] = None;
            } else {
                if read != write {
                    self.slots[write] = self.slots[read].take();
                    self.moves += 1;
                }
                write += 1;
            }
        }
        self.len = write;
    }
```

```rust
    #[test]
    fn remove_all_matches_retain() {
        let mut rng = Rng::new(11);
        for _ in 0..300 {
            let mut mine: DynArray<u64> = DynArray::new();
            let mut reference: Vec<u64> = Vec::new();
            for _ in 0..rng.below(30) {
                let x = rng.next_u64() % 4;
                mine.push(x);
                reference.push(x);
            }
            let target = rng.next_u64() % 4;
            mine.remove_all(&target);
            reference.retain(|x| *x != target);
            assert_eq!(mine.len(), reference.len());
            for (i, expected) in reference.iter().enumerate() {
                assert_eq!(mine.get(i), Some(expected));
            }
        }
    }
```

Time `O(n)`: one pass, each item moves at most once. Extra space `O(1)`: only the two counters. This "read and write
index" pattern is the same idea as `Vec::retain`, and you will meet it again as *two pointers* in Lesson 07.15.

</details>

## Quiz

1. Why is `v[i]` `O(1)` for every `i`?

<details>
<summary>Answer</summary>

The items are contiguous and have the same size, so the address of item `i` is `start + i * size`: one multiplication and one
addition, whatever `i` is.

</details>

2. What three values does a `Vec<T>` keep on the stack, and where are the items?

<details>
<summary>Answer</summary>

A pointer to the block, the length and the capacity. The items are in one contiguous block on the heap.

</details>

3. Why does doubling the capacity make `push` amortized `O(1)`, and why would growing by 10 slots at a time not?

<details>
<summary>Answer</summary>

With doubling the reallocations happen at sizes 4, 8, 16, ... and the copies add up to less than `2n`, so the average per push is a
constant. With a fixed step of 10, reallocation happens every 10 pushes and copies about `n` items each time, so the total grows
like `n²/20`: the average per push grows with `n`.

</details>

4. You must remove the first item of a huge vector again and again. Which type should you use?

<details>
<summary>Answer</summary>

`VecDeque`, where removing from the front is `O(1)` (Lesson 07.5). `Vec::remove(0)` is `O(n)`.

</details>

## Summary

- An array stores same-size items contiguously, so `address = start + i * size` and `v[i]` is `O(1)`.
- A `Vec` is a pointer, a length and a capacity on the stack, plus one contiguous block on the heap.
- `push` and `pop` at the end are cheap (`push` is amortized `O(1)` because capacity doubles); `insert` and `remove` in the middle or at
  the front shift items and cost `O(n)`.
- `DynArray` shows the mechanism in safe Rust: a box of slots, a length, a grow step that doubles, and a counter of moves.
- Test a data structure against the standard one on thousands of random operations; use a seeded pseudo-random generator so a failure can
  be repeated.
- In real code use `Vec` (`with_capacity`, `swap_remove`, `retain`) or `VecDeque` for the front.

## Sources and further reading

- [std::vec::Vec — Capacity and reallocation](https://doc.rust-lang.org/std/vec/struct.Vec.html#capacity-and-reallocation) — the official description of length, capacity and growth.
- [The Rustonomicon — Implementing Vec](https://doc.rust-lang.org/nomicon/vec/vec.html) — how the real `Vec` is built with `unsafe` code (advanced; read it after Phase 06 feels easy).
- [Vec::swap_remove](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.swap_remove) and [Vec::retain](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.retain) — the standard-library versions of Exercises 2 and 3.
- [Option::take](https://doc.rust-lang.org/std/option/enum.Option.html#method.take) — moving a value out of an `Option` you only borrow.
- [std::mem::size_of](https://doc.rust-lang.org/std/mem/fn.size_of.html) — the size of a type in bytes.

---

[🏠 Start](../START-HERE.md) · [← 07.2 Analyzing time and space](02-analyzing-time-and-space.md) · [📋 Phase 07](README.md) · [07.4 Linked lists →](04-linked-lists.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
