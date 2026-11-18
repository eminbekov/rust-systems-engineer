<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 05.4 Closures](04-closures.md) · [📋 Phase 05](README.md) · [05.6 Testing in depth →](06-testing-in-depth.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/05-rust-intermediate/05-iterators.md) <!-- nav -->

# Lesson 05.5 — Iterators

> **Phase 05 — Rust intermediate** · Lesson 5 of 14 · ⏱ about 4 hours (2 sittings)

## What you will learn

- What an iterator is, and how the `Iterator` trait and its `next` method work.
- The difference between `iter()`, `iter_mut()` and `into_iter()`.
- Why iterators are *lazy*, and the difference between adapters (`map`, `filter`, …) and consumers (`sum`, `collect`, …).
- How to collect results into a `Vec`, `String`, `HashMap` or `Result`.
- How to write your own iterator, and when a plain `for` loop is still the better choice.

## Before you start

- You finished [Lesson 05.4 — Closures](04-closures.md). Almost every iterator method takes a closure.
- You know vectors ([Lesson 04.7](../04-structs-enums-collections-errors/07-vectors.md)), hash maps
  ([Lesson 04.9](../04-structs-enums-collections-errors/09-hash-maps.md)), `Option` and `Result`, and traits ([Lesson 05.2](02-traits.md)).
- You have the package `code/p05-lessons` from Lesson 05.1.

## New words

| Word | Meaning |
|---|---|
| **iterator** | A value that produces a sequence of items one at a time, when you ask for the next one. |
| **`Iterator`** | The standard trait for iterators. Its one required method is `next`. |
| **`Item`** | The associated type of `Iterator`: the type of the things the iterator produces. |
| **adapter** | An iterator method that takes an iterator and returns a *new* iterator (`map`, `filter`, `take`, …). It does nothing by itself. |
| **consumer** | An iterator method that *uses up* the iterator and produces a final result (`sum`, `count`, `collect`, `fold`, …). |
| **lazy** | Does no work until the result is needed. Adapters are lazy. |
| **`collect`** | The consumer that gathers all items into a collection such as `Vec`, `String` or `HashMap`. |
| **`IntoIterator`** | The trait that says "this value can be turned into an iterator". `for` loops use it. |
| **range** | The values `1..5` (1 to 4) and `1..=5` (1 to 5). A range is itself an iterator. |
| **pipeline** | A chain of adapters ending in a consumer: `data.iter().filter(…).map(…).sum()`. |
| **associated type** | A type name inside a trait that each implementation chooses (Lesson 05.2). |

## 1. What an iterator is

An **iterator** is like a bookmark in a list. You ask it "what is the next item?", it gives you one
item and moves the bookmark forward. When the items are finished, it says "nothing more".

In Rust this is the standard trait `Iterator`. Here is its core (simplified):

```rust
trait Iterator {
    type Item;                          // what kind of items it produces
    fn next(&mut self) -> Option<Self::Item>;
}
```

- `type Item` is an **associated type** (Lesson 05.2): each iterator chooses its own item type.
- `next` returns `Some(item)` while there are items and `None` when they are finished. It takes
  `&mut self` because moving the bookmark *changes* the iterator.

You can call `next` yourself:

```rust
fn main() {
    let numbers = vec![10, 20, 30];
    let mut walker = numbers.iter();
    println!("{:?}", walker.next());
    println!("{:?}", walker.next());
    println!("{:?}", walker.next());
    println!("{:?}", walker.next());
}
```

**What happens:**

1. `numbers.iter()` creates an iterator over **references** to the items (`&i32`). The vector is
   unchanged and still owns the numbers.
2. `let mut walker` must be `mut`, because `next` changes the iterator's inner position.
3. The first three calls return `Some(10)`, `Some(20)`, `Some(30)`. The fourth returns `None`: the list is over.

```text
Some(10)
Some(20)
Some(30)
None
```

All the other iterator methods in this lesson (`map`, `sum`, `filter`, …) are built on top of `next`.
You get them for free, as default methods of the trait (Lesson 05.2), as soon as a type implements `next`.

## 2. `for` loops use iterators

You have used `for item in &list` since Phase 02. Now you can see what it does. A `for` loop calls
`next` until it gets `None`:

```rust
fn main() {
    let names = vec!["Aziz", "Bobur"];

    for name in &names {
        println!("for: {name}");
    }

    let mut walker = names.iter();
    loop {
        match walker.next() {
            Some(name) => println!("loop: {name}"),
            None => break,
        }
    }
}
```

```text
for: Aziz
for: Bobur
loop: Aziz
loop: Bobur
```

**What happens:** both halves do exactly the same. The `for` loop is a short way to write the `loop` +
`match` + `next`. It calls `IntoIterator::into_iter` on the thing after `in`, to get an iterator.

### Three ways to iterate a collection

| Method | Items are | The collection afterwards |
|---|---|---|
| `.iter()` | shared references `&T` | unchanged, still yours |
| `.iter_mut()` | mutable references `&mut T` | still yours, but may have been changed |
| `.into_iter()` | owned values `T` | **moved**: gone (for a `Vec`) |

