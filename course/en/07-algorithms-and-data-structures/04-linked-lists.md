<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.3 Arrays and vectors under the hood](03-arrays-and-vectors-under-the-hood.md) · [📋 Phase 07](README.md) · [07.5 Stacks and queues →](05-stacks-and-queues.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/04-linked-lists.md) <!-- nav -->

# Lesson 07.4 — Linked lists

> **Phase 07 — Algorithms and data structures** · Lesson 4 of 20 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a linked list is: nodes that point to the next node, instead of items in one block.
- How to build a singly linked list in safe Rust with `Option<Box<Node<T>>>`, including `push_front`, `pop_front`, `push_back`, `reverse` and an iterator.
- Why dropping a very long list needs a hand-written `Drop`, and what a stack overflow looks like.
- Why a linked list is almost always slower than a `Vec`, and the few cases where it is the right tool.

## Before you start

- You finished [Lesson 07.3 — Arrays and vectors under the hood](03-arrays-and-vectors-under-the-hood.md).
- You know `Box` and recursive types ([Lesson 05.9](../05-rust-intermediate/09-smart-pointers.md), section 3), `Option` and its methods `take`, `map`, `as_ref`, `as_deref` ([Lesson 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)), and how to write an iterator ([Lesson 05.5](../05-rust-intermediate/05-iterators.md), section 8).

## New words

| Word | Meaning |
|---|---|
| **linked list** | A chain of **nodes**. Each node holds a value and a link to the next node. |
| **node** | One element of the chain: a value plus a link. Each node is a separate heap allocation. |
| **head** | The first node. The list keeps only this one. The last node's link is empty (`None`). |
| **singly linked** | Each node knows only the next node, not the previous one. (A *doubly linked* list knows both.) |
| **cursor** | A variable that points at a place in a data structure and moves along it step by step. |
| **pointer chasing** | Following a link to a node that may be anywhere in memory. It is slow because the processor cannot predict where to look next. |
| **cache** | A small, very fast memory inside the processor that keeps recently used data and its neighbours. |
| **niche optimization** | Rust's trick of storing `None` as a value that a type can never have (for `Box`, the address 0), so `Option<Box<T>>` needs no extra space. |
| **stack overflow** | A crash when the stack runs out of room (Lesson 03.1). |

## 1. The idea

An array keeps its items in one block, side by side. A **linked list** does the opposite: every item lives in its own
small piece of heap memory, a **node**, and the node holds the address of the next one.

```text
head
 │
 ▼
┌────────┬───┐   ┌────────┬───┐   ┌────────┬──────┐
│ value 7│ ●─┼──▶│ value 3│ ●─┼──▶│ value 9│ None │
└────────┴───┘   └────────┴───┘   └────────┴──────┘
   node A           node B           node C          (three separate heap allocations)
```

The list stores only `head`. To reach the third item you must start at the head and follow two links: there is no
`address = start + i * size` shortcut. What you get in return is cheap changes at the front: to add a node at the front you
create it, point it at the old head and make it the new head. Nothing else moves.

| Operation | Singly linked list | `Vec` |
|---|---|---|
| read item `i` | `O(i)`: follow `i` links | `O(1)` |
| add or remove at the front | `O(1)` | `O(n)` |
| add or remove at the back | `O(n)`: walk to the end (`O(1)` with a pointer to the last node) | `O(1)` amortized |
| insert after a node you already hold | `O(1)` | `O(n)` |
| search for a value | `O(n)` | `O(n)` |

## 2. Nodes in Rust

A node contains the next node, so the type is recursive. Lesson 05.9 showed that a recursive type needs indirection, and
`Box` provides it. "A link is either empty or a boxed node" is exactly `Option<Box<Node<T>>>`. Create `src/linked_list.rs`, add `pub mod linked_list;` to
`src/lib.rs`, and start with the types:

