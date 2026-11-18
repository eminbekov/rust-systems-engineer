<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 03.2 The ownership rules](02-ownership-rules.md) · [📋 Phase 03](README.md) · [03.4 References and borrowing →](04-references-and-borrowing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/03-ownership-and-borrowing/03-move-copy-and-clone.md) <!-- nav -->

# Lesson 03.3 — Move, copy and clone

> **Phase 03 — Ownership and borrowing** · Lesson 3 of 8 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What exactly happens in memory when a value is **moved**, and why the old variable becomes unusable.
- Which types are **copied** automatically (`Copy`) and which are moved.
- How to make an independent duplicate with `.clone()`, and what it costs.
- Where moves hide: assignments, function calls, `Vec::push`, `for` loops and indexing.
- How to choose between move, copy, clone and (soon) borrowing.

## Before you start

- You finished [Lesson 03.2 — The ownership rules](02-ownership-rules.md). You know the three rules, and that passing a `String` to a function moves it.
- Create a project: `cargo new code/p03-move-copy-clone` (from `~/projects/rust`).

## New words

| Word | Meaning |
|---|---|
| **move** | Transferring ownership of a value to a new owner. The value itself is not copied; the old owner becomes invalid. |
| **handle** | The small, fixed-size part of a value that lives on the stack and points to the real data. For a `String`: pointer, length, capacity. |
| **Copy** | A *trait* (a set of abilities; Lesson 05.2) that marks a type as "cheap to duplicate by copying its bytes". Values of such types are copied instead of moved. |
| **Clone** | A trait with a method `.clone()` that makes a full, independent duplicate of a value, including its heap data. |
| **deep copy** | A duplicate that also duplicates everything the value owns on the heap. `.clone()` of a `String` is a deep copy. |
| **bitwise copy** | Copying the bytes of a value exactly as they are, without any extra work. |
| **`.clone()`** | The method that performs a clone. It always shows up in your code, so you can see where copies are made. |
| **iterator** | A thing that gives out items one by one. A `for` loop uses one. (Lesson 05.5 teaches iterators.) |

## 1. A move, in pictures

Take this code:

```rust
fn main() {
    let s = String::from("hello");
    let before = s.as_ptr();
    println!("s: len = {}, capacity = {}, letters at {:p}", s.len(), s.capacity(), before);

    let t = s; // the move
    println!("t: len = {}, capacity = {}, letters at {:p}", t.len(), t.capacity(), t.as_ptr());
    println!("same letters in memory: {}", before == t.as_ptr());
}
```

**What happens:**

1. `s` is a `String`: a handle (pointer, length, capacity) on the stack and the letters `hello` on the heap. We save the address of the letters in `before`.
2. `let t = s;` copies the 24-byte **handle** into `t`. It does **not** copy the letters. Then Rust marks `s` as "moved from" and will not let you use it again.
3. We print `t`'s details. They are the same as `s`'s were, and the letters are at the same address.

```text
s: len = 5, capacity = 5, letters at 0x102fc92d0
t: len = 5, capacity = 5, letters at 0x102fc92d0
same letters in memory: true
```

In pictures:

```text
BEFORE `let t = s;`                      AFTER `let t = s;`

 STACK                 HEAP               STACK                  HEAP
┌──────────────┐     ┌─────────┐        ┌──────────────┐       ┌─────────┐
│ s  ptr ──────┼───► │ h e l.. │        │ s  (moved,   │       │ h e l.. │
│    len = 5   │     └─────────┘        │    unusable) │       └─────────┘
│    cap = 5   │                        ├──────────────┤            ▲
└──────────────┘                        │ t  ptr ──────┼────────────┘
                                        │    len = 5   │
                                        │    cap = 5   │
                                        └──────────────┘
```

A move is therefore **very cheap**: only a few bytes are copied, however long the text is. The heap data stays where it is.

Why does Rust make `s` unusable? Think about what would happen otherwise. If both `s` and `t` pointed to the letters and both were valid, then at the end of the scope **both** would be dropped, and the same heap memory would be freed twice. This is the double free from Lesson 03.1. The rule "one owner at a time" prevents it: after the move only `t` is the owner, so only `t` frees the memory.

If you try to use `s` after the move, the compiler stops you:

