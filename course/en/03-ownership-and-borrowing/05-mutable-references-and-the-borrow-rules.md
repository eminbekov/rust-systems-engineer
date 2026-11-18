<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 03.4 References and borrowing](04-references-and-borrowing.md) · [📋 Phase 03](README.md) · [03.6 Slices and string types →](06-slices-and-string-types.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md) <!-- nav -->

# Lesson 03.5 — Mutable references and the borrowing rules

> **Phase 03 — Ownership and borrowing** · Lesson 5 of 8 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How to change a value through a reference with `&mut`.
- The **borrowing rules**: many readers *or* one writer — never both at once.
- What the errors `E0499`, `E0502` and `E0506` mean, and how to fix them.
- Why these rules prevent real bugs (changing a list while looping over it, data races).
- How to use `*` with `&mut`, and the tools `std::mem::swap`, `replace` and `take`.

## Before you start

- You finished [Lesson 03.4 — References and borrowing](04-references-and-borrowing.md). You know `&T`, borrowing, and that a shared reference cannot change anything.
- Create a project: `cargo new code/p03-mut-refs` (from `~/projects/rust`).

## New words

| Word | Meaning |
|---|---|
| **mutable reference** | A reference `&mut T` that may **change** the value it points to. At most one can exist at a time. |
| **exclusive** | Held by one party only. A `&mut` is also called an *exclusive* reference: nobody else can read or write the value while it exists. |
| **aliasing** | Having two or more names (references) for the same value at the same time. |
| **borrow checker** | The part of the compiler that checks the borrowing rules. |
| **data race** | A bug where two parts of a program use the same data at the same time, at least one of them writes, and nothing coordinates them. The result is unpredictable. |
| **iterator invalidation** | The bug of changing a collection while you are looping over it, so the loop reads memory that has moved. |
| **reborrow** | Making a new, short reference from an existing `&mut`, so that the original can be used again afterwards. Rust does it for you in function calls. |

## 1. `&mut`: borrowing with permission to change

Last lesson ended with an error: you cannot change a value through `&`. The fix is a **mutable reference**. There are three small changes compared with a shared reference:

1. The variable must be declared with `mut`.
2. You create the reference with `&mut`.
3. The parameter type is written `&mut T`.

```rust
fn add_world(s: &mut String) {
    s.push_str(", world");
}

fn main() {
    let mut s = String::from("hello");
    add_world(&mut s);
    println!("{s}");
}
```

```text
hello, world
```

**What happens:**

1. `let mut s` creates a `String` that is allowed to change.
2. `&mut s` lends `s` to the function, with permission to modify it.
3. Inside, `s.push_str(...)` appends to the original string, not to a copy. (This may reallocate the heap buffer — Lesson 03.1.)
4. The function ends, the borrow ends, and `main` owns and prints the changed string. No clone, no moving back and forth.

If you forget `mut` on the variable, the compiler tells you:

```rust
fn add_world(s: &mut String) {
    s.push_str(", world");
}

fn main() {
    let s = String::from("hello");
    add_world(&mut s);
    println!("{s}");
}
```

```text
error[E0596]: cannot borrow `s` as mutable, as it is not declared as mutable
 --> src/main.rs:7:15
  |
7 |     add_world(&mut s);
  |               ^^^^^^ cannot borrow as mutable
  |
help: consider changing this to be mutable
  |
6 |     let mut s = String::from("hello");
  |         +++
```

The rule makes sense: you cannot lend out permission to change something that you yourself are not allowed to change.

## 2. The borrowing rules

Here are the two rules that the **borrow checker** enforces. Learn them by heart:

> **At any moment, you can have either:**
> - **any number of shared references (`&T`)**, *or*
> - **exactly one mutable reference (`&mut T`)**,
>
> **but never both at the same time.** And all references must always be valid.

Think of a shared document. Many people can **read** it at the same time. Or a single person can **edit** it. But if someone edits while others read, the readers see half-changed text and get confused. So the rule is: many readers, or one editor.

```text
        allowed                                not allowed

  owner ◄── &T                           owner ◄── &T
        ◄── &T                                 ◄── &mut T     (reader + writer)
        ◄── &T       (many readers)
                                         owner ◄── &mut T
  owner ◄── &mut T   (one writer)              ◄── &mut T     (two writers)
```

Let us see the three kinds of conflict.

### Two mutable references: `E0499`

