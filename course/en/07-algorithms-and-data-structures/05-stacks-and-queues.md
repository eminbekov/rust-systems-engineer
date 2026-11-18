<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 07.4 Linked lists](04-linked-lists.md) · [📋 Phase 07](README.md) · [07.6 How hash tables work →](06-hash-tables.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/07-algorithms-and-data-structures/05-stacks-and-queues.md) <!-- nav -->

# Lesson 07.5 — Stacks and queues

> **Phase 07 — Algorithms and data structures** · Lesson 5 of 20 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a stack (last in, first out) and a queue (first in, first out) are, and what each is used for.
- How to build a stack on a `Vec` and use it to check that brackets are balanced.
- Why a queue built on `Vec::remove(0)` is slow, and how a ring buffer makes both ends `O(1)`.
- How `VecDeque` works, and when to choose it.

## Before you start

- You finished [Lesson 07.4 — Linked lists](04-linked-lists.md).
- You know `Vec` ([Lesson 07.3](03-arrays-and-vectors-under-the-hood.md)), `match` ([Lesson 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)) and the idea of amortized cost ([Lesson 07.2](02-analyzing-time-and-space.md)).

## New words

| Word | Meaning |
|---|---|
| **stack** | A collection where you add and remove at the same end. Like a pile of plates: you take the plate you put last. |
| **LIFO** (Last In, First Out) | The rule of a stack: the item added last is removed first. |
| **queue** | A collection where you add at one end (the *back*) and remove at the other (the *front*). Like people waiting in a line. |
| **FIFO** (First In, First Out) | The rule of a queue: the item added first is removed first. |
| **push / pop** | Add an item / remove an item (for a stack: at the top; for a queue the words are `push_back` and `pop_front`). |
| **peek** | Look at the next item to be removed without removing it. |
| **deque** (double-ended queue, pronounced "deck") | A queue where you can add and remove at both ends. |
| **ring buffer** (circular buffer) | A fixed block of slots used as a circle: after the last slot comes the first one again. |
| **encapsulation** | Hiding how data is kept and offering only the operations that keep it correct. |

## 1. Stacks

A **stack** follows the **LIFO** rule: the last item in is the first item out. Think of a pile of plates: you put a plate on top, and you take the
top plate. There are only three main operations, and all work at the **top**:

```text
push(1)   push(2)   push(3)   pop() -> 3   pop() -> 2
                     ┌───┐
           ┌───┐     │ 3 │ ◀── top
 ┌───┐     │ 2 │     │ 2 │      ┌───┐
 │ 1 │     │ 1 │     │ 1 │      │ 1 │         ┌───┐
 └───┘     └───┘     └───┘      └───┘         │ 1 │
                                              └───┘
```

A `Vec` is already a good stack: `push` and `pop` work at the end and cost `O(1)` (amortized for `push`). We wrap it in a small type, so that
the code says what it means, and so that nobody can accidentally insert in the middle. Create `src/stacks.rs` and add `pub mod stacks;` to `src/lib.rs`:

```rust
//! Lesson 07.5: stacks (last in, first out).

/// A stack. It is a `Vec` that only allows work at one end.
pub struct Stack<T> {
    items: Vec<T>,
}

impl<T> Stack<T> {
    pub fn new() -> Stack<T> {
        Stack { items: Vec::new() }
    }

    /// Puts a value on top: amortized O(1).
    pub fn push(&mut self, value: T) {
        self.items.push(value);
    }

    /// Takes the top value off: O(1).
    pub fn pop(&mut self) -> Option<T> {
        self.items.pop()
    }

    /// Looks at the top value: O(1).
    pub fn peek(&self) -> Option<&T> {
        self.items.last()
    }

    pub fn len(&self) -> usize {
        self.items.len()
    }

    pub fn is_empty(&self) -> bool {
        self.items.is_empty()
    }
}

impl<T> Default for Stack<T> {
    fn default() -> Stack<T> {
        Stack::new()
    }
}
```

**What happens:** `Stack<T>` holds a private `Vec<T>`. `push`, `pop` and `peek` use the end of the vector (`last()` is the end), so each is `O(1)`.
The type offers no way to reach other positions: a user cannot break the LIFO rule. This is called **encapsulation**: you hide how the data is kept and offer only the
operations that keep it correct.

### Use 1: matching brackets