```rust
fn main() {
    let mut scores = vec![70, 85, 90];

    for s in scores.iter() {
        print!("{} ", s); // s is &i32
    }
    println!();

    for s in scores.iter_mut() {
        *s += 5; // s is &mut i32
    }
    println!("{:?}", scores);

    let total: i32 = scores.into_iter().sum(); // scores is moved here
    println!("{total}");
}
```

**What happens:**

1. `iter()` lets us read each score; the vector is untouched.
2. `iter_mut()` gives `&mut i32`, so `*s += 5` changes the numbers inside the vector (the `*` follows
   the reference, Lesson 03.5).
3. `into_iter()` consumes the vector: we get owned values and `sum` adds them. After this line
   `scores` cannot be used.

```text
70 85 90 
[75, 90, 95]
260
```

The `for x in vec` form (without `&`) is the third kind, and a classic mistake:

```rust
fn main() {
    let names = vec![String::from("Aziz"), String::from("Bobur")];
    for name in names {
        println!("{name}");
    }
    println!("{}", names.len());
}
```

```text
error[E0382]: borrow of moved value: `names`
 --> examples/into_iter_err.rs:6:20
  |
2 |     let names = vec![String::from("Aziz"), String::from("Bobur")];
  |         ----- move occurs because `names` has type `Vec<String>`, which does not implement the `Copy` trait
3 |     for name in names {
  |                 ----- `names` moved due to this implicit call to `.into_iter()`
...
6 |     println!("{}", names.len());
  |                    ^^^^^ value borrowed here after move
  |
help: consider iterating over a slice of the `Vec<String>`'s content to avoid moving into the `for` loop
  |
3 |     for name in &names {
  |                 +
```

**What it means:** `for name in names` calls `into_iter`, which takes the vector by value. Write
`for name in &names` to borrow it instead (the compiler suggests exactly this).

## 3. Iterators are lazy: adapters and consumers

Methods of `Iterator` come in two groups:

- **Adapters** turn one iterator into another: `map`, `filter`, `take`, `enumerate`, … They are
  **lazy**: calling them does no work yet; they only describe what *will* happen.
- **Consumers** pull items through the pipeline and produce a final value: `sum`, `count`, `collect`,
  `fold`, `for_each`, … Only a consumer makes anything run.

```rust
fn main() {
    let numbers = vec![1, 2, 3];

    let doubled = numbers.iter().map(|x| {
        println!("  doubling {x}");
        x * 2
    });
    println!("map created, nothing printed yet");

    let result: Vec<i32> = doubled.collect();
    println!("{:?}", result);
}
```

**What happens:**

1. `.map(…)` creates a new iterator that *will* double every item. The closure has not run yet.
2. The first `println!` proves it: it prints before any "doubling" line.
3. `.collect()` is a consumer. It asks for each item; for each one the closure runs and prints
   "doubling…".

```text
map created, nothing printed yet
  doubling 1
  doubling 2
  doubling 3
[2, 4, 6]
```

If you forget the consumer, the compiler warns you:

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    numbers.iter().map(|x| println!("{x}"));
}
```

```text
warning: unused `Map` that must be used
 --> examples/lazy_warn.rs:3:5
  |
3 |     numbers.iter().map(|x| println!("{x}"));
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: iterators are lazy and do nothing unless consumed
help: you might have meant to use `Iterator::for_each`
```

**What it means:** nothing was printed, because nothing consumed the iterator. To run a closure for its
side effect (such as printing), use a `for` loop or `for_each`.

> 💡 **Tip:** Laziness is useful. `numbers.iter().map(…).take(3)` computes only three items, even if
> `numbers` has a million. Infinite iterators (such as `(1..)`, "1, 2, 3, … forever") are safe as long as
> something like `take` stops them.

## 4. A tour of adapters

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4, 5, 6];

    let squares: Vec<i32> = numbers.iter().map(|x| x * x).collect();
    println!("map:        {:?}", squares);

    let evens: Vec<&i32> = numbers.iter().filter(|x| **x % 2 == 0).collect();
    println!("filter:     {:?}", evens);

    for (index, value) in numbers.iter().enumerate().take(2) {
        println!("enumerate:  {index} -> {value}");
    }

    let names = ["a", "b", "c"];
    let pairs: Vec<(&str, &i32)> = names.iter().copied().zip(numbers.iter()).collect();
    println!("zip:        {:?}", pairs);

    let middle: Vec<i32> = numbers.iter().copied().skip(1).take(3).collect();
    println!("skip/take:  {:?}", middle);

    let joined: Vec<i32> = numbers.iter().copied().take(2).chain(100..102).collect();
    println!("chain:      {:?}", joined);

    let backwards: Vec<i32> = numbers.iter().copied().rev().collect();
    println!("rev:        {:?}", backwards);

    let small: Vec<i32> = numbers.iter().copied().take_while(|x| *x < 4).collect();
    println!("take_while: {:?}", small);

    let every_other: Vec<i32> = (0..10).step_by(3).collect();
    println!("step_by:    {:?}", every_other);

    let nested = vec![vec![1, 2], vec![3], vec![]];
    let flat: Vec<i32> = nested.into_iter().flatten().collect();
    println!("flatten:    {:?}", flat);

    for pair in numbers.windows(2).take(2) {
        println!("windows:    {:?}", pair);
    }
    for chunk in numbers.chunks(4) {
        println!("chunks:     {:?}", chunk);
    }
}
```