```rust
fn main() {
    let s = String::from("hello");
    let t = s;
    println!("{s}, world!");
    println!("{t}");
}
```

```text
error[E0382]: borrow of moved value: `s`
 --> src/main.rs:4:16
  |
2 |     let s = String::from("hello");
  |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
3 |     let t = s;
  |             - value moved here
4 |     println!("{s}, world!");
  |                ^ value borrowed here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
3 |     let t = s.clone();
  |              ++++++++
```

The important phrase is "which does not implement the `Copy` trait". Section 3 explains it.

## 2. Where moves hide

A move happens whenever a value with a handle is *used by value*. You already know function arguments. Here are the other common places.

**Pushing into a `Vec`.** `push` takes the item by value, so the item moves into the vector:

```rust
fn main() {
    let mut names = Vec::new();
    let first = String::from("Ada");
    names.push(first);
    println!("{first}");
    println!("{:?}", names);
}
```

```text
error[E0382]: borrow of moved value: `first`
 --> src/main.rs:5:16
  |
3 |     let first = String::from("Ada");
  |         ----- move occurs because `first` has type `String`, which does not implement the `Copy` trait
4 |     names.push(first);
  |                ----- value moved here
5 |     println!("{first}");
  |                ^^^^^ value borrowed here after move
```

The vector is now the owner of the `"Ada"` string. That is logical: a list that holds a string must own it, or the string could be freed while still in the list.

**A `for` loop over a `Vec`.** `for name in names` takes the whole vector by value. It *consumes* it: each item moves into `name`, one after another, and the vector itself is gone after the loop.

```rust
fn main() {
    let names = vec![String::from("Ada"), String::from("Grace")];
    for name in names {
        println!("hello, {name}");
    }
    println!("{} names", names.len());
}
```

```text
error[E0382]: borrow of moved value: `names`
 --> src/main.rs:6:26
  |
2 |     let names = vec![String::from("Ada"), String::from("Grace")];
  |         ----- move occurs because `names` has type `Vec<String>`, which does not implement the `Copy` trait
3 |     for name in names {
  |                 ----- `names` moved due to this implicit call to `.into_iter()`
...
6 |     println!("{} names", names.len());
  |                          ^^^^^ value borrowed here after move
  |
help: consider iterating over a slice of the `Vec<String>`'s content to avoid moving into the `for` loop
  |
3 |     for name in &names {
  |                 +
```

The compiler gives the fix: write `for name in &names`. The `&` means "borrow the vector, do not take it". References are the topic of the next lesson; for now, copy the pattern:

```rust
fn main() {
    let names = vec![String::from("Ada"), String::from("Grace")];
    for name in &names {
        println!("hello, {name}");
    }
    println!("{} names", names.len());
}
```

```text
hello, Ada
hello, Grace
2 names
```

**Moves in only one branch.** If a value is moved in *some* path through the code, Rust treats it as moved afterwards:

```rust
fn main() {
    let s = String::from("hello");
    let keep = std::env::args().count() > 5;
    if keep {
        drop(s);
    }
    println!("{s}");
}
```

```text
error[E0382]: borrow of moved value: `s`
 --> src/main.rs:7:16
  |
2 |     let s = String::from("hello");
  |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
...
5 |         drop(s);
  |              - value moved here
6 |     }
7 |     println!("{s}");
  |                ^ value borrowed here after move
```

(`std::env::args().count()` counts the command-line arguments; here it just gives a value the compiler cannot know in advance.) The compiler does not know whether the `if` runs, so it assumes the worst case.

**A moved variable can get a new value.** A variable declared with `mut` that was moved from can be assigned again, and then it is valid again:

```rust
fn main() {
    let mut s = String::from("first");
    let t = s;
    println!("t = {t}");

    s = String::from("second");
    println!("s = {s}");
}
```

```text
t = first
s = second
```

## 3. Copy: types that are not moved

Now the opposite question. Why does this work?

```rust
fn main() {
    let a = 5;
    let b = a;
    println!("a = {a}, b = {b}");

    let pair = (1, 2.5);
    let pair2 = pair;
    println!("{:?} and {:?}", pair, pair2);

    let letters = ['x', 'y', 'z'];
    let letters2 = letters;
    println!("{:?} and {:?}", letters, letters2);

    let flag = true;
    let flag2 = flag;
    println!("{flag} and {flag2}");
}
```