```rust
fn main() {
    let mut s = String::from("hello");
    let r1 = &mut s;
    let r2 = &mut s;
    r1.push_str(" one");
    r2.push_str(" two");
    println!("{s}");
}
```

```text
error[E0499]: cannot borrow `s` as mutable more than once at a time
 --> src/main.rs:4:14
  |
3 |     let r1 = &mut s;
  |              ------ first mutable borrow occurs here
4 |     let r2 = &mut s;
  |              ^^^^^^ second mutable borrow occurs here
5 |     r1.push_str(" one");
  |     -- first borrow later used here
```

Read it like a story: `r1` borrows `s` mutably; before `r1` is finished (it is used again on line 5), `r2` tries to borrow `s` mutably too. Two editors.

### A mutable reference while a shared one is alive: `E0502`

```rust
fn main() {
    let mut s = String::from("hello");
    let reader = &s;
    let writer = &mut s;
    writer.push_str(", world");
    println!("{reader}");
}
```

```text
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
 --> src/main.rs:4:18
  |
3 |     let reader = &s;
  |                  -- immutable borrow occurs here
4 |     let writer = &mut s;
  |                  ^^^^^^ mutable borrow occurs here
5 |     writer.push_str(", world");
6 |     println!("{reader}");
  |                ------ immutable borrow later used here
```

(*Immutable* is another word for *shared*.) The reader expects the text to stay the same, but the writer changes it.

### Assigning to a value that is borrowed: `E0506`

The owner is also bound by the rules. While someone borrows a value, even the owner may not change it:

```rust
fn main() {
    let mut x = 10;
    let r = &x;
    x = 20;
    println!("{r} {x}");
}
```

```text
error[E0506]: cannot assign to `x` because it is borrowed
 --> src/main.rs:4:5
  |
3 |     let r = &x;
  |             -- `x` is borrowed here
4 |     x = 20;
  |     ^^^^^^ `x` is assigned to here but it was already borrowed
5 |     println!("{r} {x}");
  |                - borrow later used here
```

## 3. Why these rules exist

It looks strict. What bad things do they prevent?

### Changing a list while you loop over it

This is a classic bug in many languages. Rust refuses it at compile time:

```rust
fn main() {
    let mut numbers = vec![1, 2, 3];
    for n in &numbers {
        if *n == 2 {
            numbers.push(10);
        }
    }
    println!("{:?}", numbers);
}
```

```text
error[E0502]: cannot borrow `numbers` as mutable because it is also borrowed as immutable
 --> src/main.rs:5:13
  |
3 |       for n in &numbers {
  |       -        --------
  |       |        |
  |       |        immutable borrow occurs here
  |  _____|        immutable borrow later used here
  | |
4 | |         if *n == 2 {
5 | |             numbers.push(10);
  | |             ^^^^^^^^^^^^^^^^ mutable borrow occurs here
6 | |         }
7 | |     }
  | |_____- this for loop borrows `numbers` immutably, preventing mutation within its body
  |
  = help: consider using an index-based loop instead, or collecting modifications into a separate collection
```

Why is this dangerous? The loop is a *reader* of the vector's buffer on the heap. A `push` may run out of capacity and move the whole buffer to a bigger place on the heap (Lesson 03.1: capacity doubling), freeing the old one. The loop would then read freed memory — **iterator invalidation**. The compiler's help line gives the standard fixes. One of them, "collect modifications into a separate collection", looks like this:

```rust
fn main() {
    let mut numbers = vec![1, 2, 3];
    let mut to_add = Vec::new();
    for n in &numbers {
        if *n == 2 {
            to_add.push(10);
        }
    }
    for n in to_add {
        numbers.push(n);
    }
    println!("{:?}", numbers);
}
```

```text
[1, 2, 3, 10]
```

### A reference that points into a buffer that may move

The same idea, even without a loop:

```rust
fn main() {
    let mut data = vec![1, 2, 3];
    let first = &data[0];
    data.push(4);
    println!("{first}");
}
```

```text
error[E0502]: cannot borrow `data` as mutable because it is also borrowed as immutable
 --> src/main.rs:4:5
  |
3 |     let first = &data[0];
  |                  ---- immutable borrow occurs here
4 |     data.push(4);
  |     ^^^^^^^^^^^^ mutable borrow occurs here
5 |     println!("{first}");
  |                ----- immutable borrow later used here
```

`first` is an address inside the vector's heap buffer. After `push(4)` the buffer may have moved, and `first` would be a **dangling pointer**. In a language without these checks, code like this compiles and then sometimes crashes or prints garbage. In Rust it is a compile error.