```text
map:        [1, 4, 9, 16, 25, 36]
filter:     [2, 4, 6]
enumerate:  0 -> 1
enumerate:  1 -> 2
zip:        [("a", 1), ("b", 2), ("c", 3)]
skip/take:  [2, 3, 4]
chain:      [1, 2, 100, 101]
rev:        [6, 5, 4, 3, 2, 1]
take_while: [1, 2, 3]
step_by:    [0, 3, 6, 9]
flatten:    [1, 2, 3]
windows:    [1, 2]
windows:    [2, 3]
chunks:     [1, 2, 3, 4]
chunks:     [5, 6]
```

**What each adapter does:**

| Adapter | Meaning |
|---|---|
| `map(f)` | transform every item with `f` |
| `filter(p)` | keep only items for which the predicate `p` (a closure returning `bool`) is true |
| `enumerate()` | pair each item with its index: `(0, a), (1, b), …` |
| `zip(other)` | walk two iterators together, producing pairs; stops at the shorter one |
| `skip(n)` / `take(n)` | drop the first `n` items / stop after `n` items |
| `chain(other)` | all items of the first iterator, then all items of the second |
| `rev()` | go backwards (works on iterators that know their end, like those of vectors and ranges) |
| `take_while(p)` | take items until the first one for which `p` is false |
| `step_by(n)` | every `n`-th item |
| `flatten()` | turn an iterator of collections into one iterator of their items |
| `copied()` / `cloned()` | turn `&T` items into `T` items by copying / cloning (see Section 7) |

`windows(2)` and `chunks(4)` are methods of **slices**, not of `Iterator`, but they return iterators:
`windows(n)` gives overlapping groups of `n`, `chunks(n)` gives separate groups of up to `n`.

## 5. A tour of consumers

```rust
fn main() {
    let numbers = vec![3, 8, 1, 9, 4];

    let sum: i32 = numbers.iter().sum();
    let product: i64 = numbers.iter().map(|&x| x as i64).product();
    println!("sum={sum} product={product} count={}", numbers.iter().count());

    println!("min={:?} max={:?}", numbers.iter().min(), numbers.iter().max());
    println!("last={:?}", numbers.iter().last());

    let total = numbers.iter().fold(0, |acc, x| acc + x);
    println!("fold={total}");

    println!("any > 8: {}", numbers.iter().any(|&x| x > 8));
    println!("all > 0: {}", numbers.iter().all(|&x| x > 0));
    println!("find > 5: {:?}", numbers.iter().find(|&&x| x > 5));
    println!("position of 9: {:?}", numbers.iter().position(|&x| x == 9));

    let words = ["kiwi", "banana", "fig"];
    println!("longest: {:?}", words.iter().max_by_key(|w| w.len()));
    println!("shortest: {:?}", words.iter().min_by_key(|w| w.len()));

    let empty: Vec<i32> = Vec::new();
    println!("max of empty: {:?}", empty.iter().max());
    println!("sum of empty: {}", empty.iter().sum::<i32>());
}
```

```text
sum=25 product=864 count=5
min=Some(1) max=Some(9)
last=Some(4)
fold=25
any > 8: true
all > 0: true
find > 5: Some(8)
position of 9: Some(3)
longest: Some("banana")
shortest: Some("fig")
max of empty: None
sum of empty: 0
```

**What happens:**

1. `sum` and `product` need to know the result type: `let sum: i32` or `sum::<i32>()` (the turbofish
   from Lesson 05.1). We convert to `i64` for `product` with the cast `x as i64` (Phase 02) to be safer from overflow.
2. `min`, `max`, `last`, `find` and `position` return an `Option`, because the iterator may be empty
   or have no match. This is why `max` of an empty list is `None`, not a crash.
3. `fold(start, f)` is the general consumer. It keeps an accumulator: begin with `start`, and for each
   item set `acc = f(acc, item)`. `sum` is a `fold` with `+`.
4. `any` and `all` stop as soon as the answer is known (they are *short-circuiting*): `any` stops at the first `true`.
5. `max_by_key(|w| w.len())` picks the item with the largest key.

## 6. `collect`: gathering results

`collect` builds a collection from an iterator. Which one? **The type you ask for decides.**