```text
a = 5, b = 5
(1, 2.5) and (1, 2.5)
['x', 'y', 'z'] and ['x', 'y', 'z']
true and true
```

(`{:?}` prints a value in the "debug" format, which works for tuples and arrays.)

Here `let b = a;` does not make `a` unusable. The reason is that `i32` has a special ability: it implements the **`Copy`** trait. A `Copy` type is so small and simple that Rust copies its bytes whenever you "move" it — a **bitwise copy**. Both variables stay valid, and each owns its own copy. There is no heap and no resource, so there is nothing to free twice.

Which types are `Copy`?

| Type | Copy? | Why |
|---|---|---|
| `i32`, `u8`, `u64`, `usize`, … (all integers) | yes | plain numbers |
| `f32`, `f64` | yes | plain numbers |
| `bool`, `char` | yes | plain values |
| `(i32, f64)`, `[char; 3]` | yes, **if every item is `Copy`** | all parts are small copies |
| `&T` (a shared reference, Lesson 03.4) | yes | it is just an address |
| `&str` | yes | it is a shared reference |
| `String`, `Vec<T>`, `Box<T>` | **no** | they own heap memory |
| `(i32, String)` | **no** | it contains a `String` |
| `&mut T` (a mutable reference, Lesson 03.5) | **no** | only one may exist at a time |

The rule of thumb: **a type can be `Copy` only if it owns nothing that must be cleaned up.** A type that has a `Drop` implementation or owns heap memory can never be `Copy`.

A tuple is `Copy` only if all of its items are:

```rust
fn main() {
    let mixed = (1, String::from("x"));
    let other = mixed;
    println!("{:?}", mixed);
    println!("{:?}", other);
}
```

```text
error[E0382]: borrow of moved value: `mixed`
 --> src/main.rs:4:22
  |
2 |     let mixed = (1, String::from("x"));
  |         ----- move occurs because `mixed` has type `(i32, String)`, which does not implement the `Copy` trait
3 |     let other = mixed;
  |                 ----- value moved here
4 |     println!("{:?}", mixed);
  |                      ^^^^^ value borrowed here after move
```

> 📏 **Convention:** when you design your own types in Lesson 04.1, you will see `#[derive(Clone, Copy)]`, a one-line way to make a small type `Copy`. Do it for small "plain data" types (a point, a color). Do not do it for types that own heap data — the compiler will refuse.

## 4. Clone: an explicit, independent duplicate

Sometimes you really need two independent strings. You ask for it with `.clone()`:

```rust
fn main() {
    let original = String::from("hello");
    let mut copy = original.clone();
    copy.push_str(", world");

    println!("original = {original}");
    println!("copy     = {copy}");
    println!("original letters at {:p}", original.as_ptr());
    println!("copy letters at     {:p}", copy.as_ptr());
}
```

**What happens:**

1. `original.clone()` asks the allocator for new heap memory, copies the letters into it, and builds a new handle. This is a **deep copy**: handle *and* heap data.
2. `copy.push_str(...)` changes only `copy`. `original` is not affected, because they share nothing.
3. The two letters addresses are different. There are two separate pieces of heap memory.

```text
original = hello
copy     = hello, world
original letters at 0x1036d12d0
copy letters at     0x1036d1300
```

```text
original                     copy
 ptr ──► [h e l l o]          ptr ──► [h e l l o , _ w o r l d]
 len 5                        len 12
```

Now the earlier `Vec` example has a fix: clone, then push the clone and keep the original.

```rust
fn main() {
    let mut names = Vec::new();
    let first = String::from("Ada");
    names.push(first.clone());
    println!("{first}");
    println!("{:?}", names);
}
```

```text
Ada
["Ada"]
```

> ⚡ **Performance:** a move costs a few bytes; a clone costs an allocation plus a copy of all the data, so cloning a 100 MB string copies 100 MB. The key point is that **a clone is always visible** in the code as `.clone()`. When you review code, every `.clone()` is a place to ask: "Is this copy really needed?". Beginners often add `.clone()` just to make an error disappear. That compiles, but may hide a design problem. Try borrowing first (Lesson 03.4); clone when you truly need two independent owners.

## 5. Moving out of a Vec by index