```rust
//! Lesson 07.4: a singly linked list.

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    value: T,
    next: Link<T>,
}

/// A singly linked list. `head` is the first node (or `None` for an empty list).
pub struct List<T> {
    head: Link<T>,
    len: usize,
}

impl<T> List<T> {
    pub fn new() -> List<T> {
        List { head: None, len: 0 }
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }
```

**What happens:**

1. `type Link<T> = ...` gives a short name to the long type. A *type alias* creates no new type.
2. `Node` holds the value and the link to the next node. `Node` is private: users of the list never see nodes.
3. `List` holds the head and, as in Lesson 07.3, a stored `len` so that `len()` is `O(1)`.

Is the empty link free? `examples/e04_sizes.rs`:

```rust
use std::mem::size_of;

struct Node {
    _value: u64,
    _next: Option<Box<Node>>,
}

fn main() {
    println!("Box<Node>         {} bytes", size_of::<Box<Node>>());
    println!("Option<Box<Node>> {} bytes", size_of::<Option<Box<Node>>>());
    println!("Node              {} bytes", size_of::<Node>());
}
```

```text
Box<Node>         8 bytes
Option<Box<Node>> 8 bytes
Node              16 bytes
```

`Option<Box<Node>>` takes the same 8 bytes as `Box<Node>`: the **niche optimization**. A `Box` can never be address 0, so
Rust uses 0 to mean `None`. An "empty link" costs nothing extra.

## 3. Working at the front, and walking

Continue inside the same `impl<T> List<T>` block:

```rust
    /// Adds a value at the front: O(1).
    pub fn push_front(&mut self, value: T) {
        let old_head = self.head.take();
        self.head = Some(Box::new(Node {
            value,
            next: old_head,
        }));
        self.len += 1;
    }

    /// Removes and returns the first value: O(1).
    pub fn pop_front(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            self.len -= 1;
            node.value
        })
    }

    /// Looks at the first value without removing it: O(1).
    pub fn peek_front(&self) -> Option<&T> {
        self.head.as_ref().map(|node| &node.value)
    }

    /// Adds a value at the back. We must walk to the end first: O(n).
    pub fn push_back(&mut self, value: T) {
        let mut cursor = &mut self.head;
        while let Some(node) = cursor {
            cursor = &mut node.next;
        }
        *cursor = Some(Box::new(Node { value, next: None }));
        self.len += 1;
    }
```

**What happens:**

1. `push_front`: `self.head.take()` takes the old head out and leaves `None` in its place (`Option::take`, Lesson 07.3).
   Then a new node is created that points to the old head, and it becomes the head. Four steps, whatever `n` is: `O(1)`.
2. `pop_front`: `self.head.take()` removes the first node, if there is one. `.map(|node| ...)` runs only for `Some`: the head
   becomes `node.next` (the second node, moved out of the box), the length decreases and the value is returned.
3. `peek_front` returns a reference to the first value. `as_ref()` turns `&Option<Box<Node>>` into `Option<&Box<Node>>`, so
   nothing is moved out.
4. `push_back` has no shortcut. `cursor` is a `&mut Link<T>`: a mutable reference to *the place that holds a link*. The
   `while let` loop moves the cursor from link to link until it points at the empty link at the end. Writing into that place
   (`*cursor = Some(...)`) attaches the new node. This takes `O(n)`.

```text
push_front(1) on  head -> [2] -> [3]

1. take the old head     old_head = [2] -> [3]       head = None
2. new node points to it            [1] -> [2] -> [3]
3. new node is the head       head -> [1] -> [2] -> [3]
```

Next: reversing the list in place, and an iterator.