```rust
use std::collections::HashMap;

fn main() {
    let letters: String = vec!['r', 'u', 's', 't'].into_iter().collect();
    println!("{letters}");

    let upper: Vec<String> = ["a", "b"].iter().map(|s| s.to_uppercase()).collect();
    println!("{:?}", upper);

    let keys = ["one", "two", "three"];
    let values = [1, 2, 3];
    let map: HashMap<&str, i32> = keys.iter().copied().zip(values.iter().copied()).collect();
    println!("two -> {:?}", map.get("two"));

    let good: Result<Vec<i32>, _> = "1 2 3".split(' ').map(|s| s.parse::<i32>()).collect();
    println!("{:?}", good);
    let bad: Result<Vec<i32>, _> = "1 x 3".split(' ').map(|s| s.parse::<i32>()).collect();
    println!("{:?}", bad);

    let (even, odd): (Vec<i32>, Vec<i32>) = (1..=6).partition(|n| n % 2 == 0);
    println!("{:?} {:?}", even, odd);
}
```

**What happens:**

1. Collecting `char`s into a `String` glues them together.
2. Collecting `(key, value)` pairs into a `HashMap` builds a map. Here `zip` makes the pairs.
3. Collecting an iterator of `Result<i32, _>` into `Result<Vec<i32>, _>` is a very useful trick: if **all**
   items are `Ok`, you get `Ok(vec)`; at the **first** `Err` it stops and returns that error. The same works with `Option`.
4. `partition(p)` splits one iterator into two collections: items where `p` is true and the rest.

```text
rust
["A", "B"]
two -> Some(2)
Ok([1, 2, 3])
Err(ParseIntError { kind: InvalidDigit })
[2, 4, 6] [1, 3, 5]
```

If you give no type, the compiler cannot guess which collection you want:

```rust
fn main() {
    let doubled = (1..4).map(|x| x * 2).collect();
    println!("{:?}", doubled);
}
```

```text
error[E0283]: type annotations needed
 --> examples/collect_err.rs:2:9
  |
2 |     let doubled = (1..4).map(|x| x * 2).collect();
  |         ^^^^^^^                         ------- type must be known at this point
  |
  = note: the type must implement `FromIterator<i32>`
help: consider giving `doubled` an explicit type
  |
2 |     let doubled: Vec<_> = (1..4).map(|x| x * 2).collect();
  |                ++++++++
```

**Fix:** `let doubled: Vec<_> = …` (the `_` means "work out the item type for me") or
`.collect::<Vec<_>>()`.

## 7. The reference trap: `&i32`, `&&i32` and friends

Because `iter()` produces **references**, closures often receive a reference, and `filter` gives its
closure a reference to the item, so you get a reference to a reference:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4];

    let a: Vec<&i32> = numbers.iter().filter(|x| **x > 1).collect();
    let b: Vec<i32> = numbers.iter().copied().filter(|x| *x > 1).collect();
    let c: Vec<i32> = numbers.iter().filter(|&&x| x > 1).cloned().collect();
    let d: Vec<i32> = numbers.clone().into_iter().filter(|x| *x > 1).collect();
    println!("{:?} {:?} {:?} {:?}", a, b, c, d);
}
```

```text
[2, 3, 4] [2, 3, 4] [2, 3, 4] [2, 3, 4]
```

**What happens:** four ways to say the same thing.

- `a`: `iter()` items are `&i32`; `filter` passes `&&i32` to the closure, so `**x` reaches the number.
  The result keeps references (`Vec<&i32>`).
- `b`: `.copied()` turns `&i32` into `i32` first. Now `filter` receives `&i32`, and `*x` is enough.
  `copied` works for `Copy` types; `cloned()` works for any `Clone` type (it clones each item).
- `c`: the pattern `|&&x|` takes the references apart in the parameter itself (patterns, Lesson 04.5).
- `d`: `into_iter()` on an owned vector yields `i32` values (we cloned the vector first so we keep the original).

Forgetting the reference gives this error:

```rust
fn main() {
    let numbers = vec![1, 2, 3, 4];
    let big: Vec<&i32> = numbers.iter().filter(|x| x > 1).collect();
    println!("{:?}", big);
}
```

```text
error[E0308]: mismatched types
 --> examples/refs_err.rs:3:56
  |
3 |     let big: Vec<&i32> = numbers.iter().filter(|x| x > 1).collect();
  |                                                        ^ expected `&&_`, found integer
  |
  = note: expected reference `&&_`
                  found type `{integer}`
```

**What it means:** `x` is a `&&i32`, and you compare it with the number `1`. Dereference with `**x` or use
`.copied()` first. The trick for beginners: **when you get confused, add `.copied()` right after
`.iter()`** (for numbers and other `Copy` types), and the references disappear.

The last common error is calling `next` without `mut`:

```text
error[E0596]: cannot borrow `walker` as mutable, as it is not declared as mutable
 --> examples/mut_err.rs:4:22
  |
4 |     println!("{:?}", walker.next());
  |                      ^^^^^^ cannot borrow as mutable
  |
help: consider changing this to be mutable
  |
3 |     let mut walker = numbers.iter();
  |         +++
```

## 8. Writing your own iterator

To make your own type iterable, implement `Iterator`: choose `type Item` and write `next`.

```rust
struct Countdown {
    remaining: u32,
}

impl Iterator for Countdown {
    type Item = u32;

