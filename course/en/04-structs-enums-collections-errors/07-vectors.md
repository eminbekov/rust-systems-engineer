<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 04.6 Modules, crates and packages](06-modules-crates-and-packages.md) · [📋 Phase 04](README.md) · [04.8 Strings in depth →](08-strings-in-depth.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/04-structs-enums-collections-errors/07-vectors.md) <!-- nav -->

# Lesson 04.7 — Vectors

> **Phase 04 — Structs, enums, collections and errors** · Lesson 7 of 12 · ⏱ about 4 hours (2 sittings)

## What you will learn

- What a *collection* is and why a program needs one.
- How to create a vector, add items to it, remove items from it and read them safely.
- How to loop over a vector in three ways: read only, change items, and take ownership.
- Why the borrowing rules stop you from keeping a reference into a vector while you push to it.
- What *length* and *capacity* mean, and how to store different kinds of data in one vector with an enum.

## Before you start

- You finished [Lesson 04.6 — Modules, crates and packages](06-modules-crates-and-packages.md).
- You know what the stack and the heap are ([Lesson 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)), how ownership and borrowing work ([Lesson 03.4](../03-ownership-and-borrowing/04-references-and-borrowing.md) and [Lesson 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md)), and what a slice is ([Lesson 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md)).
- Create a project for this lesson:

```bash
cd ~/projects/rust
cargo new code/p04-vectors
cd code/p04-vectors
```

## New words

| Word | Meaning |
|---|---|
| **collection** | A value that holds many other values. The data of a collection lives on the heap, so it can grow and shrink while the program runs. |
| **vector** | The most used collection: a growable list of values of the same type, stored one after another in memory. |
| **`Vec`** | The name of the vector type in Rust. It is short for *vector*. You write `Vec<i32>` for "a vector of `i32` numbers". |
| **`<T>` (type parameter)** | The type inside the angle brackets, for example `i32` in `Vec<i32>`. It tells Rust what kind of items the collection holds. Full explanation in [Lesson 05.1 — Generics](../05-rust-intermediate/01-generics.md). |
| **index** | The position of an item in the vector. The first item has index `0`. |
| **length** (`len`) | How many items are in the vector right now. |
| **capacity** | How many items the vector can hold before it must ask for more memory. Capacity is always at least the length. |
| **reallocation** | Moving the items to a new, bigger piece of heap memory because the old one is full. |
| **iterator** | A value that gives you the items of a collection one by one. `.iter()` makes one. Full explanation in [Lesson 05.5 — Iterators](../05-rust-intermediate/05-iterators.md). |
| **panic** | A crash that Rust starts on purpose when the program cannot continue, for example when you read an index that does not exist. |

## 1. What is a vector?

So far each variable held one value, or a fixed group of values such as a tuple or an array. Real
programs need to hold *as many items as the data has*: all the lines of a file, all the users who
logged in, all the scores of a class. You do not know the number when you write the code.

A **collection** is a value that holds many other values. The Rust standard library has several
collections. The most used one is the **vector**, written `Vec<T>`.

Think of a vector as a row of numbered boxes that can get longer. Every box holds a value of the
same type. You can add a box at the end, take the last box away, and look into any box by its
number.

An **array** (for example `[i32; 3]`) has a size fixed at compile time. A **vector** can grow
and shrink while the program runs. That is why vectors are far more common.

### How a vector looks in memory

A vector is a small struct on the stack with three fields. The items themselves are on the heap
(remember: [Lesson 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)).

```text
 stack                           heap
┌─────────────────────┐        ┌──────┬──────┬──────┬──────┐
│ v: pointer ─────────┼──────▶ │  10  │  20  │  30  │ free │
│    length   = 3     │        └──────┴──────┴──────┴──────┘
│    capacity = 4     │          index 0   1      2     (room for 1 more)
└─────────────────────┘
```

- the **pointer** says where the items start on the heap;
- the **length** says how many boxes are used (3);
- the **capacity** says how many boxes exist (4).

Because the items are on the heap, the vector can ask for a bigger area when it needs one.

## 2. Creating a vector

There are two common ways.

```rust
fn main() {
    let mut scores: Vec<i32> = Vec::new();
    scores.push(90);
    scores.push(75);
    scores.push(88);

    let names = vec!["Ali", "Vali", "Sara"];

    println!("{:?}", scores);
    println!("{:?}", names);
    println!("scores has {} items", scores.len());
    println!("is empty? {}", scores.is_empty());
}
```