### Data races

A **data race** happens when two parts of a program (usually two *threads*, Phase 14) use the same data at the same time, at least one of them writes, and nothing coordinates them. The result depends on tiny timing differences and is nearly impossible to debug. The rule "one writer or many readers" makes data races impossible in safe Rust: if there is a writer, there are no other users.

> 🔒 **Security:** memory bugs like iterator invalidation and data races are exploited by attackers in many real systems. The borrowing rules remove them at compile time, with no runtime cost.

> ⚡ **Performance:** the borrow checker works only while compiling. It adds **no** instructions to your program, no locks and no checks at run time. The rules are strict, but the safety they give costs nothing while the program runs.

## 4. A borrow lasts until its last use

The rules apply to references that are **alive**. A reference is alive from its creation to its **last use** (the rule called NLL, Non-Lexical Lifetimes, from Lesson 03.4). So this is allowed:

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s;
    let r2 = &s;
    println!("{r1} and {r2}"); // last use of r1 and r2

    let r3 = &mut s; // fine: r1 and r2 are no longer used
    r3.push_str(", world");
    println!("{r3}");
}
```

```text
hello and hello
hello, world
```

The readers finished (their last use is the first `println!`) before the writer started. You can also end a borrow on purpose with a block:

```rust
fn main() {
    let mut s = String::from("hello");
    {
        let r1 = &mut s;
        r1.push('!');
    } // r1 ends here
    let r2 = &mut s;
    r2.push('?');
    println!("{s}");
}
```

```text
hello!?
```

> 💡 **Tip:** when the borrow checker complains, ask: "Where does the first borrow *start* and where is its *last use*? Can I move the second action after that last use?" Very often the answer is just a reordering of two lines.

## 5. Using `*` with `&mut`

With a `&mut i32` you cannot write `n += 1`. You must say "the number that `n` points to":

```rust
fn add_one(n: &mut i32) {
    n += 1;
}

fn main() {
    let mut x = 1;
    add_one(&mut x);
    println!("{x}");
}
```

```text
error[E0368]: binary assignment operation `+=` cannot be applied to type `&mut i32`
 --> src/main.rs:2:5
  |
2 |     n += 1;
  |     -^^^^^
  |     |
  |     cannot use `+=` on type `&mut i32`
  |
help: `+=` can be used on `i32` if you dereference the left-hand side
  |
2 |     *n += 1;
  |     +
```

The fix is the star:

```rust
fn add_one(n: &mut i32) {
    *n += 1;
}

fn main() {
    let mut x = 1;
    add_one(&mut x);
    add_one(&mut x);
    println!("{x}");
}
```

```text
3
```

For methods such as `s.push_str(...)` you do not need a star: Rust dereferences automatically.

**Reborrowing.** If `r` is a `&mut String`, you can pass `r` to a function many times in a row. It looks like the `&mut` should move into the first call (a `&mut` is not `Copy`), but Rust automatically makes a short *reborrow* for the call, and gives `r` back afterwards:

```rust
fn shout(s: &mut String) {
    s.push('!');
}

fn main() {
    let mut text = String::from("hey");
    let r = &mut text;
    shout(r);
    shout(r);
    shout(r);
    println!("{r}");
    println!("{text}");
}
```

```text
hey!!!
hey!!!
```

**What happens:** `r` is used three times; each call borrows from `r` for a moment. After the last use of `r` (the `println!("{r}")`), `text` can be used again.

## 6. Swapping and replacing: `std::mem`

Sometimes you hold a `&mut T` and want to put a new value in it, or take the old value out. You cannot *move* the value out of a reference (the owner would be left with a hole), but the standard library has three helpers in `std::mem` that do the job safely, by always leaving a valid value behind:

```rust
use std::mem;