    fn next(&mut self) -> Option<u32> {
        if self.remaining == 0 {
            None
        } else {
            let current = self.remaining;
            self.remaining -= 1;
            Some(current)
        }
    }
}

fn evens_up_to(limit: u32) -> impl Iterator<Item = u32> {
    (0..=limit).filter(|n| n % 2 == 0)
}

fn main() {
    for n in (Countdown { remaining: 3 }) {
        println!("{n}...");
    }
    println!("liftoff");

    let total: u32 = Countdown { remaining: 4 }.sum();
    println!("sum = {total}");

    let doubled: Vec<u32> = Countdown { remaining: 3 }.map(|n| n * 2).collect();
    println!("{:?}", doubled);

    println!("{:?}", evens_up_to(9).collect::<Vec<_>>());
}
```

**What happens:**

1. `Countdown` stores its state in `remaining`. Each `next` returns the current value and moves one
   step closer to the end; at zero it returns `None`.
2. We only wrote `next`. Because `Countdown` implements `Iterator`, it can be used in a `for` loop, and `sum`,
   `map`, `collect` and every other method work for free. (The parentheses around the struct literal in
   `for n in (Countdown { … })` are needed because a `{` right after `in` would be read as the start of the loop body.)
3. `evens_up_to` returns `impl Iterator<Item = u32>` (Lesson 05.2): "some iterator that yields `u32`".
   The caller does not need to know it is a `Filter<RangeInclusive<u32>, …>`.

```text
3...
2...
1...
liftoff
sum = 10
[6, 4, 2]
[0, 2, 4, 6, 8]
```

> 📏 **Convention:** Functions that take "something to loop over" often use `impl IntoIterator<Item = T>`
> as the parameter type, so callers may pass a vector, a slice or an iterator. You will see it in
> libraries. For your own simple functions, `&[T]` is perfectly fine.

## 9. Iterators or loops?

Both are correct. Compare the same task written two ways: sum the numbers on the valid lines of a text.

```rust
fn main() {
    let text = "10\nabc\n32\n\n8";

    let mut total_loop = 0;
    for line in text.lines() {
        if let Ok(n) = line.parse::<i32>() {
            total_loop += n;
        }
    }

    let total_iter: i32 = text.lines().filter_map(|l| l.parse::<i32>().ok()).sum();
    println!("{total_loop} {total_iter}");

    let first_bad = text.lines().find(|l| l.parse::<i32>().is_err());
    println!("{:?}", first_bad);
}
```

```text
50 50
Some("abc")
```

**What happens:** `filter_map(f)` is `map` and `filter` in one step: the closure returns an `Option`, `Some(x)`
is kept as `x` and `None` is dropped. `.ok()` converts a `Result` into an `Option`. The iterator version
says *what* we want (the sum of the numbers that parse); the loop version says *how*.

> ⚡ **Performance:** Iterator pipelines are **zero-cost abstractions** (Lesson 05.1): the compiler
> generates the same machine code as the hand-written loop, often better, because it knows
> no index can go out of bounds. You do not pay for the clean style. Measure with a benchmark before
> believing any claim to the contrary (Lesson 05.12).

When to prefer which:

- Use **iterator pipelines** for "transform / filter / reduce a sequence" tasks.
- Use a **`for` loop** when you need `break`/`continue` with complicated conditions, several mutable variables, or early returns in the middle of the body. A clear loop beats a clever pipeline.

A last example that uses many pieces together, the word frequency count:

```rust
use std::collections::HashMap;

fn main() {
    let text = "the cat and the dog and the bird";
    let mut counts: HashMap<&str, usize> = HashMap::new();
    for word in text.split_whitespace() {
        *counts.entry(word).or_insert(0) += 1;
    }

    let mut pairs: Vec<(&str, usize)> = counts.into_iter().collect();
    pairs.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(b.0)));
    for (word, n) in pairs.iter().take(3) {
        println!("{word}: {n}");
    }
}
```

```text
the: 3
and: 2
bird: 1
```

**What happens:** a `HashMap` has no fixed order, so we `collect` its pairs into a `Vec`, then sort with
a closure: larger counts first (`b.1.cmp(&a.1)`), ties alphabetically (`.then(a.0.cmp(b.0))`, Lesson 05.4).
Then `take(3)` keeps the top three. Without the sort, the output order would change from run to run.

> 💡 **Tip:** The crate `itertools` (version 0.15) adds dozens of extra adapters such as `join`, `sorted`,
> `dedup` and `group_by`. Lesson 05.8 teaches how to add a crate to a project.

## Common mistakes

**1. Forgetting that adapters are lazy** (warning `unused Map that must be used`, Section 3). Add a consumer, or use `for`/`for_each`.

**2. Using `for x in vec` and then the vector** (`E0382`, Section 2). Loop over `&vec`.

**3. Forgetting the type for `collect`** (`E0283`, Section 6). Write `let v: Vec<_> = …`.

**4. Comparing a reference with a value inside `filter`** (`E0308`, Section 7). Use `**x`, `|&&x|`, or `.copied()`.

**5. Calling `next` on an iterator not declared `mut`** (`E0596`, Section 7).

**6. A chain that is too clever.** If a pipeline needs more than about five steps or a comment to explain it, split it into named
variables or use a loop. Readability is more important than cleverness.

## Try it yourself

This time the Git practice is a **refactoring** (changing how code is written without changing what it
does), and fixing the last commit with `--amend`.

1. Start the lesson branch:

   ```bash
   cd ~/projects/rust
   git switch main
   git switch -c lesson/05-05-iterators
   ```

2. Create `code/p05-lessons/examples/iter_sum_loop.rs` with a loop-based function and a test, and commit:

   ```rust
   fn sum_even_squares(limit: u32) -> u32 {
       let mut total = 0;
       for n in 1..=limit {
           if n % 2 == 0 {
               total += n * n;
           }
       }
       total
   }

   fn main() {
       println!("{}", sum_even_squares(10));
   }

   #[cfg(test)]
   mod tests {
       use super::*;

       #[test]
       fn sums_squares_of_even_numbers() {
           assert_eq!(sum_even_squares(10), 220);
       }
   }
   ```

   ```bash
   cd code/p05-lessons
   cargo test --example iter_sum_loop
   cd ~/projects/rust
   git add .
   git commit -m "feat(p05): sum of even squares with a loop"
   ```

3. **Refactor** the function body into the pipeline `(1..=limit).filter(|n| n % 2 == 0).map(|n| n * n).sum()`.
   Run the same test again. It must still pass: the test is your safety net that the behaviour did not change.

   ```bash
   cd code/p05-lessons
   cargo test --example iter_sum_loop
   cd ~/projects/rust
   git commit -am "refactor(p05): use an iterator pipeline"
   ```

   Rule: a `refactor:` commit never changes behaviour, and the tests pass **before and after** it. If a
   test had to change, it is not a refactoring.

4. You notice that the first commit's message is imprecise, or you forgot a file. **`git commit --amend`**
   replaces the **last** commit (new message, plus whatever you staged now):

   ```bash
   git commit --amend -m "refactor(p05): sum of even squares as an iterator pipeline"
   git log --oneline
   ```

   Look at the hash: it changed (for example from `b356577` to `b83f345`), because Git made a *new* commit
   that replaces the old one. That is why you must **never amend a commit that you have already pushed**
   (Lesson 01.8) and others may have pulled: it rewrites history that other people depend on. For local,
   unpublished commits it is safe and common.

5. Finish: merge into `main` as in the previous lessons.

## Exercises

### Exercise 1 — Sum of even squares (easy)

Write `fn sum_even_squares(limit: u32) -> u32` as one iterator pipeline: the sum of the squares of the
even numbers from 1 to `limit`. Test `limit = 10` (answer 220) and `limit = 0`.

<details>
<summary>Hint</summary>

`(1..=limit)` is a range iterator. Chain `.filter(|n| n % 2 == 0)`, `.map(|n| n * n)` and the consumer `.sum()`.
`sum` needs to know the type: it is the function's return type `u32`, so no turbofish is needed.

</details>

<details>
<summary>Answer</summary>

```rust
fn sum_even_squares(limit: u32) -> u32 {
    (1..=limit).filter(|n| n % 2 == 0).map(|n| n * n).sum()
}