Is `f(a[1], {b: 2})` correct, with every bracket closed by the right kind, in the right order? Idea: go through the text; when you meet an *opening*
bracket, push it. When you meet a *closing* bracket, the most recent unclosed opening bracket must be its partner: pop it and compare. At the end the stack must be empty.

```rust
/// Are all brackets `()`, `[]`, `{}` closed in the right order? O(n) time, O(n) space.
pub fn is_balanced(text: &str) -> bool {
    let mut open = Stack::new();
    for ch in text.chars() {
        match ch {
            '(' | '[' | '{' => open.push(ch),
            ')' | ']' | '}' => {
                let expected = match ch {
                    ')' => '(',
                    ']' => '[',
                    _ => '{',
                };
                if open.pop() != Some(expected) {
                    return false;
                }
            }
            _ => {} // other characters are ignored
        }
    }
    open.is_empty()
}
```

**What happens:**

1. Opening brackets are pushed. Other characters are ignored.
2. For a closing bracket we work out which opening bracket it needs (`expected`). `open.pop()` gives the most recent unclosed one (or `None` if
   there is none). If it is not equal to `Some(expected)`, the text is wrong: `"a)b"` (nothing to close), `"([)]"` (wrong order).
3. At the end, `open.is_empty()`: a leftover bracket such as in `"(()"` means something was never closed.

```text
text:  ( [ ] { } )
step 1  (        stack: (
step 2  [        stack: ( [
step 3  ]  pop [ stack: (
step 4  {        stack: ( {
step 5  }  pop { stack: (
step 6  )  pop ( stack: (empty)   -> balanced
```

Time `O(n)` (every character is handled once, each stack operation is `O(1)`); space `O(n)` in the worst case (a text of only opening brackets).

`examples/e05_brackets.rs`:

```rust
use p07_algorithms::stacks::is_balanced;

fn main() {
    for text in ["f(a[1], {b: 2})", "f(a[1}, 2)", "((", "a)b"] {
        println!("{text:>16}  balanced: {}", is_balanced(text));
    }
}
```

```text
 f(a[1], {b: 2})  balanced: true
      f(a[1}, 2)  balanced: false
              ((  balanced: false
             a)b  balanced: false
```

Add the tests at the bottom of `src/stacks.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn stack_is_last_in_first_out() {
        let mut s = Stack::new();
        s.push(1);
        s.push(2);
        s.push(3);
        assert_eq!(s.peek(), Some(&3));
        assert_eq!(s.pop(), Some(3));
        assert_eq!(s.pop(), Some(2));
        assert_eq!(s.len(), 1);
        assert_eq!(s.pop(), Some(1));
        assert_eq!(s.pop(), None);
        assert!(s.is_empty());
    }

    #[test]
    fn brackets() {
        for good in ["", "()", "([]{})", "a(b[c]{d})e", "{[()()]}"] {
            assert!(is_balanced(good), "{good:?} should be balanced");
        }
        for bad in ["(", ")", ")(", "([)]", "(()", "{[}]", "a)b"] {
            assert!(!is_balanced(bad), "{bad:?} should not be balanced");
        }
    }
}
```

```text
running 2 tests
test stacks::tests::stack_is_last_in_first_out ... ok
test stacks::tests::brackets ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 15 filtered out; finished in 0.00s
```

> 🔒 **Security:** `is_balanced` reads text that may come from outside. It never panics (it uses `pop()` and compares `Option`s, no `unwrap`) and it handles every input in one pass, so hostile text cannot make it
> crash or run for long. Keep that habit: handle `None`, test with broken input.

Where else do stacks appear? The **call stack** of your program (Lesson 03.1) is a stack of function calls, and recursion (Lesson 07.7) uses it. "Undo" in an editor is a stack of changes. A compiler uses a stack to
check the nesting of brackets, exactly as here.

## 2. Queues

A **queue** follows **FIFO**: the first in is the first out, like a line at a shop. Items join at the **back** and leave from the **front**.

```text
 front                    back
   ▼                       ▼
  ┌───┬───┬───┬───┬───┐
  │ 3 │ 4 │ 5 │ 6 │ 7 │   pop_front() -> 3        push_back(8) joins after 7
  └───┴───┴───┴───┴───┘
```

Queues hold work that must be done in order of arrival: print jobs, network packets, and the "to visit next" list of a breadth-first search (Lesson 07.13).

