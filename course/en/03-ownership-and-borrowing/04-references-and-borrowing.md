<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 03.3 Move, copy and clone](03-move-copy-and-clone.md) · [📋 Phase 03](README.md) · [03.5 Mutable references and the borrowing rules →](05-mutable-references-and-the-borrow-rules.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/03-ownership-and-borrowing/04-references-and-borrowing.md) <!-- nav -->

# Lesson 03.4 — References and borrowing

> **Phase 03 — Ownership and borrowing** · Lesson 4 of 8 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a **reference** (`&`) is and how it differs from ownership.
- How to let a function *look at* a value without taking it (**borrowing**).
- How to read through a reference with `*` (dereferencing), and why you cannot change a value through `&`.
- Why Rust never allows a **dangling reference**, and how long a borrow lasts.
- When a function should borrow and when it should own.

## Before you start

- You finished [Lesson 03.3 — Move, copy and clone](03-move-copy-and-clone.md). You know that passing a `String` by value moves it.
- Create a project: `cargo new code/p03-borrowing` (from `~/projects/rust`).

## New words

| Word | Meaning |
|---|---|
| **reference** | A value that points to another value without owning it. Written `&value`. Its type is `&T`, "a reference to a `T`". |
| **borrow** | To create a reference. While the reference exists, the owner still owns the value. |
| **shared reference** (also *immutable reference*) | A reference `&T` that can only **read**. Any number of them can exist at the same time. |
| **dereference** | To follow a reference to the value it points to. Written `*r`. |
| **dangling reference** | A reference to a value that no longer exists. Rust makes it impossible. |
| **lifetime** | The period during which a reference is valid. The compiler checks it. (Lessons 05.3 and 06.5 teach lifetimes fully.) |
| **NLL** (Non-Lexical Lifetimes) | The compiler rule that a borrow ends at the *last use* of the reference, not at the end of the block. |
| **signature** | The first line of a function: its name, parameters and return type. |

## 1. The problem to solve

In Lesson 03.2 we wrote a function that only wanted the *length* of a string. Because passing a `String` moves it, the function had to hand the string back:

```rust
fn calculate_length(s: String) -> (String, usize) {
    let length = s.len();
    (s, length)
}

fn main() {
    let s1 = String::from("hello");
    let (s2, len) = calculate_length(s1);
    println!("The length of '{s2}' is {len}.");
}
```

```text
The length of 'hello' is 5.
```

It works, but it is silly. The function does not need to *own* the string; it only needs to look at it for a moment. A person who wants to read your book does not need to own it. They borrow it and give it back.

## 2. References: borrowing without owning

A **reference** lets you refer to a value without taking ownership of it. You create one with the `&` operator, and you write its type as `&T`:

```rust
fn calculate_length(s: &String) -> usize {
    s.len()
}

fn main() {
    let s1 = String::from("hello");
    let len = calculate_length(&s1);
    println!("The length of '{s1}' is {len}.");
}
```

```text
The length of 'hello' is 5.
```

**What happens:**

1. `&s1` creates a reference to `s1`. The `String` is **not moved**. `s1` stays its owner.
2. The function parameter `s: &String` means "a reference to a `String`". Inside the function, `s.len()` works as if `s` were the string itself.
3. When the function ends, `s` goes out of scope. But `s` does not own the `String`, so **nothing is dropped**. The reference simply disappears.
4. Back in `main`, `s1` is still fully valid. We use it again in `println!`.

The action of creating a reference is called **borrowing**. Like in real life, you borrow something, use it, and give it back; the owner stays the owner.

A reference is just an address. Let us look at it:

```rust
fn main() {
    let s = String::from("hello");
    let r = &s;

    println!("s is at   {:p}", &s);
    println!("r points to {:p}", r);
    println!("r itself is at {:p}", &r);
    println!("letters at  {:p}", s.as_ptr());
    println!("size of s: {} bytes, size of r: {} bytes", size_of::<String>(), size_of::<&String>());
}
```

```text
s is at   0x16d129d28
r points to 0x16d129d28
r itself is at 0x16d129d40
letters at  0x102d752d0
size of s: 24 bytes, size of r: 8 bytes
```

The reference `r` holds the address of `s` (the same number `0x16d129d28`), and it takes only 8 bytes:

```text
 STACK                               HEAP
┌─────────────────────┐
│ r: &String          │
│   address of s ─────┼──┐
└─────────────────────┘  │
┌─────────────────────┐  │         ┌───┬───┬───┬───┬───┐
│ s: String      ◄────┼──┘         │ h │ e │ l │ l │ o │
│   ptr ──────────────┼──────────► └───┴───┴───┴───┴───┘
│   len = 5           │
│   cap = 5           │
└─────────────────────┘
```