fn main() {
    println!("{}", sum_even_squares(10));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sums_squares_of_even_numbers() {
        assert_eq!(sum_even_squares(10), 4 + 16 + 36 + 64 + 100);
    }

    #[test]
    fn limit_zero_gives_zero() {
        assert_eq!(sum_even_squares(0), 0);
    }
}
```

```text
220
```

Why it works: `1..=0` is an empty range, so `sum` returns 0. The `n` in `filter` is a `&u32`, and `n % 2`
works on references too, because the arithmetic operators are implemented for them.

</details>

### Exercise 2 — Count long words (easy)

Write `fn count_long_words(text: &str, min_len: usize) -> usize` that counts the words with more than `min_len`
letters. Test with `"the quick brown fox jumps"` and `min_len = 3` (answer 3), and with an empty text.

<details>
<summary>Hint</summary>

`text.split_whitespace()` gives an iterator over the words. Use `filter` with `w.len() > min_len`, and `count()`.

</details>

<details>
<summary>Answer</summary>

```rust
fn count_long_words(text: &str, min_len: usize) -> usize {
    text.split_whitespace().filter(|w| w.len() > min_len).count()
}

fn main() {
    println!("{}", count_long_words("the quick brown fox jumps", 3));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_words_longer_than_the_limit() {
        assert_eq!(count_long_words("the quick brown fox jumps", 3), 3);
    }

    #[test]
    fn empty_text_has_no_words() {
        assert_eq!(count_long_words("", 3), 0);
    }
}
```

```text
3
```

Why it works: "quick", "brown" and "jumps" have five letters; "the" and "fox" have three. Note that `len()` counts
bytes, which equals letters for English text (Lesson 04.8 explains the difference).

</details>

### Exercise 3 — A Fibonacci iterator (medium)

Implement `struct Fibonacci` that implements `Iterator<Item = u64>` and produces 0, 1, 1, 2, 3, 5, … It must
**stop (return `None`) instead of overflowing** when the next number would not fit into a `u64`.
Test: the first ten numbers; the sum of the even ones among them; and that the iterator ends.

<details>
<summary>Hint</summary>

Keep two fields, `current` and `next`. In `next()` remember `current` as the result, compute the new
`next` with `self.current.checked_add(self.next)?` (`checked_add` returns `None` on overflow, and `?` returns that `None`
from your function), then shift the two fields.

</details>

<details>
<summary>Answer</summary>

```rust
struct Fibonacci {
    current: u64,
    next: u64,
}