```rust
    /// Reverses the list in place by turning every `next` link around: O(n).
    pub fn reverse(&mut self) {
        let mut reversed: Link<T> = None;
        let mut rest = self.head.take();
        while let Some(mut node) = rest {
            rest = node.next.take();
            node.next = reversed;
            reversed = Some(node);
        }
        self.head = reversed;
    }

    /// Walks over the values from the first to the last.
    pub fn iter(&self) -> Iter<'_, T> {
        Iter {
            next: self.head.as_deref(),
        }
    }
}

pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<&'a T> {
        self.next.map(|node| {
            self.next = node.next.as_deref();
            &node.value
        })
    }
}
```

**What happens:**

1. `reverse` walks the list once and turns every link around. `rest` is the part not yet processed, `reversed` is the part
   already turned. In each round we cut the first node off `rest`, point it at `reversed`, and make it the new start of
   `reversed`. No node is allocated, none is copied: `O(n)` time, `O(1)` extra space.
2. `Iter` holds `next: Option<&Node<T>>`, a reference to the node it will show next. Its `next` method returns a reference to
   that node's value and moves on to the following node with `as_deref()`, which turns `&Option<Box<Node>>` into
   `Option<&Node>`. The lifetime `'a` says: the references handed out live as long as the borrow of the list.

With the iterator, every standard tool works on our list: `for`, `map`, `sum`, `collect`. `examples/e04_list.rs`:

```rust
use p07_algorithms::linked_list::List;

fn main() {
    let mut list = List::new();
    list.push_front(2);
    list.push_front(1);
    list.push_back(3);
    list.push_back(4);
    println!("{:?}", list.iter().collect::<Vec<_>>());

    list.reverse();
    println!("{:?}", list.iter().collect::<Vec<_>>());

    println!("first = {:?}", list.pop_front());
    println!("peek  = {:?}", list.peek_front());
    println!("length = {}", list.len());
}
```

```text
[1, 2, 3, 4]
[4, 3, 2, 1]
first = Some(4)
peek  = Some(3)
length = 3
```

(`list.iter().collect::<Vec<_>>()` collects the references into a vector for printing.)

## 4. Dropping a long list

Now a surprise. When a value goes out of scope, Rust *drops* it (Lesson 05.9: `Drop`): it drops the fields, one after another.
Dropping a `Node` drops its `next`, which is a `Box<Node>`, which drops *its* `next`... That is **recursion**, with one stack frame
(Lesson 03.1) per node. A list of a million nodes needs a million frames. `examples/e04_overflow.rs` builds such a list **without**
a custom `Drop`:

```rust
struct Node {
    value: u64,
    next: Option<Box<Node>>,
}

fn main() {
    let mut head: Option<Box<Node>> = None;
    for i in 0..1_000_000 {
        head = Some(Box::new(Node {
            value: i,
            next: head,
        }));
    }
    if let Some(node) = &head {
        println!(
            "built 1000000 nodes; first value {}, has next: {}",
            node.value,
            node.next.is_some()
        );
    }
    // `head` is dropped here
}
```

```text
built 1000000 nodes; first value 999999, has next: true

thread 'main' (4788096) has overflowed its stack
fatal runtime error: stack overflow, aborting
```

The list was built successfully, and the crash came at the closing brace of `main`, when it was dropped: the stack overflowed (the stack
is about 8 MiB for the main thread, and fewer for test threads).

> 🔒 **Security:** A program that builds a long list from external input and then crashes while dropping it can be crashed on purpose
> by an attacker who sends a long input. A stack overflow is not a panic you can catch: the process is aborted.

The cure is to drop the nodes **in a loop**, not by recursion. This is the last part of `src/linked_list.rs` (before the tests):

```rust
impl<T> Default for List<T> {
    fn default() -> List<T> {
        List::new()
    }
}

/// Without this, dropping a very long list would use one stack frame per node.
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut rest = self.head.take();
        while let Some(mut node) = rest {
            rest = node.next.take();
        }
    }
}
```

**What happens:** `Drop::drop` is called when the list dies. It takes the head and walks the chain; in each round `node.next.take()`
detaches the next node, so when the loop variable `node` is dropped at the end of the round it owns **no** successor and
drops only itself: no recursion. (`Default` is there because Clippy asks for it when a type has a `new()` without arguments.)