**What happens:**

1. `Vec::new()` creates an empty vector. It has no items and has not asked the heap for memory yet.
   Rust cannot guess the item type of an empty vector, so we write it: `Vec<i32>`.
2. `scores.push(90)` adds `90` at the end. `push` changes the vector, so `scores` must be declared
   with `mut`.
3. `vec![...]` is a **macro** (a name that ends with `!`; you have used `println!` already). It
   creates a vector and fills it. Here Rust sees strings, so it knows this is a `Vec<&str>`.
4. `{:?}` prints a vector in the *debug* format: `[90, 75, 88]`.
5. `len()` returns the number of items. `is_empty()` returns `true` when there are none.

```text
[90, 75, 88]
["Ali", "Vali", "Sara"]
scores has 3 items
is empty? false
```

There is a third way, `Vec::with_capacity(n)`, which reserves room for `n` items at the start.
You will see why in section 6.

> 📏 **Convention:** Use `vec![...]` when you already know the first items. Use `Vec::new()`
> (or `Vec::with_capacity(n)`) and `push` when you build the vector step by step.

## 3. Adding and removing items

```rust
fn main() {
    let mut v = vec![10, 20, 30];

    v.push(40);
    println!("after push:   {:?}", v);

    let last = v.pop();
    println!("popped:       {:?}", last);
    println!("after pop:    {:?}", v);

    v.insert(1, 15);
    println!("after insert: {:?}", v);

    let removed = v.remove(0);
    println!("removed:      {}", removed);
    println!("after remove: {:?}", v);

    v.clear();
    println!("after clear:  {:?}", v);
    println!("pop on empty: {:?}", v.pop());
}
```

**What happens:**

1. `push(40)` adds `40` to the end.
2. `pop()` removes the last item and returns it **inside an `Option`**: `Some(40)`. Why `Option`?
   Because the vector might be empty, and then there is nothing to give back. `Option` is the
   type for "a value or nothing" ([Lesson 04.4 — Option and match](04-option-and-match.md)).
3. `insert(1, 15)` puts `15` at index 1 and moves every item after it one place to the right.
4. `remove(0)` takes out the item at index 0, returns it (not in an `Option`), and moves every
   item after it one place to the left.
5. `clear()` removes all items. The last line shows that `pop()` on an empty vector gives `None`.

```text
after push:   [10, 20, 30, 40]
popped:       Some(40)
after pop:    [10, 20, 30]
after insert: [10, 15, 20, 30]
removed:      10
after remove: [15, 20, 30]
after clear:  []
pop on empty: None
```

> ⚡ **Performance:** `push` and `pop` work at the end, so they are cheap. `insert` and `remove`
> in the middle or at the start must move all the items after that place, so they get slower as
> the vector gets longer. If you often remove from the front, a different collection is better
> (you will meet one in Phase 07 — Algorithms and data structures).

## 4. Reading items: `[]` or `get`

There are two ways to read an item, and the difference matters.

```rust
fn main() {
    let v = vec![10, 20, 30];

    let second = v[1];
    println!("v[1] = {}", second);

    match v.get(5) {
        Some(x) => println!("v[5] = {}", x),
        None => println!("there is no item at index 5"),
    }

    println!("first: {:?}", v.first());
    println!("last:  {:?}", v.last());
    println!("get(2): {:?}", v.get(2));
}
```

**What happens:**

1. `v[1]` reads the item at index 1 (the second item, because counting starts at 0). The index
   must be a `usize` (an unsigned integer the size of a memory address).
2. `v.get(5)` also reads by index, but it returns an `Option`: `Some(&item)` when the index
   exists and `None` when it does not. The program does not crash; you decide what to do with
   `None` using `match`.
3. `first()` and `last()` are shortcuts for `get(0)` and `get(len - 1)`. They also return an
   `Option`.

```text
v[1] = 20
there is no item at index 5
first: Some(10)
last:  Some(30)
get(2): Some(30)
```

What if you use `[]` with an index that does not exist?

```rust
fn main() {
    let v = vec![10, 20, 30];
    let i = 7;
    println!("about to read index {}", i);
    println!("{}", v[i]);
}
```