fn main() {
    let mut a = String::from("left");
    let mut b = String::from("right");
    mem::swap(&mut a, &mut b);
    println!("a = {a}, b = {b}");

    let old = mem::replace(&mut a, String::from("new"));
    println!("old = {old}, a = {a}");

    let taken = mem::take(&mut b);
    println!("taken = {taken}, b = '{b}'");
}
```

**What happens:**

1. `mem::swap(&mut a, &mut b)` exchanges the two values. Two `&mut` at once is fine here, because they point to two *different* variables.
2. `mem::replace(&mut a, new)` puts `new` into `a` and returns the old value to you.
3. `mem::take(&mut b)` takes the value out and leaves the "default" in its place (for a `String`: the empty string).

```text
a = right, b = left
old = right, a = new
taken = left, b = ''
```

## 7. Change in place, or return a new value?

You now have two styles for a function that changes data:

```rust
// style A: change in place (needs &mut, the caller keeps the same value)
fn double_all(numbers: &mut Vec<i32>) {
    for i in 0..numbers.len() {
        numbers[i] *= 2;
    }
}
```

```rust
// style B: build and return a new value (takes & or ownership, returns a new Vec)
// fn doubled(numbers: &Vec<i32>) -> Vec<i32> { ... }
```

Style A, with `&mut`, avoids allocating a new vector. Style B is easier to test and reason about, because the input does not change. Choose A when the data is large or when "update this thing" is the natural meaning, and B when you want a pure computation. Here is A in action:

```rust
fn double_all(numbers: &mut Vec<i32>) {
    for i in 0..numbers.len() {
        numbers[i] *= 2;
    }
}

fn main() {
    let mut numbers = vec![1, 2, 3];
    double_all(&mut numbers);
    println!("{:?}", numbers);
}
```

```text
[2, 4, 6]
```

(The loop uses indexes instead of `for n in numbers`, because a `for` over the vector would hold a borrow of it for the whole loop; Lesson 05.5 shows `iter_mut`, a neat way to change items during iteration.)

> 📏 **Convention:** a function that needs to change its argument takes `&mut T`, and the call site shows it with `&mut x`. When you read `foo(&mut x)` you know at once that `foo` may change `x`. Keep functions small, and keep the part of the code that holds a `&mut` as short as possible.

## Common mistakes

**1. Forgetting `mut` on the variable** (`E0596`). Write `let mut s`.

**2. Holding two `&mut` to the same variable** (`E0499`). Finish using the first before creating the second, or restructure.

**3. Reading while a `&mut` is alive** (`E0502`). Use the data through the `&mut` itself, or end its life first.

**4. Changing a vector while looping over it** (`E0502`). Use an index loop, collect changes and apply them after, or build a new vector.

**5. Writing `n += 1` for a `&mut i32`** (`E0368`). Write `*n += 1`.

**6. Trying to take two `&mut` to two items of the same vector.** `let a = &mut v[0]; let b = &mut v[1];` fails with `E0499`: the compiler cannot know that the two indexes are different. The standard library has methods for this case (like `split_at_mut`); see Exercise 4.

## Try it yourself

1. In `~/projects/rust`, run `cargo new code/p03-mut-refs` (if you have not already done it in "Before you start"). Write the `add_world` program of section 1 and run it.
2. Remove `mut` from `let mut s`. Read the error and apply the compiler's suggestion.
3. Add `let r2 = &mut s;` right after `let r1 = &mut s;` and use both afterwards. Find the two labels in the error message.
4. Reorder the code so that `r1` is last used *before* `r2` is created. Does it compile now?
5. Commit: `git add code/p03-mut-refs && git commit -m "p03: mutable references"`.

## Exercises

### Exercise 1 — Be polite (easy)

Write `fn make_polite(s: &mut String)` that appends `", please"` to the string. In `main`, create `"open the door"`, call the function, and print the result.

<details>
<summary>Hint</summary>

Use `s.push_str(", please")`. In `main` you need `let mut request = ...` and `make_polite(&mut request)`.

</details>

<details>
<summary>Answer</summary>

```rust
fn make_polite(s: &mut String) {
    s.push_str(", please");
}

fn main() {
    let mut request = String::from("open the door");
    make_polite(&mut request);
    println!("{request}");
}
```

```text
open the door, please
```

Why it works: the function changes the caller's string in place through the mutable reference.

</details>

### Exercise 2 — Fix two borrow errors (easy)

Fix each program with the smallest change.

```rust
// program 1
fn main() {
    let mut count = 0;
    let a = &mut count;
    let b = &mut count;
    *a += 1;
    *b += 1;
    println!("{count}");
}
```

```rust
// program 2
fn shorten(s: &mut String) {
    s.truncate(3);
}

fn main() {
    let word = String::from("mutable");
    shorten(&mut word);
    println!("{word}");
}
```

<details>
<summary>Hint</summary>

Program 1: finish all use of `a` before creating `b`. Program 2: a variable must be declared `mut` to be borrowed mutably.

</details>

<details>
<summary>Answer</summary>

```rust
// program 1
fn main() {
    let mut count = 0;
    let a = &mut count;
    *a += 1;
    let b = &mut count;
    *b += 1;
    println!("{count}");
}
```

```text
2
```

```rust
// program 2
fn shorten(s: &mut String) {
    s.truncate(3);
}