## 5. Tests

Two tests, as in Lesson 07.3: a random comparison with `Vec` as the reference, and a test for the stack-overflow fix. At the bottom of
`src/linked_list.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;

    fn contents(list: &List<u64>) -> Vec<u64> {
        list.iter().copied().collect()
    }

    #[test]
    fn behaves_like_vec_on_random_operations() {
        let mut rng = Rng::new(99);
        let mut list: List<u64> = List::new();
        let mut model: Vec<u64> = Vec::new();
        for step in 0..3000 {
            match rng.below(5) {
                0 => {
                    let x = rng.next_u64() % 100;
                    list.push_front(x);
                    model.insert(0, x);
                }
                1 => {
                    let x = rng.next_u64() % 100;
                    list.push_back(x);
                    model.push(x);
                }
                2 | 3 => {
                    let expected = if model.is_empty() {
                        None
                    } else {
                        Some(model.remove(0))
                    };
                    assert_eq!(list.pop_front(), expected);
                }
                _ => {
                    list.reverse();
                    model.reverse();
                }
            }
            assert_eq!(list.len(), model.len(), "step {step}");
            assert_eq!(contents(&list), model, "step {step}");
        }
    }

    #[test]
    fn dropping_a_very_long_list_does_not_overflow_the_stack() {
        let mut list = List::new();
        for i in 0..1_000_000u64 {
            list.push_front(i);
        }
        assert_eq!(list.len(), 1_000_000);
        // `list` is dropped here
    }
}
```

**What happens:**

1. The random test does 3,000 random steps: `push_front` (the model does `insert(0, ..)`), `push_back` (`push`), `pop_front` (twice as
   often, `remove(0)`) and `reverse`. After every step the whole content of the list must equal the model.
2. The second test builds a list of a million nodes and lets it drop. Remove the `Drop` implementation and run it: the test process aborts
   with a stack overflow instead of passing.

```text
running 2 tests
test linked_list::tests::behaves_like_vec_on_random_operations ... ok
test linked_list::tests::dropping_a_very_long_list_does_not_overflow_the_stack ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 12 filtered out; finished in 0.03s
```

## 6. Measuring: linked list against vector

`benches/linked_list.rs` compares our list, `std::collections::LinkedList` (the standard doubly linked list) and `Vec`. Register it
in `Cargo.toml` as in Lesson 07.2:

```toml
[[bench]]
name = "linked_list"
harness = false
```

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use p07_algorithms::linked_list::List;
use std::collections::LinkedList;
use std::hint::black_box;

const N: u64 = 20_000;

fn bench_sum(c: &mut Criterion) {
    let vec: Vec<u64> = (0..N).collect();
    let mut list = List::new();
    let mut std_list = LinkedList::new();
    for i in 0..N {
        list.push_front(i);
        std_list.push_front(i);
    }

    let mut group = c.benchmark_group("sum 20_000 items");
    group.bench_function("Vec", |b| b.iter(|| black_box(&vec).iter().sum::<u64>()));
    group.bench_function("our List", |b| {
        b.iter(|| black_box(&list).iter().sum::<u64>())
    });
    group.bench_function("std LinkedList", |b| {
        b.iter(|| black_box(&std_list).iter().sum::<u64>())
    });
    group.finish();
}

fn bench_push_front(c: &mut Criterion) {
    let mut group = c.benchmark_group("push 20_000 items at the front");
    group.bench_function("Vec insert(0, ..)", |b| {
        b.iter(|| {
            let mut v = Vec::new();
            for i in 0..black_box(N) {
                v.insert(0, i);
            }
            v
        })
    });
    group.bench_function("our List", |b| {
        b.iter(|| {
            let mut list = List::new();
            for i in 0..black_box(N) {
                list.push_front(i);
            }
            list
        })
    });
    group.finish();
}