```text
about to read index 7

thread 'main' (3849831) panicked at src/main.rs:5:21:
index out of bounds: the len is 3 but the index is 7
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**What happens:** the program starts a **panic**: it prints a message with the file and line,
and stops. (The number in parentheses is an internal thread number, and the file name is that of
the scratch project; yours will differ.) Rust refuses to read memory that is not part of the
vector. Without such a check, a program could silently read garbage memory and carry on with a wrong
answer. A loud crash is much better than a quiet wrong answer.

> 📏 **Convention:** Use `v[i]` when an invalid index would be a bug in your own program, and a
> crash is the right reaction. Use `v.get(i)` when the index comes from somewhere you do not
> control (user input, a file, a network message).

> 🔒 **Security:** Never use data from outside the program as an index with `[]`. A user who
> sends `7` to a program that holds 3 items can crash it. With `get`, the program can answer with
> an error message instead. Panics from bad indexes are a common way to crash a server.

## 5. Looping over a vector

You learned `for` loops in [Lesson 02.5](../02-programming-basics/05-control-flow.md). A `for` loop can also walk through
a vector. There are three forms, and each one has a different effect on ownership.

```rust
fn main() {
    let mut v = vec![1, 2, 3];

    // 1) read-only loop
    for x in &v {
        println!("item: {}", x);
    }

    // 2) loop that changes every item
    for x in &mut v {
        *x *= 10;
    }
    println!("after: {:?}", v);

    // 3) loop with a counter
    for (i, x) in v.iter().enumerate() {
        println!("index {} -> {}", i, x);
    }

    // 4) loop that takes ownership
    for x in v {
        println!("owned: {}", x);
    }
    // v is gone here
}
```

**What happens:**

1. `for x in &v` **borrows** the vector (a shared reference). In each round `x` is a reference
   `&i32` to one item. The vector is still yours after the loop.
2. `for x in &mut v` borrows the vector **mutably**. `x` is a `&mut i32`. To change the number
   behind the reference you write `*x`; the star means "the value this reference points to"
   ([Lesson 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md)). `*x *= 10` multiplies the item by 10.
3. `v.iter()` makes an **iterator**: a helper that hands out the items one by one.
   `.enumerate()` wraps it so each round gives a pair `(index, item)`. Use it when you need the
   position as well as the value. (Iterators get a full lesson later: [Lesson 05.5](../05-rust-intermediate/05-iterators.md).)
4. `for x in v` (no `&`) **moves** the vector into the loop. Each `x` is an owned value. After the
   loop `v` is gone and you cannot use it any more.

```text
item: 1
item: 2
item: 3
after: [10, 20, 30]
index 0 -> 10
index 1 -> 20
index 2 -> 30
owned: 10
owned: 20
owned: 30
```

> 💡 **Tip:** Write `for x in &v` by default. Use `&mut v` when you need to change items. Use
> plain `v` only when you are done with the vector and want to take the items out.

## 6. Ownership and vectors: the classic error

This is the most famous Rust error for vectors. Read the code first and think about whether it
should be allowed.

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let first = &v[0];
    v.push(4);
    println!("first = {}", first);
}
```

```text
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 --> src/main.rs:4:5
  |
3 |     let first = &v[0];
  |                  - immutable borrow occurs here
4 |     v.push(4);
  |     ^^^^^^^^^ mutable borrow occurs here
5 |     println!("first = {}", first);
  |                            ----- immutable borrow later used here
```

**What the error means:** `first` is a reference to the first box. Then `push` runs. If the
vector is full, `push` allocates a bigger area on the heap, copies the items there and frees the
old area (that is a **reallocation**). Then `first` would point into freed memory, a *dangling
reference*. The borrow checker (the part of the compiler that checks the borrowing rules) sees
this danger at compile time and rejects the program.

```text
 before push                          after push (vector was full)
 first ──▶ ┌────┬────┬────┐           old area: freed ✗   (first still points here!)
           │ 1  │ 2  │ 3  │           new area: ┌────┬────┬────┬────┬────┬────┬────┬────┐
           └────┴────┴────┘                     │ 1  │ 2  │ 3  │ 4  │    │    │    │    │
```