fn main() {
    let mut word = String::from("mutable");
    shorten(&mut word);
    println!("{word}");
}
```

```text
mut
```

Why it works: in program 1, `a`'s last use is before `b` is created, so there is only one live `&mut` at any time. In program 2, `mut` gives `word` permission to be changed.

</details>

### Exercise 3 — Reset negatives, with tests (medium)

Write `fn reset_negatives(numbers: &mut Vec<i32>)` that replaces every negative number with `0`. Write three tests: a mixed vector, an empty vector, and a vector without negatives.

<details>
<summary>Hint</summary>

Loop with `for i in 0..numbers.len()` and check `numbers[i] < 0`. Compare vectors in tests with `assert_eq!(v, vec![4, 0, 7, 0]);`.

</details>

<details>
<summary>Answer</summary>

```rust
fn reset_negatives(numbers: &mut Vec<i32>) {
    for i in 0..numbers.len() {
        if numbers[i] < 0 {
            numbers[i] = 0;
        }
    }
}

fn main() {
    let mut data = vec![4, -1, 7, -3];
    reset_negatives(&mut data);
    println!("{:?}", data);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn negatives_become_zero() {
        let mut v = vec![4, -1, 7, -3];
        reset_negatives(&mut v);
        assert_eq!(v, vec![4, 0, 7, 0]);
    }

    #[test]
    fn empty_vector_is_fine() {
        let mut v: Vec<i32> = Vec::new();
        reset_negatives(&mut v);
        assert!(v.is_empty());
    }

    #[test]
    fn no_negatives_means_no_change() {
        let mut v = vec![1, 2, 3];
        reset_negatives(&mut v);
        assert_eq!(v, vec![1, 2, 3]);
    }
}
```

```text
[4, 0, 7, 0]
```

`cargo test` prints `test result: ok. 3 passed`.

Why it works: an index loop does not keep a borrow of the vector between steps, so we may assign to `numbers[i]`.

</details>

### Exercise 4 — Which compile? (medium)

Decide for each snippet whether it compiles. If not, name the error code.

```rust
// a
let mut s = String::from("a");
let r = &mut s;
r.push('b');
s.push('c');
println!("{s}");

// b
let mut s = String::from("a");
let r = &mut s;
s.push('c');
r.push('b');
println!("{s}");

// c
let mut v = vec![1, 2, 3];
let total: i32 = v.iter().sum();
v.push(total);
println!("{:?}", v);

// d
let mut v = vec![1, 2, 3];
let first = &mut v[0];
let second = &mut v[1];
*first += 1;
*second += 1;
println!("{:?}", v);
```

(Each snippet is the body of `main`. In (c), `v.iter().sum()` adds up all items; it is Lesson 05.5.)

<details>
<summary>Hint</summary>

For each `&mut`, find its last use. Is any other use of the same variable *between* the creation of the `&mut` and its last use?

</details>

<details>
<summary>Answer</summary>

- a: compiles; prints `abc`. The last use of `r` is before `s.push('c')`.
- b: **`E0499`**. `s.push('c')` needs a mutable borrow of `s` while `r` is still going to be used.
- c: compiles; prints `[1, 2, 3, 6]`. The shared borrow made by `v.iter()` ends when `sum()` is done, before `v.push`.
- d: **`E0499`**. The compiler does not know that indexes 0 and 1 are different items, so it treats both as a mutable borrow of the whole vector. (The standard library offers `split_at_mut` for this.)

Why it works: the rules look only at which borrows overlap in time, not at what you "meant".

</details>

### Exercise 5 — Swap ends and rotate, with tests (hard)

Write two functions, both taking `&mut Vec<i32>`:

- `swap_ends` exchanges the first and last items (a vector with fewer than 2 items stays unchanged).
- `rotate_left_one` moves the first item to the end: `[1, 2, 3, 4]` becomes `[2, 3, 4, 1]` (an empty vector stays unchanged).

Write tests for both.

<details>
<summary>Hint</summary>

For `swap_ends`, remember the first value in a variable (`i32` is `Copy`), then assign `v[0] = v[last]` and `v[last] = first_value`. For `rotate_left_one`, use `let first = v.remove(0);` (removes and returns the item at index 0) and then `v.push(first);`.

</details>

<details>
<summary>Answer</summary>

```rust
fn swap_ends(v: &mut Vec<i32>) {
    if v.len() < 2 {
        return;
    }
    let last = v.len() - 1;
    let first_value = v[0];
    v[0] = v[last];
    v[last] = first_value;
}