impl Fibonacci {
    fn new() -> Self {
        Fibonacci { current: 0, next: 1 }
    }
}

impl Iterator for Fibonacci {
    type Item = u64;

    fn next(&mut self) -> Option<u64> {
        let result = self.current;
        let new_next = self.current.checked_add(self.next)?;
        self.current = self.next;
        self.next = new_next;
        Some(result)
    }
}

fn main() {
    let first: Vec<u64> = Fibonacci::new().take(10).collect();
    println!("{:?}", first);
    println!("count until overflow: {}", Fibonacci::new().count());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn first_ten_numbers() {
        let v: Vec<u64> = Fibonacci::new().take(10).collect();
        assert_eq!(v, vec![0, 1, 1, 2, 3, 5, 8, 13, 21, 34]);
    }

    #[test]
    fn works_with_adapters() {
        let even_sum: u64 = Fibonacci::new().take(10).filter(|n| n % 2 == 0).sum();
        assert_eq!(even_sum, 2 + 8 + 34);
    }

    #[test]
    fn stops_instead_of_overflowing() {
        assert!(Fibonacci::new().count() < 100);
    }
}
```

```text
[0, 1, 1, 2, 3, 5, 8, 13, 21, 34]
count until overflow: 92
```

Why it works: this simple version stops a little early. It returns `None` when it cannot compute the number two
steps ahead, so it yields 92 numbers although a couple more would still fit. That is acceptable here, and it never overflows.
🔒 **Security:** in many languages integer overflow silently produces wrong numbers; `checked_add` makes the edge explicit
(integer overflow was explained in Phase 02).

</details>

### Exercise 4 — Parse `key=value` lines into a map (medium)

Write `fn parse_pairs(text: &str) -> HashMap<&str, &str>` that reads one `key = value` pair per line, trims spaces,
skips lines without `=`, and collects the result **with one iterator pipeline** (no `for` loop). Test: valid
and broken lines; duplicate keys (the later line wins).

<details>
<summary>Hint</summary>

`text.lines().filter_map(|line| line.split_once('='))` gives `(&str, &str)` pairs and drops lines without `=`.
Then `.map(|(k, v)| (k.trim(), v.trim()))` and `.collect()` into the `HashMap` that the return type names.
For duplicate keys, `collect` into a map inserts items in order, so the last one wins.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashMap;

fn parse_pairs(text: &str) -> HashMap<&str, &str> {
    text.lines()
        .filter_map(|line| line.split_once('='))
        .map(|(k, v)| (k.trim(), v.trim()))
        .collect()
}

fn main() {
    let pairs = parse_pairs("host = example.org\nbroken\nport=8080");
    let mut keys: Vec<_> = pairs.keys().collect();
    keys.sort();
    println!("{:?}", keys);
    println!("{:?}", pairs.get("port"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_valid_lines_and_skips_broken_ones() {
        let map = parse_pairs("host = a\nbroken\nport=1");
        assert_eq!(map.len(), 2);
        assert_eq!(map.get("host"), Some(&"a"));
        assert_eq!(map.get("port"), Some(&"1"));
    }

    #[test]
    fn later_line_wins_for_duplicate_keys() {
        let map = parse_pairs("a=1\na=2");
        assert_eq!(map.get("a"), Some(&"2"));
    }
}
```

```text
["host", "port"]
Some("8080")
```

Why it works: the keys and values are slices of `text` (lifetime elision, Lesson 05.3), so nothing is copied. We sort the
keys before printing because a `HashMap` has no fixed order.

</details>

### Exercise 5 — The most common words (hard)

Write `fn top_words(text: &str, n: usize) -> Vec<(String, usize)>` returning the `n` most frequent words, case-insensitive
(`"The"` and `"the"` are the same word), most frequent first, ties in alphabetical order. Test: counting and order; ties;
fewer words than `n`; empty text.

<details>
<summary>Hint</summary>

Count in a `HashMap<String, usize>` using `word.to_lowercase()` as the key. Collect `counts.into_iter()` into a `Vec`, sort with
`sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)))`, and finish with `.into_iter().take(n).collect()`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashMap;

fn top_words(text: &str, n: usize) -> Vec<(String, usize)> {
    let mut counts: HashMap<String, usize> = HashMap::new();
    for word in text.split_whitespace() {
        *counts.entry(word.to_lowercase()).or_insert(0) += 1;
    }
    let mut pairs: Vec<(String, usize)> = counts.into_iter().collect();
    pairs.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)));
    pairs.into_iter().take(n).collect()
}