**How to fix it.** Either finish using `first` before you push, or copy the number instead of
borrowing it:

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    let first = v[0]; // copies the number; no borrow is kept
    v.push(4);
    println!("first = {}", first);
    println!("{:?}", v);
}
```

```text
first = 1
[1, 2, 3, 4]
```

**What happens:** `v[0]` of an `i32` makes a copy (numbers implement `Copy`,
[Lesson 03.3](../03-ownership-and-borrowing/03-move-copy-and-clone.md)). `first` is now an independent number, so `push` is free to move the items.

## 7. Length and capacity

Asking the heap for memory is slow compared with ordinary work. So a vector asks for *more than
it needs* and uses the extra room for later `push` calls. This extra room is the difference
between **length** and **capacity**.

```rust
fn main() {
    let mut v: Vec<i32> = Vec::new();
    println!("len {} capacity {}", v.len(), v.capacity());
    for i in 1..=9 {
        v.push(i);
        println!("len {} capacity {}", v.len(), v.capacity());
    }

    let mut w: Vec<i32> = Vec::with_capacity(100);
    for i in 0..100 {
        w.push(i);
    }
    println!("w: len {} capacity {}", w.len(), w.capacity());
}
```

**What happens:**

1. A new empty vector has capacity 0: it has asked for no memory.
2. The first `push` reserves room for 4 items. The 5th `push` finds the vector full, so it
   reallocates to capacity 8; the 9th `push` makes it 16. The capacity grows by doubling.
3. `Vec::with_capacity(100)` reserves room for 100 items at once, so the 100 `push` calls never
   reallocate.

```text
len 0 capacity 0
len 1 capacity 4
len 2 capacity 4
len 3 capacity 4
len 4 capacity 4
len 5 capacity 8
len 6 capacity 8
len 7 capacity 8
len 8 capacity 8
len 9 capacity 16
w: len 100 capacity 100
```

The exact numbers (0, 4, 8, 16) are a detail of the current standard library, not a promise.
Your program must never depend on them.

> ⚡ **Performance:** If you know (or can guess) how many items you will add, call
> `Vec::with_capacity(n)`. It avoids repeated reallocation and copying. But do not guess wildly:
> measure first ([Lesson 05.12 — Benchmarking basics](../05-rust-intermediate/12-benchmarking-basics.md)).
> Reserving memory for a million items you never add wastes memory.

## 8. Useful methods

```rust
fn main() {
    let mut v = vec![5, 3, 8, 3, 1, 8];

    println!("contains 8? {}", v.contains(&8));

    v.sort();
    println!("sorted:   {:?}", v);

    v.dedup();
    println!("dedup:    {:?}", v);

    v.reverse();
    println!("reversed: {:?}", v);

    v.extend([100, 200]);
    println!("extended: {:?}", v);

    v.truncate(3);
    println!("truncate: {:?}", v);

    v.swap(0, 2);
    println!("swap:     {:?}", v);

    let total: i32 = v.iter().sum();
    println!("sum = {}", total);
    println!("max = {:?}", v.iter().max());

    let part = &v[1..3];
    println!("slice: {:?}", part);

    let mut a = vec![1, 2];
    let mut b = vec![3, 4];
    a.append(&mut b);
    println!("a = {:?}, b = {:?}", a, b);
}
```

**What happens:**

1. `contains(&8)` is `true` if some item equals 8. It takes a reference to the value to look for.
2. `sort()` puts the items in increasing order, in place (it changes `v`).
3. `dedup()` removes items that are equal to the item just before them. It works as expected
   only on a sorted vector, which is why we sorted first.
4. `reverse()` flips the order. `extend([100, 200])` adds all items of another list at the end.
   `truncate(3)` keeps only the first 3 items. `swap(0, 2)` exchanges the items at two indexes.
5. `v.iter().sum()` adds all the items. `v.iter().max()` finds the biggest one and returns an
   `Option<&i32>` (`None` for an empty vector). Both are iterator methods; you will use them a lot
   after [Lesson 05.5](../05-rust-intermediate/05-iterators.md).
6. `&v[1..3]` is a **slice** (a borrowed view of a part of the vector, [Lesson 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md)): items at
   indexes 1 and 2. The end of the range is not included.
7. `a.append(&mut b)` moves all items of `b` into `a`, and leaves `b` empty.

```text
contains 8? true
sorted:   [1, 3, 3, 5, 8, 8]
dedup:    [1, 3, 5, 8]
reversed: [8, 5, 3, 1]
extended: [8, 5, 3, 1, 100, 200]
truncate: [8, 5, 3]
swap:     [3, 5, 8]
sum = 16
max = Some(8)
slice: [5, 8]
a = [1, 2, 3, 4], b = []
```

> 📏 **Convention:** A function that only reads a list should take a slice, `&[T]`, not
> `&Vec<T>`. A slice accepts a vector, an array and part of either, so the function is more
> flexible. Compare: `fn average(numbers: &[f64])`.

## 9. Different kinds of items: a vector of enums or structs

A vector holds items of **one** type. What if your data has several kinds? Wrap them in an
**enum** ([Lesson 04.3](03-enums.md)): the enum is the one type, and its variants are the kinds. A vector of **structs**
([Lesson 04.1](01-structs.md)) is also very common.

```rust
#[derive(Debug)]
enum Reading {
    Temperature(f64),
    Missing,
}