fn rotate_left_one(v: &mut Vec<i32>) {
    if v.is_empty() {
        return;
    }
    let first = v.remove(0);
    v.push(first);
}

fn main() {
    let mut a = vec![1, 2, 3, 4];
    swap_ends(&mut a);
    println!("{:?}", a);
    rotate_left_one(&mut a);
    println!("{:?}", a);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn swaps_first_and_last() {
        let mut v = vec![1, 2, 3, 4];
        swap_ends(&mut v);
        assert_eq!(v, vec![4, 2, 3, 1]);
    }

    #[test]
    fn short_vectors_are_left_alone() {
        let mut empty: Vec<i32> = vec![];
        let mut one = vec![5];
        swap_ends(&mut empty);
        swap_ends(&mut one);
        assert!(empty.is_empty());
        assert_eq!(one, vec![5]);
    }

    #[test]
    fn rotates_left() {
        let mut v = vec![1, 2, 3, 4];
        rotate_left_one(&mut v);
        assert_eq!(v, vec![2, 3, 4, 1]);
    }
}
```

```text
[4, 2, 3, 1]
[2, 3, 1, 4]
```

Why it works: `swap_ends` copies the first value out (it is `Copy`), so no second `&mut` is needed. `remove(0)` moves the item out of the vector legally, because the vector shifts its other items and closes the hole itself.

</details>

## Quiz

1. State the borrowing rules.

<details>
<summary>Answer</summary>

At any time you may have either any number of shared references or exactly one mutable reference, never both; and all references must be valid.

</details>

2. What do you need in order to create `&mut x`?

<details>
<summary>Answer</summary>

`x` must be declared with `let mut`, and no other reference to `x` may be alive.

</details>

3. Why can `for n in &v { v.push(1); }` not compile?

<details>
<summary>Answer</summary>

The loop holds a shared borrow of `v` for its whole length, and `push` needs a mutable borrow. `push` might move the buffer and invalidate the loop's view of it.

</details>

4. What is a data race, and how do the rules prevent it?

<details>
<summary>Answer</summary>

Two parts of a program use the same data at the same time, at least one writes, and nothing coordinates them. With "one writer or many readers", a writer never shares the data with anyone.

</details>

5. Why does `*n += 1` need a star when `n` is a `&mut i32`?

<details>
<summary>Answer</summary>

`n` is a reference, not a number. `*n` means the number it points to, and that is what we add to.

</details>

6. Does the borrow checker slow down my program?

<details>
<summary>Answer</summary>

No. It works only at compile time and adds no instructions at run time.

</details>

## Summary

- `&mut T` is a mutable (exclusive) reference. To use it you need `let mut`, `&mut x` at the call and `&mut T` in the signature.
- The borrowing rules: many `&T` **or** one `&mut T`, never both; references must always be valid.
- Typical errors: `E0499` (two `&mut`), `E0502` (`&mut` and `&` together), `E0506` (assigning to a borrowed value), `E0596` (not declared `mut`), `E0368` (forgot `*`).
- A borrow lasts until its last use, so reordering two lines often fixes a conflict.
- The rules prevent iterator invalidation, dangling pointers into moved buffers and data races, at no run-time cost.
- `std::mem::swap`, `replace` and `take` change values behind a `&mut` safely.

## Sources and further reading

- [The Rust Programming Language — 4.2 References and Borrowing](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html) — mutable references and the two rules.
- [std::mem](https://doc.rust-lang.org/std/mem/index.html) — `swap`, `replace`, `take` and friends.
- [Error code E0499](https://doc.rust-lang.org/error_codes/E0499.html) — more than one mutable borrow.
- [Error code E0502](https://doc.rust-lang.org/error_codes/E0502.html) — a mutable borrow while a shared borrow exists.
- [Error code E0506](https://doc.rust-lang.org/error_codes/E0506.html) — assigning to a borrowed value.

---

[🏠 Start](../START-HERE.md) · [← 03.4 References and borrowing](04-references-and-borrowing.md) · [📋 Phase 03](README.md) · [03.6 Slices and string types →](06-slices-and-string-types.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