criterion_group!(benches, bench_sum, bench_push_front);
criterion_main!(benches);
```

Run `cargo bench --bench linked_list -- --warm-up-time 1 --measurement-time 3`:

```text
sum 20_000 items/Vec    time:   [1.3024 µs 1.3063 µs 1.3109 µs]
sum 20_000 items/our List
                        time:   [22.193 µs 22.245 µs 22.301 µs]
sum 20_000 items/std LinkedList
                        time:   [23.972 µs 24.045 µs 24.133 µs]
push 20_000 items at the front/Vec insert(0, ..)
                        time:   [17.680 ms 17.769 ms 17.904 ms]
push 20_000 items at the front/our List
                        time:   [276.65 µs 277.55 µs 278.51 µs]
```

(Progress and outlier lines are left out. Apple M4 Pro; **your numbers will differ**.)

**How to read it:**

1. **Summing 20,000 numbers**: `Vec` takes 1.3 µs, our list 22 µs, `std::LinkedList` 24 µs. The list is about **17 times slower**,
   although all three are `O(n)`. Reading the vector goes through memory in order, and the processor loads the neighbours into its
   cache before they are needed. A list means pointer chasing: each node must be fetched before the processor even knows where the next
   one is. (Here the nodes were allocated one after another, so they sit close together. In a real program that adds and removes nodes
   for hours, they are scattered, and the gap is larger.)
2. **Pushing 20,000 numbers at the front**: `Vec::insert(0, ..)` takes 17.8 ms, our list 0.28 ms. This is the case the list was
   made for: `O(1)` against `O(n)`, so the list is **64 times faster**. The `VecDeque` of Lesson 07.5 is `O(1)` here too, and keeps the
   cache advantage.

> ⚡ **Performance:** "Linked list" and "fast" rarely go together on a modern processor. Big-O counts steps; it does not see the cache.
> Reach for `Vec` or `VecDeque` first.

## 7. When to use a linked list

- **Almost always: do not.** `Vec` (or `VecDeque` for both ends) is simpler and faster for stacks, queues and sequences.
- **Use it when** you need `O(1)` insertion or removal in the *middle* and you already hold the position (an editor's
  text buffer, an LRU (Least Recently Used) cache that throws away the item unused for the longest time), or when you want to join two lists in `O(1)`. For these, use `std::collections::LinkedList`.
- **Learn it anyway.** Linked nodes are the building block of trees (Lesson 07.11) and graphs (Lesson 07.13), and the technique of
  `take()` and `&mut` cursors appears in all of them.

A doubly linked list (each node also knows the previous one) needs shared ownership between neighbours: `Rc`, `RefCell` and `Weak`
(Lessons 05.9 and 06.4), or `unsafe`. That is why `std::collections::LinkedList` is written with `unsafe` inside, and why we stop at the
singly linked one.

## Common mistakes

**1. Forgetting `take()`.** Writing `self.head = Some(Box::new(Node { value, next: self.head }))` in `push_front` does not compile:

```text
error[E0507]: cannot move out of `self.head` which is behind a mutable reference
  --> src/main.rs:14:19
   |
14 |             next: self.head,
   |                   ^^^^^^^^^ move occurs because `self.head` has type `Option<Box<Node>>`, which does not implement the `Copy` trait