#[derive(Debug)]
struct Student {
    name: String,
    score: u32,
}

fn main() {
    let readings = vec![
        Reading::Temperature(21.5),
        Reading::Missing,
        Reading::Temperature(19.0),
    ];
    for r in &readings {
        match r {
            Reading::Temperature(t) => println!("temperature {t}"),
            Reading::Missing => println!("no data"),
        }
    }

    let mut class = Vec::new();
    class.push(Student { name: String::from("Ali"), score: 90 });
    class.push(Student { name: String::from("Sara"), score: 72 });
    for s in &class {
        println!("{} has {}", s.name, s.score);
    }
    println!("{:?}", class[0]);
}
```

**What happens:**

1. `Reading` has two variants: a measured temperature, or a missing value. All three items of
   `readings` have the same type, `Reading`, even though they look different.
2. The `match` inside the loop handles every variant. Because `r` is a reference, `t` is a
   reference too; `{t}` prints the number behind it.
3. `class` has no type written. Rust infers `Vec<Student>` from the first `push`.
4. `class[0]` borrows the first student; `{:?}` prints it because of `#[derive(Debug)]`.

```text
temperature 21.5
no data
temperature 19
Ali has 90
Sara has 72
Student { name: "Ali", score: 90 }
```

Note that `19.0` prints as `19`: the `Display` format of a float drops a useless `.0`.

## 10. Testing code that uses vectors

> 🧪 **Testing:** Test the normal case, the edge case (one item) and the empty case. Empty and
> one-item inputs are where most bugs hide. Here is a function that returns `None` for an empty
> list instead of crashing on a division by zero, with two tests.

```rust
/// Returns the average of the numbers, or None for an empty slice.
fn average(numbers: &[f64]) -> Option<f64> {
    if numbers.is_empty() {
        return None;
    }
    let mut total = 0.0;
    for n in numbers {
        total += n;
    }
    Some(total / numbers.len() as f64)
}

fn main() {
    let v = vec![2.0, 4.0, 9.0];
    println!("{:?}", average(&v));
    println!("{:?}", average(&[]));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn average_of_three() {
        assert_eq!(average(&[2.0, 4.0, 9.0]), Some(5.0));
    }

    #[test]
    fn average_of_empty_is_none() {
        assert_eq!(average(&[]), None);
    }
}
```

**What happens:**

1. `numbers.len() as f64` converts the length (a `usize`) to a float so we can divide.
2. `average(&v)` passes `&Vec<f64>`; Rust turns it into the slice `&[f64]` automatically.
3. `cargo test` runs the two functions marked `#[test]` ([Lesson 02.8](../02-programming-basics/08-your-first-tests.md)).

```text
Some(5.0)
None
```

```text
running 2 tests
test tests::average_of_empty_is_none ... ok
test tests::average_of_three ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Common mistakes

**1. Forgetting `mut`.**

```rust
fn main() {
    let v = Vec::new();
    v.push(1);
    println!("{:?}", v);
}
```

```text
error[E0596]: cannot borrow `v` as mutable, as it is not declared as mutable
 --> src/main.rs:3:5
  |
3 |     v.push(1);
  |     ^ cannot borrow as mutable
  |
help: consider changing this to be mutable
  |
2 |     let mut v = Vec::new();
  |         +++