Passing `&s` to a function copies only 8 bytes, whatever the length of the text. It is cheap, like a move, but the owner keeps the value.

## 3. Reading through a reference

**Many readers at once.** A shared reference only reads, so it is safe to have as many as you want, and they can all be used at the same time. A reference can even be copied (references to shared data are `Copy`, as you saw in Lesson 03.3):

```rust
fn main() {
    let s = String::from("hello");
    let r1 = &s;
    let r2 = &s;
    let r3 = r1; // a reference can be copied
    println!("{r1} {r2} {r3}");
    println!("and the owner can still read: {s}");
}
```

```text
hello hello hello
and the owner can still read: hello
```

**Dereferencing.** For a reference to a number, you use `*` to get the number itself:

```rust
fn main() {
    let x = 5;
    let r = &x;
    println!("x = {x}, *r = {}", *r);
    println!("*r + 1 = {}", *r + 1);
    println!("r + 1 = {}", r + 1);
    println!("{}", *r == 5);
}
```

```text
x = 5, *r = 5
*r + 1 = 6
r + 1 = 6
true
```

`*r` means "the value that `r` points to". Rust often does the dereference for you: method calls like `s.len()` and the `println!` macro look through references automatically. Arithmetic with a plain number happens to work too (`r + 1`), but a **comparison** does not:

```rust
fn main() {
    let x = 5;
    let r = &x;
    if r == 5 {
        println!("five");
    }
}
```

```text
error[E0277]: can't compare `&{integer}` with `{integer}`
 --> src/main.rs:4:10
  |
4 |     if r == 5 {
  |          ^^ no implementation for `&{integer} == {integer}`
  |
  = help: the trait `PartialEq<{integer}>` is not implemented for `&{integer}`
help: consider dereferencing here
  |
4 |     if *r == 5 {
  |        +
```

It says that a reference-to-integer and an integer are different types, so they cannot be compared. The fix is the one shown: write `*r == 5`.

**A borrowed `Vec`.** The same works for a vector:

```rust
fn total(numbers: &Vec<i32>) -> i32 {
    let mut sum = 0;
    for n in numbers {
        sum += n;
    }
    sum
}

fn main() {
    let numbers = vec![3, 4, 5];
    println!("total = {}", total(&numbers));
    println!("total again = {}", total(&numbers));
    println!("numbers = {:?}", numbers);
}
```

```text
total = 12
total again = 12
numbers = [3, 4, 5]
```

Here `for n in numbers` loops over a *reference* to the vector, so `n` is a `&i32`, and `sum += n` adds the number it points to. Because the function only borrowed the vector, `main` can call it twice and still own `numbers` at the end.

> 📏 **Convention:** a function that only needs to read a value should take a reference. It does not force the caller to give up the value, and it is cheaper than a clone. The Rust tool `cargo clippy` (see Lesson 03.7) will, in fact, suggest `&str` instead of `&String` and `&[T]` instead of `&Vec<T>` for parameters. We keep `&String` and `&Vec<T>` for a few more lessons; Lesson 03.6 explains the better types.

## 4. You cannot change a value through `&`

A shared reference is read-only. If a function tries to change what it borrowed:

```rust
fn add_world(s: &String) {
    s.push_str(", world");
}

fn main() {
    let s = String::from("hello");
    add_world(&s);
    println!("{s}");
}
```

```text
error[E0596]: cannot borrow `*s` as mutable, as it is behind a `&` reference
 --> src/main.rs:2:5
  |
2 |     s.push_str(", world");
  |     ^ `s` is a `&` reference, so it cannot be borrowed as mutable
  |
help: consider changing this to be a mutable reference
  |
1 | fn add_world(s: &mut String) {
  |                  +++
```

This is a safety feature. Readers share the value, so nobody may change it under their feet. If you need to change a value through a reference, you need a **mutable reference**, `&mut`. That is the whole subject of [Lesson 03.5](05-mutable-references-and-the-borrow-rules.md), and the compiler already shows you the fix: `&mut String`.

## 5. References must always be valid

The compiler guarantees that **a reference never outlives the value it points to.** A reference that points to freed memory would be a dangling pointer (Lesson 03.1). Rust checks this at compile time.

### Returning a reference to a local variable

```rust
fn make_greeting() -> &String {
    let s = String::from("hello");
    &s
}

fn main() {
    let g = make_greeting();
    println!("{g}");
}
```

```text
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:23
  |
1 | fn make_greeting() -> &String {
  |                       ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: consider using the `'static` lifetime, but this is uncommon unless you're returning a borrowed value from a `const` or a `static`
  |