A common beginner surprise:

```rust
fn main() {
    let names = vec![String::from("Ada"), String::from("Grace")];
    let first = names[0];
    println!("{first}");
}
```

```text
error[E0507]: cannot move out of index of `Vec<String>`
 --> src/main.rs:3:17
  |
3 |     let first = names[0];
  |                 ^^^^^^^^ move occurs because value has type `String`, which does not implement the `Copy` trait
  |
help: consider borrowing here
  |
3 |     let first = &names[0];
  |                 +
help: consider cloning the value if the performance cost is acceptable
  |
3 |     let first = names[0].clone();
  |                         ++++++++
```

If this worked, the vector would have a "hole" at position 0: an element that was moved out, with the vector still believing it owns it. At the end the vector would try to drop it, and the new owner would drop it too — a double free. So Rust forbids moving out of an index. The compiler offers two fixes:

```rust
fn main() {
    let names = vec![String::from("Ada"), String::from("Grace")];

    let borrowed = &names[0];
    println!("borrowed: {borrowed}");

    let owned = names[1].clone();
    println!("owned: {owned}");

    println!("names is still whole: {:?}", names);
}
```

```text
borrowed: Ada
owned: Grace
names is still whole: ["Ada", "Grace"]
```

With `&names[0]` you *look at* the item (a borrow; next lesson). With `.clone()` you get your own copy. (There are also methods that remove an item from the vector and hand it to you, such as `remove`. You will meet them in Lesson 04.7.)

## 6. How to choose

| Situation | Use |
|---|---|
| The value is a small plain type (`i32`, `bool`, `char`, …) | nothing special — it is copied |
| I am done with the old variable and want the new owner to have the value | **move** (just assign or pass it) |
| I only want to look at the value for a while | **borrow** (`&`, next lesson) |
| I need two independent owners that may change separately | **`.clone()`** |
| The compiler says "value used after move" and I don't know why | find the move (the label `value moved here`); ask whether borrowing is enough |

## Common mistakes

**1. Adding `.clone()` everywhere "until it compiles".** It works, but it hides the real question (who should own this?) and costs memory and time. Fix the design, or borrow, first.

**2. Thinking a move copies the heap data.** It copies only the handle. This is why moves are cheap and why the old variable has to be disabled.

**3. Expecting your own types to be `Copy` automatically.** A new type is *not* `Copy` unless you say so (Lesson 04.1).

**4. Forgetting that `for x in vec` consumes the vector.** Write `for x in &vec` when you still need the vector afterwards.

**5. Moving out of a vector with `v[0]` for a non-`Copy` item (`E0507`).** Use `&v[0]` or `v[0].clone()`.

## Try it yourself

1. In `~/projects/rust`, run `cargo new code/p03-move-copy-clone` (if you have not already done it in "Before you start"). In `src/main.rs`, write the `move_demo` program from section 1 and run it. Is `same letters in memory` really `true`?
2. Add `println!("{s}");` at the end of `main`. Read the compiler's error and find the three labels.
3. Replace `let t = s;` with `let t = s.clone();`. The error disappears. Now compare `s.as_ptr()` and `t.as_ptr()`: what do you see?
4. Change the `String` to an `i32` (`let s = 5;`) and remove the pointer lines. It compiles and both variables work. Why?
5. Commit: `git add code/p03-move-copy-clone && git commit -m "p03: move, copy, clone experiments"`.

## Exercises

### Exercise 1 — Does it compile? (easy)

For each program, say whether it compiles. If not, name the error code.

```rust
// a
let a = String::from("one");
let b = a;
println!("{b}");

// b
let a = 10;
let b = a;
println!("{a} {b}");

// c
let a = String::from("one");
let b = a.clone();
println!("{a} {b}");

// d
fn take(s: String) { println!("{s}"); }
let a = String::from("one");
take(a);
take(a);

// e
let a = [1, 2, 3];
let b = a;
println!("{:?} {:?}", a, b);

// f
let a = vec![1, 2, 3];
let b = a;
println!("{:?} {:?}", a, b);
```

(Each snippet is the body of `main`, except that `take` in (d) is defined next to it.)

<details>
<summary>Hint</summary>

Ask for each type: is it `Copy`? If not, `let b = a;` moves, and `a` is dead afterwards.