```

`push` changes the vector, so the variable must be `mut`. Fix: `let mut v = Vec::new();`.

**2. Pushing the wrong type.**

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    v.push("four");
    println!("{:?}", v);
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:3:12
  |
3 |     v.push("four");
  |       ---- ^^^^^^ expected integer, found `&str`
  |       |
  |       arguments to this method are incorrect
```

A vector holds one type. `v` is a vector of integers, so a string cannot go in. Fix: push a
number, or use an enum if you really need two kinds.

**3. Using a vector after `for x in v`.**

```rust
fn main() {
    let names = vec![String::from("Ali"), String::from("Vali")];
    for n in names {
        println!("{}", n);
    }
    println!("{}", names.len());
}
```

```text
error[E0382]: borrow of moved value: `names`
 --> src/main.rs:6:20
  |
2 |     let names = vec![String::from("Ali"), String::from("Vali")];
  |         ----- move occurs because `names` has type `Vec<String>`, which does not implement the `Copy` trait
3 |     for n in names {
  |              ----- `names` moved due to this implicit call to `.into_iter()`
...
6 |     println!("{}", names.len());
  |                    ^^^^^ value borrowed here after move
  |
help: consider iterating over a slice of the `Vec<String>`'s content to avoid moving into the `for` loop
  |
3 |     for n in &names {
  |              +
```

`for n in names` moves the vector into the loop (rule from [Lesson 03.2](../03-ownership-and-borrowing/02-ownership-rules.md)). The compiler even
suggests the fix: write `for n in &names`.

**4. Keeping a reference while pushing.** This is the E0502 error from section 6. Finish with the
reference first, or copy the value.

**5. Using `v[i]` with an index you did not check.** Section 4: it panics. Use `get`.

## Try it yourself

Inside `code/p04-vectors`, replace `src/main.rs` with a small "to-do list":

```rust
fn main() {
    let mut todo: Vec<String> = Vec::new();
    todo.push(String::from("read lesson 04.7"));
    todo.push(String::from("do the exercises"));
    todo.push(String::from("commit to Git"));

    for (i, task) in todo.iter().enumerate() {
        println!("{}. {}", i + 1, task);
    }

    let done = todo.remove(0);
    println!("done: {}", done);
    println!("left: {}", todo.len());
}
```

1. Run it with `cargo run`. Expected output:

```text
1. read lesson 04.7
2. do the exercises
3. commit to Git
done: read lesson 04.7
left: 2
```

2. Add a fourth task with `push` and run again. What number does it get?
3. Change `todo.remove(0)` to `todo.remove(10)` and run. Read the panic message. Which words tell
   you the problem? (Answer: *index out of bounds*, the length and the index.)
4. Put the line back to `remove(0)`, then save your work in Git as you learned in Phase 01:
   `git add .` and `git commit -m "Add vector practice"`.

## Exercises

### Exercise 1 — Shopping list (easy)

Make a vector with `"milk"`, `"bread"`, `"eggs"`. Add `"apples"`. Print each item with its
number starting at 1, then print how many items there are. Expected output:

```text
1. milk
2. bread
3. eggs
4. apples
4 items
```

<details>
<summary>Hint</summary>

Use `push` for the new item and `.iter().enumerate()` for the numbers. `enumerate` starts at 0,
so print `i + 1`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let mut list = vec!["milk", "bread", "eggs"];
    list.push("apples");
    for (i, item) in list.iter().enumerate() {
        println!("{}. {}", i + 1, item);
    }
    println!("{} items", list.len());
}
```

Why it works: `vec!` builds the first three items, `push` adds the fourth, and `enumerate`
gives `(index, item)` pairs.

</details>

### Exercise 2 — The third item, safely (easy)

Write `fn third(v: &[i32]) -> Option<i32>` that returns the third item, or `None` if there are
fewer than three. It must never panic. Call it with `&[1, 2, 3, 4]`, `&[1, 2]` and `&[]`.

<details>
<summary>Hint</summary>

Use `v.get(2)`. It returns `Option<&i32>`. The method `.copied()` turns an `Option<&i32>` into an
`Option<i32>`.

</details>

<details>
<summary>Answer</summary>

```rust
fn third(v: &[i32]) -> Option<i32> {
    v.get(2).copied()
}