The obvious implementation, a `Vec` with `push` for joining and `remove(0)` for leaving, is `O(n)` per leaving: all other items shift (Lesson 07.3). A linked list with a tail pointer would work,
but it is slow in practice (Lesson 07.4). The good solution is a **ring buffer**.

### The ring buffer

Keep a block of slots and two numbers: `head`, the index of the front item, and `len`, the number of items. The items occupy `len` slots starting at `head`; when they reach the end of the block,
they **wrap around** to slot 0. Leaving moves `head` forward one slot: **nothing is shifted**.

```text
capacity 4, head = 2, len = 3:       slot:   0     1     2     3
                                           [ 5 ] [   ] [ 3 ] [ 4 ]
                                             ▲           ▲
                                      (wrapped: 5 is   head: the front is 3
                                       the last item)
index of the back slot = (head + len) % capacity = (2 + 3) % 4 = 1   <- the next push goes here
```

The remainder `%` makes the index wrap around: `(2 + 3) % 4 = 1`. When the block is full, we allocate a block of twice the size and copy the items **in order** to its start (head becomes 0), as in `DynArray`.
Create `src/queues.rs` and add `pub mod queues;` to `src/lib.rs`:

```rust
//! Lesson 07.5: a queue (first in, first out) on a ring buffer.

/// A queue stored in a ring: `head` is the index of the front item, and the
/// items continue after it, wrapping around to index 0 at the end of `slots`.
pub struct RingQueue<T> {
    slots: Vec<Option<T>>,
    head: usize,
    len: usize,
}

impl<T> RingQueue<T> {
    pub fn new() -> RingQueue<T> {
        RingQueue {
            slots: Vec::new(),
            head: 0,
            len: 0,
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

    /// Adds a value at the back: amortized O(1).
    pub fn push_back(&mut self, value: T) {
        if self.len == self.slots.len() {
            self.grow();
        }
        let tail = (self.head + self.len) % self.slots.len();
        self.slots[tail] = Some(value);
        self.len += 1;
    }

    /// Removes the value at the front: O(1). Nothing is shifted.
    pub fn pop_front(&mut self) -> Option<T> {
        if self.len == 0 {
            return None;
        }
        let value = self.slots[self.head].take();
        self.head = (self.head + 1) % self.slots.len();
        self.len -= 1;
        value
    }

    /// Looks at the front value: O(1).
    pub fn front(&self) -> Option<&T> {
        if self.len == 0 {
            None
        } else {
            self.slots[self.head].as_ref()
        }
    }
```

```rust
    /// Doubles the capacity and lays the items out again from index 0: O(n).
    fn grow(&mut self) {
        let new_capacity = if self.slots.is_empty() {
            4
        } else {
            self.slots.len() * 2
        };
        let mut bigger: Vec<Option<T>> = Vec::new();
        bigger.resize_with(new_capacity, || None);
        for (i, slot) in bigger.iter_mut().enumerate().take(self.len) {
            let from = (self.head + i) % self.slots.len();
            *slot = self.slots[from].take();
        }
        self.slots = bigger;
        self.head = 0;
    }
}

impl<T> Default for RingQueue<T> {
    fn default() -> RingQueue<T> {
        RingQueue::new()
    }
}
```

**What happens:**

1. `push_back` grows the ring if it is full, computes the slot after the last item with `(head + len) % capacity`, and writes the value there.
2. `pop_front` takes the value out of the `head` slot, moves `head` forward by one (wrapping with `%`), and decreases `len`. It is `O(1)` and moves no other item.
3. `grow` doubles the capacity and copies the items in queue order into the new block. Because of the doubling, `push_back` is amortized `O(1)`, the same argument as in Lesson 07.3.
   (The method `take(self.len)` on the iterator limits the loop to the `len` real items.)

`examples/e05_queue.rs` follows the picture above (4 pushes, 2 pops, 2 pushes that wrap around, 1 push that grows):

```rust
use p07_algorithms::queues::RingQueue;

fn main() {
    let mut q = RingQueue::new();
    for i in 1..=4 {
        q.push_back(i);
    }
    println!("after 4 pushes: len {}, capacity {}", q.len(), q.capacity());

    println!("pop {:?}, pop {:?}", q.pop_front(), q.pop_front());
    q.push_back(5);
    q.push_back(6);
    println!(
        "after wrap-around: len {}, capacity {}",
        q.len(),
        q.capacity()
    );

    q.push_back(7);
    println!(
        "after one more push: len {}, capacity {}",
        q.len(),
        q.capacity()
    );

    let mut order = Vec::new();
    while let Some(x) = q.pop_front() {
        order.push(x);
    }
    println!("order out: {order:?}");
}
```