```

`self.head` is borrowed through `&mut self`, so you cannot move the value out and leave a hole. `take()` moves the value out *and*
leaves `None` there, so the list is valid at every moment.

**2. Recursion over a long list.** Any function that calls itself once per node (a recursive `len`, a recursive print) overflows the stack on a
long list, exactly like the default drop. Write loops, or use the iterator.

## Try it yourself

1. Type `linked_list.rs` yourself and run `cargo test linked_list`, `cargo clippy` and `cargo fmt`.
2. Remove the `Drop` implementation, run the tests and read the failure. Put it back.
3. Run `examples/e04_sizes.rs`, then change `next: Option<Box<Node>>` to `next: Box<Node>` in a copy (it will not compile). Why?
4. Run the benchmark and write down the "pointer chasing" ratio on your computer.
5. Commit: `git add code/p07-algorithms && git commit -m "feat(p07): singly linked list"`.

## Exercises

### Exercise 1 — What does it cost? (easy)

For our singly linked list with only a `head` pointer, give the time complexity of: (a) `push_front`; (b) `push_back`; (c) removing the *last*
node; (d) `len()`; (e) finding the value 42; (f) `reverse`. Which of these would change if the list also kept a pointer to the last node?

<details>
<summary>Hint</summary>

Count the links that have to be followed. For (d) look at the struct.

</details>

<details>
<summary>Answer</summary>

(a) `O(1)`. (b) `O(n)`: walk to the end. (c) `O(n)`: we must find the second-to-last node to cut the link. (d) `O(1)`: the stored `len`.
(e) `O(n)`. (f) `O(n)`. With a pointer to the last node, (b) would become `O(1)`; (c) would stay `O(n)`, because even then we cannot
reach the second-to-last node without walking (a doubly linked list solves that).

</details>

### Exercise 2 — `contains` and `pop_back` (medium)

Add `contains(&self, target: &T) -> bool where T: PartialEq` (use the iterator) and `pop_back(&mut self) -> Option<T>` (use a `&mut Link`
cursor, stop at the last node). Write a test that compares both with `Vec`. State the time of both.

<details>
<summary>Hint</summary>

For `pop_back`, keep moving the cursor while the node it points at has a successor: `cursor.as_ref()?.next.is_some()`. (The `?` returns `None` at
once if the list is empty.) When the loop stops, `cursor` points at the link of the last node, and `cursor.take()` removes it.

</details>

<details>
<summary>Answer</summary>

Add both methods inside an `impl<T> List<T>` block:

```rust
    /// Is `target` in the list? O(n).
    pub fn contains(&self, target: &T) -> bool
    where
        T: PartialEq,
    {
        self.iter().any(|value| value == target)
    }

    /// Removes and returns the last value: O(n), because we must find the second-to-last node.
    pub fn pop_back(&mut self) -> Option<T> {
        let mut cursor = &mut self.head;
        // walk until `cursor` points at the link of the last node
        while cursor.as_ref()?.next.is_some() {
            cursor = &mut cursor.as_mut()?.next;
        }
        let last = cursor.take()?;
        self.len -= 1;
        Some(last.value)
    }
```

A test (the same style as in section 5; it also checks `nth_from_end` of the next exercise):

```rust
    #[test]
    fn contains_pop_back_and_nth_from_end_match_vec() {
        let mut rng = Rng::new(5);
        for _ in 0..200 {
            let mut list: List<u64> = List::new();
            let mut model: Vec<u64> = Vec::new();
            for _ in 0..rng.below(8) {
                let x = rng.next_u64() % 5;
                list.push_back(x);
                model.push(x);
            }
            assert_eq!(list.contains(&3), model.contains(&3));
            for k in 0..10 {
                let expected = model.len().checked_sub(k + 1).map(|i| &model[i]);
                assert_eq!(list.nth_from_end(k), expected, "k = {k}");
            }
            assert_eq!(list.pop_back(), model.pop());
            assert_eq!(list.len(), model.len());
            assert_eq!(contents(&list), model);
        }
    }