</details>

<details>
<summary>Answer</summary>

- a: compiles (`a` is not used after the move).
- b: compiles (`i32` is `Copy`).
- c: compiles (`clone` makes an independent copy).
- d: **error `E0382`** (`a` moved into the first `take`).
- e: compiles (an array of `Copy` items is `Copy`).
- f: **error `E0382`** (`Vec` owns heap memory, so it is moved).

Why it works: only types that own nothing to clean up can be copied implicitly.

</details>

### Exercise 2 — Copy or not? (easy)

Without running anything, decide which of these types are `Copy`: `i32`, `String`, `bool`, `(i32, f64)`, `(i32, String)`, `[i32; 3]`, `Vec<i32>`, `char`, `&str`, `&String`. Then check yourself with this program, which compiles only if every type in it is `Copy`:

```rust
fn is_copy<T: Copy>() {}

fn main() {
    is_copy::<i32>();
    is_copy::<bool>();
    is_copy::<char>();
    is_copy::<(i32, f64)>();
    is_copy::<[i32; 3]>();
    is_copy::<&str>();
    is_copy::<&String>();
    println!("all of these are Copy");
}
```

(`T: Copy` means "`T` must be a `Copy` type"; this is a trait bound, Lesson 05.2.)

<details>
<summary>Hint</summary>

Does the type own heap memory? If yes, it is not `Copy`. A shared reference (`&…`) is only an address, so it is `Copy`.

</details>

<details>
<summary>Answer</summary>

`Copy`: `i32`, `bool`, `(i32, f64)`, `[i32; 3]`, `char`, `&str`, `&String`. Not `Copy`: `String`, `(i32, String)`, `Vec<i32>`.

```text
all of these are Copy
```

If you add `is_copy::<String>();`, the compiler answers `error[E0277]: the trait bound `String: Copy` is not satisfied`.

Why it works: the function `is_copy` can only be instantiated with `Copy` types. It is a small trick to ask the compiler a question about a type.

</details>

### Exercise 3 — Fix with the smallest change (medium)

This program fails with `E0382`. Fix it so that it prints the title and then the vector. Use exactly one `.clone()`, in the right place, and explain why it is the right place.

```rust
fn main() {
    let title = String::from("Rust");
    let mut titles = Vec::new();
    titles.push(title);
    titles.push(String::from("Ownership"));
    println!("first title was {title}");
    println!("{:?}", titles);
}
```

<details>
<summary>Hint</summary>

The vector needs an owned `String`. You also want to keep `title`. So the vector should get a copy.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let title = String::from("Rust");
    let mut titles = Vec::new();
    titles.push(title.clone());
    titles.push(String::from("Ownership"));
    println!("first title was {title}");
    println!("{:?}", titles);
}
```

```text
first title was Rust
["Rust", "Ownership"]
```

Why it works: `push` needs ownership. Cloning at the call gives the vector its own `String`, and `title` keeps the original. The second `push` needs no clone, because the new `String::from(...)` is a fresh value that nobody else uses.

</details>

### Exercise 4 — Duplicate, with tests (medium)

Write `fn duplicate(s: String) -> (String, String)` that returns two independent copies of its input. Write two tests: one that checks both strings are equal, and one that changes the first and checks the second is unchanged.

<details>
<summary>Hint</summary>

Clone first, then return the tuple `(s, copy)`. In the second test, declare the first part `mut`, call `push_str` on it, and compare both with `assert_eq!`.

</details>

<details>
<summary>Answer</summary>

```rust
fn duplicate(s: String) -> (String, String) {
    let copy = s.clone();
    (s, copy)
}

fn main() {
    let (a, b) = duplicate(String::from("echo"));
    println!("{a} {b}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn both_halves_are_equal() {
        let (a, b) = duplicate(String::from("echo"));
        assert_eq!(a, b);
    }

    #[test]
    fn halves_are_independent() {
        let (mut a, b) = duplicate(String::from("echo"));
        a.push_str("!!");
        assert_eq!(a, "echo!!");
        assert_eq!(b, "echo");
        assert_ne!(a.as_ptr(), b.as_ptr());
    }
}
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the clone is a deep copy, so changing `a` cannot change `b`. The last assertion also proves that the letters live at different addresses.

</details>