fn main() {
    println!("{:?}", third(&[1, 2, 3, 4]));
    println!("{:?}", third(&[1, 2]));
    println!("{:?}", third(&[]));
}
```

```text
Some(3)
None
None
```

Why it works: `get(2)` checks the index, so a short vector gives `None` instead of a panic.
`.copied()` copies the `i32` out of the reference.

</details>

### Exercise 3 — Smallest and largest, with tests (medium)

Write `fn min_max(v: &[i32]) -> Option<(i32, i32)>` that returns the smallest and the largest
item as a tuple, or `None` for an empty slice. Use a loop (no `.iter().max()`). Write **three
tests**: a normal list, a list with one item and an empty list.

<details>
<summary>Hint</summary>

Return `None` first if `v.is_empty()`. Otherwise start with `min = v[0]` and `max = v[0]`, then
loop and update them. In the loop, `for &n in v` gives you each number by value.

</details>

<details>
<summary>Answer</summary>

```rust
/// Returns (smallest, largest) or None for an empty slice.
fn min_max(v: &[i32]) -> Option<(i32, i32)> {
    if v.is_empty() {
        return None;
    }
    let mut min = v[0];
    let mut max = v[0];
    for &n in v {
        if n < min {
            min = n;
        }
        if n > max {
            max = n;
        }
    }
    Some((min, max))
}

fn main() {
    println!("{:?}", min_max(&[4, -2, 9, 0]));
    println!("{:?}", min_max(&[]));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_both_ends() {
        assert_eq!(min_max(&[4, -2, 9, 0]), Some((-2, 9)));
    }

    #[test]
    fn one_item_is_both() {
        assert_eq!(min_max(&[7]), Some((7, 7)));
    }

    #[test]
    fn empty_gives_none() {
        assert_eq!(min_max(&[]), None);
    }
}
```

```text
running 3 tests
test tests::empty_gives_none ... ok
test tests::one_item_is_both ... ok
test tests::finds_both_ends ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the empty check makes `v[0]` safe. The pattern `&n` in `for &n in v` copies each
number out of its reference.

</details>

### Exercise 4 — Even and odd (medium)

Write `fn split_even_odd(v: &[i32]) -> (Vec<i32>, Vec<i32>)` that returns two new vectors: the
even numbers and the odd numbers, each in the original order. For `[1, 2, 3, 4, 5, 6]` print:

```text
even: [2, 4, 6]
odd:  [1, 3, 5]
```

<details>
<summary>Hint</summary>

Create two empty vectors with `Vec::new()`, loop over `v`, and use `n % 2 == 0` to choose which
one to `push` into. Return them as a tuple `(even, odd)`.

</details>

<details>
<summary>Answer</summary>

```rust
fn split_even_odd(v: &[i32]) -> (Vec<i32>, Vec<i32>) {
    let mut even = Vec::new();
    let mut odd = Vec::new();
    for &n in v {
        if n % 2 == 0 {
            even.push(n);
        } else {
            odd.push(n);
        }
    }
    (even, odd)
}

fn main() {
    let (e, o) = split_even_odd(&[1, 2, 3, 4, 5, 6]);
    println!("even: {:?}", e);
    println!("odd:  {:?}", o);
}
```

Why it works: the function owns the two new vectors and returns them, so ownership moves to the
caller. `let (e, o) = ...` splits the tuple into two variables.

</details>

### Exercise 5 — Best student (hard)

Use the `Student` struct from section 9. Write `fn best(students: &[Student]) -> Option<&Student>`
that returns a reference to the student with the highest score, or `None` for an empty slice.
For the class Ali 90, Sara 97, Vali 61 it should find Sara.

<details>
<summary>Hint</summary>

Keep a variable `best: Option<&Student> = None`. For each student: if `best` is `None`, set it to
this student; if it is `Some(current)` and the new score is bigger, replace it. Return `best` at
the end. The returned reference is valid because it points into the slice the caller owns.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug)]
struct Student {
    name: String,
    score: u32,
}

fn best(students: &[Student]) -> Option<&Student> {
    let mut best: Option<&Student> = None;
    for s in students {
        match best {
            None => best = Some(s),
            Some(current) => {
                if s.score > current.score {
                    best = Some(s);
                }
            }
        }
    }
    best
}