1 | fn make_greeting() -> &'static String {
  |                        +++++++
help: instead, you are more likely to want to return an owned value
  |
1 - fn make_greeting() -> &String {
1 + fn make_greeting() -> String {
  |
```

**What happens:** `s` is created inside the function and is dropped when the function ends. A reference to it would point to freed memory. The compiler notices that the return type is a reference, but there is no outside value to borrow from. The message mentions a **lifetime**: the period for which a reference is valid. You will learn how to write lifetimes in Lesson 05.3. For now the helpful line is the last one: "you are more likely to want to return an owned value". That is the fix:

```rust
fn make_greeting() -> String {
    String::from("hello")
}

fn main() {
    let g = make_greeting();
    println!("{g}");
}
```

```text
hello
```

Ownership moves out to the caller, and nothing dangles.

### A reference that lives longer than its owner

```rust
fn main() {
    let r;
    {
        let x = 5;
        r = &x;
    }
    println!("r = {r}");
}
```

```text
error[E0597]: `x` does not live long enough
 --> src/main.rs:5:13
  |
4 |         let x = 5;
  |             - binding `x` declared here
5 |         r = &x;
  |             ^^ borrowed value does not live long enough
6 |     }
  |     - `x` dropped here while still borrowed
7 |     println!("r = {r}");
  |                    - borrow later used here
```

This error reads like a story: `x` is created; `r` borrows it; `x` is dropped at the closing brace "while still borrowed"; and later `r` is used. `r` would have pointed to a dead variable. The compiler stops it.

### The owner cannot be moved while it is borrowed

```rust
fn main() {
    let s = String::from("hello");
    let r = &s;
    let t = s;
    println!("{r} {t}");
}
```

```text
error[E0505]: cannot move out of `s` because it is borrowed
 --> src/main.rs:4:13
  |
2 |     let s = String::from("hello");
  |         - binding `s` declared here
3 |     let r = &s;
  |             -- borrow of `s` occurs here
4 |     let t = s;
  |             ^ move out of `s` occurs here
5 |     println!("{r} {t}");
  |                - borrow later used here
```

If `s` moved to `t`, then `r` would point at the old place, and `t` could then drop the value while `r` is still in use. The rule: **while a borrow is alive, the owner cannot move or drop the value.**

### How long does a borrow last?

A borrow lasts until the **last use** of the reference, not until the end of the block. This is the rule called **NLL** (Non-Lexical Lifetimes). So this version is fine:

```rust
fn main() {
    let s = String::from("hello");
    let r = &s;
    println!("reading through r: {r}"); // last use of r
    let t = s; // fine: the borrow is already over
    println!("t = {t}");
}
```

```text
reading through r: hello
t = hello
```

After `println!` the reference `r` is never used again, so the borrow is over, and the move is allowed. This makes Rust friendlier than a "until the closing brace" rule would be.

> 🔒 **Security:** a dangling reference is the same bug as use-after-free from Lesson 03.1. In C, the compiler happily accepts it, and attackers have used such bugs for decades. In Rust the three errors above (`E0106`, `E0597`, `E0505`) turn that whole class of bugs into compile errors.

## 6. Borrow or own? A short guide

| The function needs to… | Take |
|---|---|
| read a value | `&T` |
| keep the value (store it, return it, send it away) | `T` (ownership) |
| change the value in place | `&mut T` (next lesson) |

Most functions only read, so most parameters are references.

## Common mistakes

**1. Forgetting the `&` at the call site.** If the function takes `&String` and you pass `s`, you get a type error `expected &String, found String`. Write `calculate_length(&s)`.

**2. Trying to modify through `&`.** Error `E0596` (section 4). You need `&mut` (next lesson).

**3. Returning a reference to a local variable.** Errors `E0106` or `E0515` (Lesson 03.7 shows the second one). Return the owned value instead.

**4. Comparing a reference with a plain value.** Error `E0277`. Dereference with `*`.

**5. Moving the owner while a reference is still used later.** Error `E0505`. Finish using the reference first, or reorder the code.

## Try it yourself

1. In `~/projects/rust`, run `cargo new code/p03-borrowing` (if you have not already done it in "Before you start"). In `src/main.rs`, write `calculate_length` with `&String` and call it twice with the same string. Confirm the string is still usable afterwards.
2. Add the `{:p}` lines from section 2 and check that `&s` and `r` print the same address.
3. Remove the `&` in the call `calculate_length(&s1)`. Read the error: what does it say is expected and what was found?
4. Write a function `fn print_all(items: &Vec<String>)` that prints every string with a `for` loop, and call it twice.
5. Commit: `git add code/p03-borrowing && git commit -m "p03: references and borrowing"`.

## Exercises

### Exercise 1 — Count a letter (easy)

Write `fn count_char(s: &String, target: char) -> usize` that counts how many times `target` appears in `s`. In `main`, call it twice with the same `String` (once for `'a'` and once for `'n'` in `"banana"`) and then print the string to show that it is still yours.

<details>
<summary>Hint</summary>

Loop with `for c in s.chars()`. `s.chars()` gives the characters one at a time. Compare each `c` with `target` and add 1 to a counter.

</details>

<details>
<summary>Answer</summary>

```rust
fn count_char(s: &String, target: char) -> usize {
    let mut n = 0;
    for c in s.chars() {
        if c == target {
            n += 1;
        }
    }
    n
}

fn main() {
    let text = String::from("banana");
    println!("a appears {} times", count_char(&text, 'a'));
    println!("n appears {} times", count_char(&text, 'n'));
    println!("text is still mine: {text}");
}
```

```text
a appears 3 times
n appears 2 times
text is still mine: banana
```

Why it works: the function borrows the string, so `text` is never moved and can be used again.

</details>

### Exercise 2 — Stop the move (easy)

Fix this program by changing **only the function and the call**, not by cloning:

```rust
fn print_len(v: Vec<i32>) {
    println!("the vector has {} items", v.len());
}

fn main() {
    let numbers = vec![1, 2, 3];
    print_len(numbers);
    println!("first number: {}", numbers[0]);
}
```

<details>
<summary>Hint</summary>

Change the parameter type to `&Vec<i32>` and pass `&numbers`.

</details>

<details>
<summary>Answer</summary>

```rust
fn print_len(v: &Vec<i32>) {
    println!("the vector has {} items", v.len());
}

fn main() {
    let numbers = vec![1, 2, 3];
    print_len(&numbers);
    println!("first number: {}", numbers[0]);
}
```

```text
the vector has 3 items
first number: 1
```

Why it works: borrowing leaves `numbers` with `main`.

</details>

### Exercise 3 — Largest number, with tests (medium)

Write `fn largest(numbers: &Vec<i32>) -> i32` that returns the biggest number in the vector, or `i32::MIN` (the smallest possible `i32`) when the vector is empty. Write four tests: a normal case, negative numbers, an empty vector, and a test that the vector is still usable after the call.

<details>
<summary>Hint</summary>

Start with `let mut best = i32::MIN;`, loop `for n in numbers`, and compare `*n > best`. In the tests, pass `&vec![3, 9, 2]`.

</details>

<details>
<summary>Answer</summary>

```rust
fn largest(numbers: &Vec<i32>) -> i32 {
    let mut best = i32::MIN;
    for n in numbers {
        if *n > best {
            best = *n;
        }
    }
    best
}

fn main() {
    let numbers = vec![3, 9, 2];
    println!("largest = {}", largest(&numbers));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_the_largest() {
        assert_eq!(largest(&vec![3, 9, 2]), 9);
    }

    #[test]
    fn works_with_negative_numbers() {
        assert_eq!(largest(&vec![-5, -2, -9]), -2);
    }

    #[test]
    fn empty_vector_gives_the_smallest_i32() {
        assert_eq!(largest(&vec![]), i32::MIN);
    }

    #[test]
    fn does_not_consume_the_vector() {
        let v = vec![1, 2, 3];
        let _ = largest(&v);
        assert_eq!(v.len(), 3);
    }
}
```

```text
running 4 tests
....
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `n` is a `&i32`, so we dereference with `*n` to compare and store the number. The last test would not even compile if `largest` took the vector by value.

</details>

### Exercise 4 — Which programs compile? (medium)

Decide for each program whether it compiles. If not, give the error code and say why.

```rust
// a
let name = String::from("Ada");
let r = &name;
let moved = name;
println!("{moved}");
println!("{r}");

// b
let name = String::from("Ada");
let r = &name;
println!("{r}");
let moved = name;
println!("{moved}");

// c
let outer;
{
    let inner = String::from("temporary");
    outer = &inner;
    println!("inside: {outer}");
}

// d
let a = String::from("x");
let r1 = &a;
let r2 = &a;
let r3 = &r1;
println!("{r1} {r2} {r3}");
```

(Each snippet is the body of `main`.)

<details>
<summary>Hint</summary>

For each reference ask: when is it used last? And is the owner moved or dropped *before* that point?

</details>

<details>
<summary>Answer</summary>

- a: **`E0505`**. `name` is moved while `r` is still used later (`println!("{r}")`).
- b: compiles. `r` is last used before the move.
- c: compiles. `outer` is only used inside the block, while `inner` is alive. (If you used `outer` after the closing brace, you would get `E0597`.)
- d: compiles. Any number of shared references can exist, and `r3` is simply a reference to a reference (`&&String`).

Why it works: the compiler tracks, for each borrow, the region from its creation to its last use. A conflict appears only when the owner is moved or dropped inside that region.

</details>

### Exercise 5 — Borrow twice, then give away (hard)

Write `fn total_length(words: &Vec<String>) -> usize` that returns the sum of the lengths of all strings, and `fn consume(words: Vec<String>)` that prints how many words it received. In `main`, create a vector of three strings, call `total_length` twice (printing both results), and only then call `consume`. Then try to call `total_length` *after* `consume` and read the error.

<details>
<summary>Hint</summary>

`total_length` takes `&words`; `consume` takes `words` itself. After `consume(words)`, the vector has been moved.

</details>

<details>
<summary>Answer</summary>

```rust
fn total_length(words: &Vec<String>) -> usize {
    let mut total = 0;
    for w in words {
        total += w.len();
    }
    total
}

fn consume(words: Vec<String>) {
    println!("consumed {} words", words.len());
}

fn main() {
    let words = vec![String::from("alpha"), String::from("beta"), String::from("gamma")];
    println!("first call:  {}", total_length(&words));
    println!("second call: {}", total_length(&words));
    consume(words);
}
```

```text
first call:  14
second call: 14
consumed 3 words
```

If you add `total_length(&words);` after `consume(words);`, you get `error[E0382]: borrow of moved value: `words``.

Why it works: borrowing can happen any number of times while the owner holds the value. Once the value moves into `consume`, the owner is gone.

</details>

## Quiz

1. What is a reference, and what does `&s` create?

<details>
<summary>Answer</summary>

A reference is a value that points to another value without owning it. `&s` creates a shared (read-only) reference to `s`; `s` remains the owner.

</details>

2. What happens when a reference goes out of scope?

<details>
<summary>Answer</summary>

Nothing is dropped, because the reference does not own the value. Only the owner's scope end drops the value.

</details>

3. How many shared references to one value can exist at the same time?

<details>
<summary>Answer</summary>

Any number. They can only read, so they cannot disturb each other.

</details>

4. Why can you not call `push_str` on a `&String`?

<details>
<summary>Answer</summary>

A shared reference is read-only. Changing a value needs a mutable reference, `&mut String` (Lesson 03.5).

</details>

5. Why does Rust refuse `fn f() -> &String { let s = String::from("x"); &s }`?

<details>
<summary>Answer</summary>

`s` is dropped when the function ends, so the returned reference would dangle. Return the owned `String` instead.

</details>

6. When does a borrow end?

<details>
<summary>Answer</summary>

At the last use of the reference (NLL), not at the end of the block.

</details>

7. What does `*r` mean?

<details>
<summary>Answer</summary>

Dereferencing: the value that the reference `r` points to.

</details>

## Summary

- A reference (`&T`) points to a value without owning it. Creating one is called borrowing; the owner keeps the value.
- A function that only reads should take `&T`. The caller writes `&value` at the call.
- A shared reference is read-only, and any number of them can exist at once. Use `*r` to get the value behind a reference when needed.
- A reference never outlives its owner: returning a reference to a local variable (`E0106`/`E0515`) or using one after its owner is dropped (`E0597`) is a compile error.
- While a borrow is alive, the owner cannot be moved or dropped (`E0505`). A borrow lasts until its last use.
- Borrowing is as cheap as a move (8 bytes) and avoids clones.

## Sources and further reading

- [The Rust Programming Language — 4.2 References and Borrowing](https://doc.rust-lang.org/book/ch04-02-references-and-borrowing.html) — the official chapter, including dangling references.
- [Rust By Example — Borrowing](https://doc.rust-lang.org/rust-by-example/scope/borrow.html) — short runnable examples.
- [Error code E0106](https://doc.rust-lang.org/error_codes/E0106.html) — missing lifetime specifier.
- [Error code E0597](https://doc.rust-lang.org/error_codes/E0597.html) — value does not live long enough.
- [Error code E0505](https://doc.rust-lang.org/error_codes/E0505.html) — move out of a value that is borrowed.

---

[🏠 Start](../START-HERE.md) · [← 03.3 Move, copy and clone](03-move-copy-and-clone.md) · [📋 Phase 03](README.md) · [03.5 Mutable references and the borrowing rules →](05-mutable-references-and-the-borrow-rules.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