fn main() {
    for (word, count) in top_words("The cat and the dog and THE bird", 3) {
        println!("{word}: {count}");
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_ignoring_case_and_sorts_by_count() {
        let top = top_words("The cat and the dog and THE bird", 2);
        assert_eq!(top, vec![("the".to_string(), 3), ("and".to_string(), 2)]);
    }

    #[test]
    fn ties_are_sorted_alphabetically() {
        let top = top_words("b a c", 3);
        assert_eq!(top[0].0, "a");
        assert_eq!(top[2].0, "c");
    }

    #[test]
    fn fewer_words_than_n_is_fine() {
        assert_eq!(top_words("one", 5).len(), 1);
        assert!(top_words("", 5).is_empty());
    }
}
```

```text
the: 3
and: 2
bird: 1
```

Why it works: the sorting makes the result deterministic (the same every run), which is what makes it testable. `take(n)`
simply stops early when there are fewer than `n` pairs. Punctuation is not handled here: `"dog,"` and `"dog"` would be
different words. Improving that is a good stretch task.

</details>

## Quiz

1. What is the one method you must write to implement `Iterator`, and what does it return?

<details>
<summary>Answer</summary>

`next(&mut self) -> Option<Self::Item>`. It returns `Some(item)` for the next item and `None` when the iterator is finished. You also name the `Item` type.

</details>

2. What is the difference between `iter()`, `iter_mut()` and `into_iter()` on a `Vec<String>`?

<details>
<summary>Answer</summary>

`iter()` gives `&String` (read-only borrow), `iter_mut()` gives `&mut String` (can modify), `into_iter()` gives owned `String` values and consumes the vector.

</details>

3. What does it mean that adapters are lazy? What makes the pipeline run?

<details>
<summary>Answer</summary>

Adapters such as `map` and `filter` only build a new iterator and do no work. The pipeline runs when a consumer (`collect`, `sum`, `count`, `for_each`, a `for` loop, …) asks for the items.

</details>

4. How do you turn `Vec<Result<i32, E>>`-like data (an iterator of `Result`s) into `Result<Vec<i32>, E>`?

<details>
<summary>Answer</summary>

Call `collect()` into the type `Result<Vec<i32>, E>`. It returns `Ok(vec)` if every item is `Ok`, otherwise the first `Err`.

</details>

5. Why does `numbers.iter().filter(|x| x > 1)` not compile?

<details>
<summary>Answer</summary>

`iter()` yields `&i32`, and `filter` passes a reference to that item, so `x` is `&&i32`, which cannot be compared with the integer `1`. Use `**x > 1`, `|&&x| x > 1`, or `.copied()` before `filter`.

</details>

6. Are iterator pipelines slower than hand-written loops?

<details>
<summary>Answer</summary>

Normally no. They are zero-cost abstractions: the compiler produces equivalent machine code. Measure with a benchmark if performance really matters (Lesson 05.12).

</details>

## Summary

- An iterator produces items one by one through `next`, returning `Some(item)` and finally `None`.
- A `for` loop is built on iterators. `iter()` borrows items, `iter_mut()` borrows them mutably, `into_iter()` takes ownership.
- Adapters (`map`, `filter`, `enumerate`, `zip`, `take`, `skip`, `chain`, `rev`, …) are lazy; consumers (`sum`, `count`, `fold`, `collect`, `find`, `any`, …) run the pipeline.
- `collect` builds a `Vec`, `String`, `HashMap` or even `Result<Vec<_>, _>`; the target type decides, so annotate it.
- Items from `iter()` are references; use `.copied()` / `.cloned()` or patterns like `|&&x|` to deal with them.
- You can implement `Iterator` for your own types by writing `next`; all the adapters then work for free.
- Pipelines are as fast as loops; choose the clearer one. Refactor with tests green before and after, and `git commit --amend` only unpublished commits.

## Sources and further reading

- [The Rust Programming Language — 13.2 Processing a Series of Items with Iterators](https://doc.rust-lang.org/book/ch13-02-iterators.html) — the official chapter.
- [std::iter::Iterator](https://doc.rust-lang.org/std/iter/trait.Iterator.html) — the full list of adapters and consumers, with examples.
- [std::iter (module overview)](https://doc.rust-lang.org/std/iter/index.html) — how iterators, `IntoIterator` and `FromIterator` fit together.
- [Rust by Example — Iterators](https://doc.rust-lang.org/rust-by-example/trait/iter.html) — short examples.
- [itertools on docs.rs](https://docs.rs/itertools) — extra iterator methods (the crate is explained in Lesson 05.8).
- [Git documentation — git commit (--amend)](https://git-scm.com/docs/git-commit) — rewriting the last commit.

---

[🏠 Start](../START-HERE.md) · [← 05.4 Closures](04-closures.md) · [📋 Phase 05](README.md) · [05.6 Testing in depth →](06-testing-in-depth.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