```text
after 4 pushes: len 4, capacity 4
pop Some(1), pop Some(2)
after wrap-around: len 4, capacity 4
after one more push: len 5, capacity 8
order out: [3, 4, 5, 6, 7]
```

The two extra pushes reused the freed slots, so the capacity stayed 4. Only the next push, which found the ring full, grew it to 8. The items still came out in the order they went in.

The test compares the queue with the standard `VecDeque` on 10,000 random pushes and pops:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use crate::testutil::Rng;
    use std::collections::VecDeque;

    #[test]
    fn behaves_like_vec_deque_on_random_operations() {
        let mut rng = Rng::new(31);
        let mut mine: RingQueue<u64> = RingQueue::new();
        let mut reference: VecDeque<u64> = VecDeque::new();
        for step in 0..10_000 {
            if rng.below(5) < 3 {
                let x = rng.next_u64() % 1000;
                mine.push_back(x);
                reference.push_back(x);
            } else {
                assert_eq!(mine.pop_front(), reference.pop_front(), "step {step}");
            }
            assert_eq!(mine.len(), reference.len(), "step {step}");
            assert_eq!(mine.front(), reference.front(), "step {step}");
        }
    }
}
```

```text
running 1 test
test queues::tests::behaves_like_vec_deque_on_random_operations ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 17 filtered out; finished in 0.00s
```

## 3. Measuring: why not `Vec::remove(0)`

`benches/queues.rs` fills a queue with 20,000 numbers and empties it completely, once with `Vec::remove(0)` and once with the standard `VecDeque`. Add to `Cargo.toml` (as in Lesson 07.2):

```toml
[[bench]]
name = "queues"
harness = false
```

```rust
use criterion::{Criterion, criterion_group, criterion_main};
use std::collections::VecDeque;
use std::hint::black_box;

const N: u64 = 20_000;

fn bench_drain(c: &mut Criterion) {
    let mut group = c.benchmark_group("fill and empty a queue of 20_000");
    group.bench_function("Vec, remove(0)", |b| {
        b.iter(|| {
            let mut q: Vec<u64> = (0..black_box(N)).collect();
            let mut sum = 0;
            while !q.is_empty() {
                sum += q.remove(0);
            }
            sum
        })
    });
    group.bench_function("VecDeque", |b| {
        b.iter(|| {
            let mut q: VecDeque<u64> = (0..black_box(N)).collect();
            let mut sum = 0;
            while let Some(x) = q.pop_front() {
                sum += x;
            }
            sum
        })
    });
    group.finish();
}

criterion_group!(benches, bench_drain);
criterion_main!(benches);
```

Run `cargo bench --bench queues -- --warm-up-time 1 --measurement-time 3`:

```text
fill and empty a queue of 20_000/Vec, remove(0)
                        time:   [17.874 ms 17.927 ms 17.982 ms]
fill and empty a queue of 20_000/VecDeque
                        time:   [23.609 µs 24.009 µs 24.578 µs]