fn main() {
    let class = vec![
        Student { name: String::from("Ali"), score: 90 },
        Student { name: String::from("Sara"), score: 97 },
        Student { name: String::from("Vali"), score: 61 },
    ];
    match best(&class) {
        Some(s) => println!("best: {} ({})", s.name, s.score),
        None => println!("no students"),
    }
    println!("{:?}", best(&[]).is_none());
}
```

```text
best: Sara (97)
true
```

Why it works: the function only borrows the slice and returns a borrowed `&Student`; nothing is
copied or moved. (How Rust knows the returned reference is valid is explained in
[Lesson 05.3 — Lifetimes](../05-rust-intermediate/03-lifetimes.md).)

</details>

## Quiz

1. What is the difference between an array and a vector?

<details>
<summary>Answer</summary>

An array has a fixed size known at compile time. A vector can grow and shrink while the program
runs, because its items are stored on the heap.

</details>

2. What does `v.pop()` return, and why?

<details>
<summary>Answer</summary>

An `Option`: `Some(item)` with the removed last item, or `None` when the vector is empty. The
`Option` is there because there may be nothing to remove.

</details>

3. What happens when you run `let x = v[10];` on a vector with 3 items? How can you avoid it?

<details>
<summary>Answer</summary>

The program panics with "index out of bounds". To avoid a crash, use `v.get(10)`, which returns
`None` for a missing index.

</details>

4. Why does Rust reject code that holds `&v[0]` and then calls `v.push(...)`?

<details>
<summary>Answer</summary>

`push` may reallocate (move the items to a bigger heap area). The old reference would then point
to freed memory. The borrowing rules (no mutable borrow while a shared borrow is alive) stop this
at compile time.

</details>

5. What is the difference between length and capacity?

<details>
<summary>Answer</summary>

Length is the number of items in the vector now. Capacity is the number of items it can hold
before it must reallocate. Capacity is always greater than or equal to the length.

</details>

6. What is the difference between `for x in &v`, `for x in &mut v` and `for x in v`?

<details>
<summary>Answer</summary>

`&v` borrows the vector and gives shared references (read only). `&mut v` borrows it mutably and
gives mutable references (you can change items with `*x`). `v` moves the vector into the loop and
gives owned items; the vector cannot be used afterwards.

</details>

7. A vector holds one type. How can you store a number, a word and "nothing" in one vector?

<details>
<summary>Answer</summary>

Define an enum with a variant for each kind, for example `enum Item { Number(i32), Word(String),
Nothing }`, and make a `Vec<Item>`.

</details>

## Summary

- A vector, `Vec<T>`, is a growable list of items of one type; the items live on the heap.
- Create it with `Vec::new()`, `Vec::with_capacity(n)` or `vec![...]`; change it with `push`,
  `pop`, `insert`, `remove` and `clear`.
- `v[i]` panics on a bad index; `v.get(i)` returns an `Option` and is the safe choice for
  untrusted indexes.
- `for x in &v` reads, `for x in &mut v` changes, `for x in v` consumes the vector.
- You cannot hold a reference into a vector while you push to it, because `push` can move the
  items.
- Length is the number of items; capacity is the room reserved. Use `with_capacity` when you
  know the size.
- Use an enum to put different kinds of data in one vector; take `&[T]` in functions that only
  read.
- Always test the empty and one-item cases.

## Sources and further reading

- [The Rust Programming Language — 8.1 Storing Lists of Values with Vectors](https://doc.rust-lang.org/book/ch08-01-vectors.html) — the official book chapter on vectors.
- [std::vec::Vec](https://doc.rust-lang.org/std/vec/struct.Vec.html) — the full list of vector methods, with examples.
- [Rust by Example — Vectors](https://doc.rust-lang.org/rust-by-example/std/vec.html) — short examples you can run.
- [The Rustonomicon — Implementing Vec](https://doc.rust-lang.org/nomicon/vec/vec.html) — an advanced chapter that builds a vector from scratch (read it much later, after Phase 06).
- [The Rust Programming Language — 9.1 Unrecoverable Errors with panic!](https://doc.rust-lang.org/book/ch09-01-unrecoverable-errors-with-panic.html) — more about panics, including the out-of-bounds example.

---

[🏠 Start](../START-HERE.md) · [← 04.6 Modules, crates and packages](06-modules-crates-and-packages.md) · [📋 Phase 04](README.md) · [04.8 Strings in depth →](08-strings-in-depth.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