### Exercise 5 — Same memory or not? (hard)

Predict the three lines of output, then run the program.

```rust
fn main() {
    let a = String::from("data");
    let p = a.as_ptr();

    let b = a;
    println!("1) after move:   {}", b.as_ptr() == p);

    let c = b.clone();
    println!("2) clone vs b:   {}", c.as_ptr() == b.as_ptr());

    let d = c.clone();
    let e = d;
    println!("3) d moved to e: {}", e.as_ptr() == c.as_ptr());
}
```

<details>
<summary>Hint</summary>

A move copies the handle only. A clone allocates new heap memory. In line 3, `e` is a move of `d`, but `d` itself was a clone of `c`.

</details>

<details>
<summary>Answer</summary>

```text
1) after move:   true
2) clone vs b:   false
3) d moved to e: false
```

Why it works: line 1: a move keeps the same heap memory. Line 2: a clone creates a new piece, so the addresses differ. Line 3: `e` has the heap memory of `d`, which was created by cloning `c`, so it is a different piece from `c`'s.

</details>

## Quiz

1. What is copied when you write `let t = s;` for a `String`?

<details>
<summary>Answer</summary>

Only the 24-byte handle (pointer, length, capacity). The letters on the heap are not copied. Then `s` becomes unusable.

</details>

2. Why does Rust disable `s` after the move?

<details>
<summary>Answer</summary>

Otherwise both `s` and `t` would try to free the same heap memory at the end of the scope (a double free). With one owner at a time, only `t` frees it.

</details>

3. What makes `i32` different from `String` for assignments?

<details>
<summary>Answer</summary>

`i32` is `Copy`: it owns nothing that needs cleaning up, so its bytes are simply copied and both variables stay valid. `String` owns heap memory, so it is moved.

</details>

4. When is a tuple `Copy`?

<details>
<summary>Answer</summary>

Only when every item in it is `Copy`. `(i32, f64)` is; `(i32, String)` is not.

</details>

5. What does `.clone()` cost compared with a move?

<details>
<summary>Answer</summary>

A clone allocates new heap memory and copies all the data (time and memory proportional to the size). A move copies only the small handle.

</details>

6. Why does `for name in names` make `names` unusable, and what is the usual fix?

<details>
<summary>Answer</summary>

The loop takes the vector by value and consumes it, moving each item out. Write `for name in &names` to borrow the vector instead.

</details>

7. Why can you not write `let first = names[0];` for a `Vec<String>`?

<details>
<summary>Answer</summary>

It would move a `String` out of the middle of the vector and leave a hole, which could lead to a double free. Use `&names[0]` or `names[0].clone()`.

</details>

## Summary

- A **move** copies only the small handle of a value and invalidates the old variable; the heap data does not move and nothing is freed twice.
- Types that own nothing to clean up (numbers, `bool`, `char`, `&T`, and tuples/arrays of those) are **`Copy`**: assignments copy them and both variables stay valid.
- Types that own heap memory (`String`, `Vec`, `Box`) are moved.
- `.clone()` makes a deep, independent duplicate. It is explicit in the code and has a real cost.
- Moves hide in function calls, `push`, `for x in vec`, and conditional branches; moving out of an index (`v[0]`) is forbidden for non-`Copy` items.
- Prefer borrowing to cloning, and clone only when two independent owners are really needed.

## Sources and further reading

- [The Rust Programming Language — 4.1 What Is Ownership?](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html) — the sections "Variables and Data Interacting with Move" and "…with Clone".
- [std::marker::Copy](https://doc.rust-lang.org/std/marker/trait.Copy.html) — the exact rules for `Copy`, and the list of standard types that implement it.
- [std::clone::Clone](https://doc.rust-lang.org/std/clone/trait.Clone.html) — the `Clone` trait and its relation to `Copy`.
- [Rust By Example — Ownership and moves](https://doc.rust-lang.org/rust-by-example/scope/move.html) — more small move examples.
- [Error code E0507](https://doc.rust-lang.org/error_codes/E0507.html) — "cannot move out of borrowed content".

---

[🏠 Start](../START-HERE.md) · [← 03.2 The ownership rules](02-ownership-rules.md) · [📋 Phase 03](README.md) · [03.4 References and borrowing →](04-references-and-borrowing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