```

(Apple M4 Pro; **your numbers will differ**.)

**How to read it:** `Vec::remove(0)` takes 17.9 ms: it is `O(n)` per removal and `O(n²)` in total. `VecDeque` takes 24 µs, **about 750 times faster** for the same job, because it is a ring buffer like ours: removing from the front moves a `head` index and shifts nothing.

## 4. When to use the standard library instead

| You need | Use | Notes |
|---|---|---|
| a stack | `Vec<T>`: `push`, `pop`, `last` | no wrapper needed in everyday code |
| a queue, or both ends | `std::collections::VecDeque<T>`: `push_back`, `pop_front`, `push_front`, `pop_back` | a ring buffer inside, like ours |
| "the smallest or largest item next" | `std::collections::BinaryHeap<T>` | a priority queue (Lesson 07.12) |
| threads that pass work to each other | channels (Lesson 05.10) | a queue between threads |

`VecDeque` also supports indexing (`q[3]`) in `O(1)` and iteration from front to back.

## Common mistakes

**1. Using `Vec::remove(0)` as a queue.** It works and passes small tests, then becomes slow when the queue holds thousands of items (see the benchmark). Use `VecDeque`.

**2. Popping from an empty stack and calling `unwrap`.** `stack.pop().unwrap()` panics when the stack is empty. In `is_balanced` the closing bracket `)` in `"a)b"` meets an empty stack, and that is a normal answer (`false`), not a crash, so we compare the `Option` instead of unwrapping it.

**3. Forgetting the wrap-around.** Writing `self.head + 1` instead of `(self.head + 1) % self.slots.len()` works until the head reaches the end of the block, then panics with an index out of bounds. The random test
catches it quickly, which is why such a test is worth writing.

## Try it yourself

1. Type `stacks.rs` and `queues.rs` yourself. Run `cargo test stacks`, `cargo test queues`, `cargo clippy` and `cargo fmt`.
2. Add your own bracket cases to the test, such as `"<>"` (not supported yet: what do you expect, and what happens?).
3. Teach `is_balanced` the pair `<` `>` and add tests for it.
4. Run the benchmark and compare the two numbers with the ones above.
5. Commit: `git add code/p07-algorithms && git commit -m "feat(p07): stacks and ring-buffer queue"`.

## Exercises

### Exercise 1 — Trace the stack (easy)

Show the stack after every character while `is_balanced("{[(a)]")` runs, and give the result. Then say which kind of mistake (wrong order, missing closing bracket, extra closing bracket) each of `"([)]"`, `"(()"` and `"a)b"` is.

<details>
<summary>Hint</summary>

An opening bracket is pushed; a closing one pops and compares. Write the stack from the bottom to the top.

</details>

<details>
<summary>Answer</summary>

```text
{   -> [ { ]
[   -> [ {, [ ]
(   -> [ {, [, ( ]
a   -> (ignored)
)   -> pop ( matches  -> [ {, [ ]
]   -> pop [ matches  -> [ { ]
end -> the stack still holds { -> false
```

The text is not balanced: the `{` was never closed. `"([)]"` is a wrong order (the `)` meets `[` on top), `"(()"` has a missing closing bracket (the stack is not empty at the end), and `"a)b"` has an extra closing bracket (the stack is empty when `)` comes).

</details>

### Exercise 2 — A stack that knows its minimum (medium)

Write `MinStack<T: Ord + Copy>` with `push`, `pop` and `min() -> Option<T>`, where **all three are `O(1)`** (not a scan over the items). Write a test that compares it with a scan
over a `Vec` on 2,000 random operations.

<details>
<summary>Hint</summary>

Store, next to each value, the smallest value in the stack up to and including that entry. The minimum of the whole stack is the second number of the top entry.

</details>

<details>
<summary>Answer</summary>

```rust
pub struct MinStack<T> {
    // each entry: the value, and the smallest value in the stack up to and including it
    items: Vec<(T, T)>,
}

impl<T: Ord + Copy> MinStack<T> {
    pub fn new() -> MinStack<T> {
        MinStack { items: Vec::new() }
    }

    pub fn push(&mut self, value: T) {
        let smallest = match self.items.last() {
            Some(&(_, m)) if m < value => m,
            _ => value,
        };
        self.items.push((value, smallest));
    }

    pub fn pop(&mut self) -> Option<T> {
        self.items.pop().map(|(value, _)| value)
    }

    /// The smallest value in the stack: O(1).
    pub fn min(&self) -> Option<T> {
        self.items.last().map(|&(_, m)| m)
    }
}
```

```rust
    #[test]
    fn min_stack_matches_a_scan() {
        let mut rng = Rng::new(3);
        let mut s: MinStack<u64> = MinStack::new();
        let mut model: Vec<u64> = Vec::new();
        for _ in 0..2000 {
            if model.is_empty() || rng.below(3) > 0 {
                let x = rng.next_u64() % 50;
                s.push(x);
                model.push(x);
            } else {
                assert_eq!(s.pop(), model.pop());
            }
            assert_eq!(s.min(), model.iter().copied().min());
        }
    }
```

Why it works: when an entry is pushed, its second number is `min(value, second number of the entry below)`. When an entry is popped, the entry below still holds the correct minimum for the
smaller stack. Cost: `O(1)` time for every operation and `O(n)` extra space (one more number per item).

</details>

### Exercise 3 — A queue from two stacks (hard)

Build `QueueFromStacks<T>` with `push_back` and `pop_front` using only two `Vec`s as stacks: an `inbox` where new items are pushed, and an `outbox` from which items are popped. When the outbox is empty, move all
items from the inbox to the outbox (which reverses their order). Test it against `VecDeque`. Why is `pop_front` amortized `O(1)` even though one call can cost `O(n)`?

<details>
<summary>Hint</summary>

Popping every item of a stack and pushing it onto another stack reverses the order. A reversed LIFO order is FIFO order.

</details>

<details>
<summary>Answer</summary>

```rust
pub struct QueueFromStacks<T> {
    inbox: Vec<T>,  // new items are pushed here
    outbox: Vec<T>, // items leave from here, oldest on top
}

impl<T> QueueFromStacks<T> {
    pub fn new() -> QueueFromStacks<T> {
        QueueFromStacks {
            inbox: Vec::new(),
            outbox: Vec::new(),
        }
    }

    pub fn push_back(&mut self, value: T) {
        self.inbox.push(value);
    }

    pub fn pop_front(&mut self) -> Option<T> {
        if self.outbox.is_empty() {
            while let Some(x) = self.inbox.pop() {
                self.outbox.push(x); // reverses the order
            }
        }
        self.outbox.pop()
    }

    pub fn len(&self) -> usize {
        self.inbox.len() + self.outbox.len()
    }

    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }
}
```

```rust
    #[test]
    fn queue_from_stacks_matches_vec_deque() {
        let mut rng = Rng::new(8);
        let mut q: QueueFromStacks<u64> = QueueFromStacks::new();
        let mut model: VecDeque<u64> = VecDeque::new();
        for _ in 0..5000 {
            if rng.below(5) < 3 {
                let x = rng.next_u64() % 100;
                q.push_back(x);
                model.push_back(x);
            } else {
                assert_eq!(q.pop_front(), model.pop_front());
            }
            assert_eq!(q.len(), model.len());
        }
        assert_eq!(q.is_empty(), model.is_empty());
    }
```

Each item is moved from the inbox to the outbox **at most once** in its life, so over `n` operations the transfers cost at most `n` moves in total. A single `pop_front` that finds an empty outbox can cost `O(n)`, but
the average is `O(1)`: **amortized** (Lesson 07.2).

</details>

## Quiz

1. What do LIFO and FIFO mean, and which structure follows which?

<details>
<summary>Answer</summary>

LIFO (last in, first out) is a stack; FIFO (first in, first out) is a queue.

</details>

2. Why is `Vec::remove(0)` a bad way to take the next item from a queue?

<details>
<summary>Answer</summary>

It shifts all other items one place to the left, `O(n)` per removal, so emptying a queue of `n` items costs `O(n²)`. `VecDeque::pop_front` is `O(1)`.

</details>

3. In a ring buffer with capacity 8, `head = 6` and `len = 3`. In which slot does the next `push_back` write, and which slots are in use?

<details>
<summary>Answer</summary>

`(6 + 3) % 8 = 1`, so slot 1. The items are in slots 6, 7 and 0.

</details>

4. Which structure does the "undo" command of an editor need, a stack or a queue?

<details>
<summary>Answer</summary>

A stack: you undo the most recent change first (LIFO).

</details>

## Summary

- A stack is LIFO: push and pop at one end. A `Vec` is a stack with `O(1)` operations. Use it for matching brackets, undo, and (later) recursion.
- A queue is FIFO: join at the back, leave at the front. `Vec::remove(0)` is `O(n)`; a ring buffer makes both ends `O(1)` with a `head` index and `%` wrap-around.
- `std::collections::VecDeque` is the standard ring buffer. Use `Vec` as a stack and `VecDeque` as a queue.
- A queue can be built from two stacks; the cost is `O(1)` amortized, because every item is moved only once.
- Handle `None` instead of unwrapping when reading outside input; test with broken input.
- Compare your structure with the standard one on random operations.

## Sources and further reading

- [std::collections::VecDeque](https://doc.rust-lang.org/std/collections/struct.VecDeque.html) — the standard double-ended queue, a growable ring buffer.
- [std::collections — When Should You Use Which Collection?](https://doc.rust-lang.org/std/collections/index.html#when-should-you-use-which-collection) — the official guide to choosing a collection.
- [Vec::push](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.push) and [Vec::pop](https://doc.rust-lang.org/std/vec/struct.Vec.html#method.pop) — the stack operations of `Vec`.

---

[🏠 Start](../START-HERE.md) · [← 07.4 Linked lists](04-linked-lists.md) · [📋 Phase 07](README.md) · [07.6 How hash tables work →](06-hash-tables.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