```

Both are `O(n)` time and `O(1)` extra space.

</details>

### Exercise 3 — The k-th from the end in one pass (hard)

Add `nth_from_end(&self, k: usize) -> Option<&T>` that returns the value `k` places before the last one (`k = 0` is the last value), walking the list **once** and
without computing the length first. State the time and the space.

<details>
<summary>Hint</summary>

Use two iterators. Move the first one `k` steps ahead. Then advance both together until the first one reaches the end: the second one is
`k` places behind it.

</details>

<details>
<summary>Answer</summary>

Inside an `impl<T> List<T>` block:

```rust
    /// The value `k` places before the last one (`k = 0` is the last value), in one pass: O(n).
    pub fn nth_from_end(&self, k: usize) -> Option<&T> {
        let mut lead = self.iter();
        for _ in 0..k {
            lead.next()?; // the list is too short
        }
        let mut lag = self.iter();
        let mut found = None;
        while lead.next().is_some() {
            found = lag.next();
        }
        found
    }
```

This is the *two pointers* technique (Lesson 07.15). Time `O(n)`, one pass; space `O(1)`. If the list is shorter than `k + 1` items, the first loop returns `None`.

</details>

## Quiz

1. Why is reading item `i` of a linked list `O(i)` and not `O(1)`?

<details>
<summary>Answer</summary>

The nodes are separate allocations; the only way to find node `i` is to follow `i` links from the head. There is no address formula.

</details>

2. What does `Option::take` do, and why do `push_front` and `pop_front` need it?

<details>
<summary>Answer</summary>

It moves the value out of an `Option` and leaves `None` behind. Without it we could not move the old head out of `self.head` (borrowed through `&mut self`) without
leaving the list in an invalid state.

</details>

3. Why can dropping a list of a million nodes crash the program, and how does the custom `Drop` prevent it?

<details>
<summary>Answer</summary>

The default drop is recursive: each node drops its successor, so the call depth equals the list length and the stack overflows. The custom `Drop` detaches nodes one by one in
a loop, so each node is dropped with no successor and the depth stays constant.

</details>

4. A `Vec` and a linked list both sum their items in `O(n)`. Why was the list 17 times slower in our benchmark?

<details>
<summary>Answer</summary>

Big-O ignores memory behaviour. The vector is read in order, so the processor prefetches the data into its cache. The list needs one pointer jump per item, and each jump may
miss the cache.

</details>

## Summary

- A linked list is a chain of separately allocated nodes; the list keeps only the head. In safe Rust: `Option<Box<Node<T>>>`.
- `push_front` and `pop_front` are `O(1)`; reading by position, `push_back` (without a tail pointer) and removing the last node are `O(n)`.
- `Option::take` and `&mut` cursors are the tools for changing links in safe Rust.
- Dropping a long chain recursively overflows the stack; a hand-written `Drop` with a loop fixes it.
- Test the list against `Vec` on random operations, and keep a test with a million nodes.
- On real hardware a linked list is usually much slower than a `Vec`. Use `Vec` or `VecDeque`, and use `std::collections::LinkedList` only for its few special strengths.

## Sources and further reading

- [Learn Rust With Entirely Too Many Linked Lists](https://rust-unofficial.github.io/too-many-lists/) — a free book that builds many kinds of linked lists in Rust, from this lesson's design to doubly linked and unsafe versions.
- [std::collections::LinkedList](https://doc.rust-lang.org/std/collections/struct.LinkedList.html) — the standard doubly linked list, and its own advice on when to use it.
- [Option::take](https://doc.rust-lang.org/std/option/enum.Option.html#method.take) and [Option::as_deref](https://doc.rust-lang.org/std/option/enum.Option.html#method.as_deref) — the two `Option` methods that carry this lesson.
- [std::ops::Drop](https://doc.rust-lang.org/std/ops/trait.Drop.html) — how and when `drop` runs.
- [std::option — Representation](https://doc.rust-lang.org/std/option/index.html#representation) — which `Option<T>` types take no extra space (the niche optimization).

---

[🏠 Start](../START-HERE.md) · [← 07.3 Arrays and vectors under the hood](03-arrays-and-vectors-under-the-hood.md) · [📋 Phase 07](README.md) · [07.5 Stacks and queues →](05-stacks-and-queues.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
