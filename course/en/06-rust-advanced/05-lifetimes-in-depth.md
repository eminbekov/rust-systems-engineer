<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 06.4 Interior mutability](04-interior-mutability.md) · [📋 Phase 06](README.md) · [06.6 Unsafe Rust and Miri →](06-unsafe-rust-and-miri.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/06-rust-advanced/05-lifetimes-in-depth.md) <!-- nav -->

# Lesson 06.5 — Lifetimes in depth

> **Phase 06 — Rust advanced** · Lesson 5 of 12 · ⏱ about 4 hours (2 sittings)

## What you will learn

- The exact lifetime elision rules, so you know when the compiler fills in lifetimes for you.
- How to use several lifetimes in one signature, and when it matters.
- How structs and `impl` blocks use lifetimes, and the difference between `&'a T` and `&self`.
- What `'static` and `T: 'static` really mean (and why `thread::spawn` demands it).
- Subtyping and variance: why `&'static str` can be used as `&'a str`, but `&mut &'static str` cannot be shortened.
- Higher-ranked trait bounds (`for<'a>`), and how `impl Trait` captures lifetimes in edition 2024.

## Before you start

- You finished [Lesson 06.4 — Interior mutability](04-interior-mutability.md).
- You know basic lifetimes from [Lesson 05.3 — Lifetimes](../05-rust-intermediate/03-lifetimes.md): the `'a` syntax, structs that hold references, and the idea that a reference must not outlive its data.
- You know closures ([Lesson 05.4](../05-rust-intermediate/04-closures.md)) and threads ([Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md)).
- Create a practice project: `cargo new code/p06-lifetimes`.

## New words

| Word | Meaning |
|---|---|
| **lifetime** | The part of a program during which a reference is valid. Written `'a`, `'b`, ... |
| **lifetime elision** | The compiler rules that let you leave lifetimes out of signatures in common cases. |
| **outlives** | `'long: 'short` means "`'long` lasts at least as long as `'short`". |
| **`'static`** | The longest lifetime: the whole run of the program. String literals have it. |
| **subtype** | A type that can be used where another type is expected. A longer lifetime is a subtype of a shorter one. |
| **variance** | The rule that says how a subtype relationship passes through a type: covariant (same direction), invariant (not at all), contravariant (reversed). |
| **API** (Application Programming Interface) | The functions and types that your code offers to other code. |
| **HRTB** (Higher-Ranked Trait Bound) | A bound with `for<'a>` that must hold for every lifetime `'a`, not for one chosen lifetime. |
| **scoped thread** | A thread started with `thread::scope`. It is guaranteed to finish before the scope ends, so it may borrow local data. |
| **self-referential struct** | A struct with a field that points into another field of the same struct. Safe Rust cannot express it. |

## 1. A quick recap

A reference points to data owned by someone else. A **lifetime** is the stretch of code where that reference is valid. The compiler compares the lifetime of every reference with the lifetime of the data and refuses the program if a reference could outlive its data:

```rust
fn main() {
    let reference;
    {
        let value = 5;
        reference = &value;
    }
    println!("{reference}");
}
```

```text
error[E0597]: `value` does not live long enough
 --> src/main.rs:5:21
  |
4 |         let value = 5;
  |             ----- binding `value` declared here
5 |         reference = &value;
  |                     ^^^^^^ borrowed value does not live long enough
6 |     }
  |     - `value` dropped here while still borrowed
7 |     println!("{reference}");
  |                --------- borrow later used here

For more information about this error, try `rustc --explain E0597`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

**What happens:**

1. `value` lives in the inner block only. `reference` is declared outside and holds `&value`.
2. At the closing `}` of the block, `value` is dropped while `reference` is still used afterwards.
3. The compiler marks exactly where the data is dropped (`dropped here while still borrowed`) and where the reference is used later (`borrow later used here`). These two marks are the key to reading almost every lifetime error.

Lifetime annotations never change how long data lives. They only **describe** relations to the compiler, so that it can check them.

## 2. The elision rules, exactly

You rarely write lifetimes because the compiler applies three rules to function signatures. If, after the rules, every output lifetime is known, you write nothing:

1. **Each reference parameter gets its own lifetime.** `fn f(a: &str, b: &str)` becomes `fn f<'1, '2>(a: &'1 str, b: &'2 str)`.
2. **If there is exactly one input lifetime, it is given to all output lifetimes.** `fn f(a: &str) -> &str` becomes `fn f<'1>(a: &'1 str) -> &'1 str`.
3. **If one of the parameters is `&self` or `&mut self`, its lifetime is given to all output lifetimes.**

```rust
// Rule 1: every reference parameter gets its own lifetime.
// Rule 2: if there is exactly one input lifetime, the output gets it.
fn first_word(text: &str) -> &str {
    text.split_whitespace().next().unwrap_or("")
}

// The same function with the lifetimes written out:
#[allow(clippy::needless_lifetimes)] // Clippy would remove them; we want to see them
fn first_word_explicit<'a>(text: &'a str) -> &'a str {
    text.split_whitespace().next().unwrap_or("")
}

struct Library {
    name: String,
}

impl Library {
    // Rule 3: with `&self`, the output gets the lifetime of `self`.
    fn label(&self, prefix: &str) -> &str {
        if prefix.is_empty() {
            &self.name
        } else {
            &self.name[1..]
        }
    }
}

fn main() {
    println!("{}", first_word("hello brave world"));
    println!("{}", first_word_explicit("good morning"));

    let library = Library {
        name: String::from("Central"),
    };
    println!("{} {}", library.label(""), library.label("x"));
}
```

```text
hello
good
Central entral
```

**What happens:**

1. `first_word` has one input reference, so rule 2 applies. It is the same as `first_word_explicit`.
2. `label(&self, prefix: &str) -> &str` has two input lifetimes, but rule 3 says the output gets the lifetime of `self`. So the result borrows from the `Library`, not from `prefix`.
3. The two calls print `Central` (no prefix) and `entral` (a slice that skips the first character).

If no rule can decide, you must write the lifetime yourself:

```text
error[E0106]: missing lifetime specifier
 --> src/main.rs:1:33
  |
1 | fn longest(a: &str, b: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `a` or `b`
help: consider introducing a named lifetime parameter
  |
1 | fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
  |           ++++     ++          ++          ++
```

Two inputs, no `self`: rules 2 and 3 do not apply, and the compiler cannot guess whether the result comes from `a` or from `b`.

## 3. Several lifetimes in one signature

The easy fix for the error above is to use the same lifetime `'a` for everything. That is safe, but it makes the signature stricter than needed. Look at a function that returns its first argument only:

```rust
// The result is tied to `x` only. `y` may be dropped earlier.
fn first_of<'a>(x: &'a str, _y: &str) -> &'a str {
    x
}

fn main() {
    let long_lived = String::from("I live long");
    let result;
    {
        let short_lived = String::from("short");
        result = first_of(&long_lived, &short_lived);
    } // `short_lived` is dropped here
    println!("{result}");
}
```

```text
I live long
```

**What happens:**

1. `x` has the named lifetime `'a`, and the result has `'a` too: "the result borrows from `x`".
2. `_y: &str` is an input without a name. By rule 1 it gets its own, separate lifetime. The result has nothing to do with it.
3. In `main`, `short_lived` is dropped before the `println!`, but `result` only depends on `long_lived`, so the program is accepted.

Now put `'a` on both parameters:

```rust
fn first_of<'a>(x: &'a str, _y: &'a str) -> &'a str {
    x
}

fn main() {
    let long_lived = String::from("I live long");
    let result;
    {
        let short_lived = String::from("short");
        result = first_of(&long_lived, &short_lived);
    }
    println!("{result}");
}
```

```text
error[E0597]: `short_lived` does not live long enough
  --> src/main.rs:10:40
   |
 9 |         let short_lived = String::from("short");
   |             ----------- binding `short_lived` declared here
10 |         result = first_of(&long_lived, &short_lived);
   |                                        ^^^^^^^^^^^^ borrowed value does not live long enough
11 |     }
   |     - `short_lived` dropped here while still borrowed
12 |     println!("{result}");
   |                ------ borrow later used here

For more information about this error, try `rustc --explain E0597`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

With a single `'a` for both parameters, the compiler must assume that the result may borrow from either one. The result lives as long as the **shorter** of the two, so `short_lived` has to live until the last use of `result`. It does not, so the compiler stops.

> 📏 **Convention:** Give a lifetime a name only for the references that really are related. Do not put `'a` everywhere "to make it compile". An unneeded relation makes your API harder to use. Clippy's `needless_lifetimes` lint helps you find unneeded names.

## 4. Lifetimes in structs and `impl` blocks

A struct that holds a reference needs a lifetime parameter. The struct cannot live longer than the data it points to. In an `impl` block you declare the parameter once (`impl<'a> Excerpt<'a>`) and use it in the methods:

```rust
// 'long: 'short reads "'long outlives 'short".
fn pick<'short, 'long: 'short>(a: &'short str, b: &'long str, take_b: bool) -> &'short str {
    if take_b { b } else { a }
}

// A struct holding a reference: the struct cannot outlive the data.
struct Excerpt<'a> {
    text: &'a str,
}

impl<'a> Excerpt<'a> {
    // Returns the SAME lifetime as the data, not the lifetime of `&self`.
    fn text(&self) -> &'a str {
        self.text
    }
}

fn main() {
    let a = String::from("short-lived");
    let result;
    {
        let b = String::from("long-lived");
        result = pick(&a, &b, true).len();
    }
    println!("{result}");

    let book = String::from("Call me Ishmael. Some years ago...");
    let kept: &str;
    {
        let excerpt = Excerpt { text: &book[..15] };
        kept = excerpt.text();
    } // the Excerpt is gone, but `kept` still points into `book`
    println!("{kept}");
}
```

```text
10
Call me Ishmael
```

**What happens:**

1. `pick<'short, 'long: 'short>` declares an **outlives bound**: `'long` lasts at least as long as `'short`. That allows returning `b` (a `&'long str`) where a `&'short str` is promised. `'long: 'short` is read "long outlives short".
2. In `main`, `pick` is called with `&a` and `&b`; the result's length (10) is taken inside the inner block, so everything is valid.
3. `Excerpt<'a>` holds a `&'a str`. The method `text(&self) -> &'a str` returns the data with the lifetime **`'a`** of the original text. Compare with the elision rule 3: a plain `fn text(&self) -> &str` would return a reference tied to `&self`.
4. That difference matters: `kept` is still usable after the inner block ends and `excerpt` is gone. If `text` returned `&str` tied to `&self`, `kept` could not outlive `excerpt`, and the program would not compile.

> 💡 **Tip:** When a struct holds a reference `&'a T`, methods that return parts of that data should return `&'a ...`, not a reference tied to `&self`. Then callers can keep the result after the struct is gone.

Here is the same idea in a real example: an iterator that gives out pieces of text it borrows.

```rust
// An iterator that hands out pieces of a string it BORROWS.
struct Words<'a> {
    rest: &'a str,
}

impl<'a> Iterator for Words<'a> {
    type Item = &'a str; // tied to the original text, not to &mut self

    fn next(&mut self) -> Option<&'a str> {
        let trimmed = self.rest.trim_start();
        if trimmed.is_empty() {
            return None;
        }
        let end = trimmed.find(' ').unwrap_or(trimmed.len());
        let (word, rest) = trimmed.split_at(end);
        self.rest = rest;
        Some(word)
    }
}

fn main() {
    let text = String::from("the quick  brown fox");
    let words: Vec<&str> = Words { rest: &text }.collect();
    println!("{words:?}");
    // All words can be kept at the same time: each points into `text`.
    println!("{} words, first = {}", words.len(), words[0]);
}
```

```text
["the", "quick", "brown", "fox"]
4 words, first = the
```

**What happens:**

1. `Words<'a>` stores the text that is left to read: `rest: &'a str`.
2. The `Iterator` trait says `fn next(&mut self) -> Option<Self::Item>`. The `Item` is `&'a str`: the lifetime of the **text**, not of the `&mut self` borrow. So the caller can collect all words into a `Vec` and keep them together; each word points into `text`.
3. Standard `Iterator` cannot return references into the iterator itself (that would tie each item to the `&mut self` borrow, and only one item could live at a time). Such "lending iterators" need a different design.

## 5. `'static` and the bound `T: 'static`

`'static` is the longest lifetime: the entire run of the program. String literals have it, because they are stored inside the program file.

The bound `T: 'static` means something slightly different: "**`T` contains no references that can expire**". It does *not* mean "`T` lives forever". An owned `String` satisfies `T: 'static`: it has no borrowed data, and it can live as long as its owner wants.

```rust
// `T: 'static` means: T contains no borrowed data that could expire.
fn needs_static<T: 'static>(value: T) -> T {
    value
}

fn main() {
    let owned = String::from("I own my data"); // no references inside: 'static
    let literal: &'static str = "text baked into the program";
    let number = 42;

    println!("{}", needs_static(owned)); // the String is MOVED in; it is not a reference
    println!("{}", needs_static(literal));
    println!("{}", needs_static(number));
}
```

```text
I own my data
text baked into the program
42
```

A reference to a local variable does not satisfy it:

```rust
fn needs_static<T: 'static>(value: T) -> T {
    value
}

fn main() {
    let owned = String::from("local");
    let borrowed: &str = &owned; // a reference to a local variable
    println!("{}", needs_static(borrowed));
}
```

```text
error[E0597]: `owned` does not live long enough
 --> src/main.rs:7:26
  |
6 |     let owned = String::from("local");
  |         ----- binding `owned` declared here
7 |     let borrowed: &str = &owned; // a reference to a local variable
  |                          ^^^^^^ borrowed value does not live long enough
8 |     println!("{}", needs_static(borrowed));
  |                    ---------------------- argument requires that `owned` is borrowed for `'static`
9 | }
  | - `owned` dropped here while still borrowed
  |
note: requirement that the value outlives `'static` introduced here
 --> src/main.rs:1:20
  |
1 | fn needs_static<T: 'static>(value: T) -> T {
  |                    ^^^^^^^
```

> ⚠️ **Common mistake:** When the compiler suggests `'static`, do not add it just to silence the error. Usually the data should be owned (`String` instead of `&str`) or the reference should get a real lifetime.

The most important place where you meet `'static` is `thread::spawn`. A new thread may run longer than the function that started it, so the closure must not borrow local data:

```rust
use std::thread;

fn main() {
    let names = vec![String::from("Ana"), String::from("Bo")];
    let handle = thread::spawn(|| {
        println!("{}", names.len());
    });
    handle.join().unwrap();
}
```

```text
error[E0373]: closure may outlive the current function, but it borrows `names`, which is owned by the current function
 --> src/main.rs:5:32
  |
5 |     let handle = thread::spawn(|| {
  |                                ^^ may outlive borrowed value `names`
6 |         println!("{}", names.len());
  |                        ----- `names` is borrowed here
  |
note: function requires argument type to outlive `'static`
 --> src/main.rs:5:18
  |
5 |       let handle = thread::spawn(|| {
  |  __________________^
6 | |         println!("{}", names.len());
7 | |     });
  | |______^
help: to force the closure to take ownership of `names` (and any other referenced variables), use the `move` keyword
  |
5 |     let handle = thread::spawn(move || {
  |                                ++++

For more information about this error, try `rustc --explain E0373`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

There are two fixes. You can move the data into the thread (`move ||`, then the thread owns it). Or, if the thread should only borrow, use a **scoped thread**. `thread::scope` guarantees that every thread inside it finishes before the scope ends, so the borrow is safe:

```rust
use std::thread;

fn main() {
    let numbers = [1, 2, 3, 4, 5, 6, 7, 8];
    let (left, right) = numbers.split_at(4);

    // Scoped threads may BORROW local data: the scope ends before `numbers` does.
    let (sum_left, sum_right) = thread::scope(|s| {
        let a = s.spawn(|| left.iter().sum::<i32>());
        let b = s.spawn(|| right.iter().sum::<i32>());
        (a.join().unwrap(), b.join().unwrap())
    });

    println!(
        "left = {sum_left}, right = {sum_right}, total = {}",
        sum_left + sum_right
    );
    println!("numbers is still ours: {}", numbers.len());
}
```

```text
left = 10, right = 26, total = 36
numbers is still ours: 8
```

**What happens:**

1. `split_at(4)` divides the vector into two borrowed halves.
2. `thread::scope(|s| { ... })` opens a scope. `s.spawn(...)` starts a thread that is allowed to borrow `left` and `right`.
3. `a.join().unwrap()` waits for the thread and returns the sum. The scope would also wait for any thread that you did not join.
4. After the scope, `numbers` is fully usable again: all borrows ended.

## 6. Subtyping and variance

If `'long` outlives `'short`, a `&'long str` can be used where a `&'short str` is expected. We say that `&'long T` is a **subtype** of `&'short T`. Lifetimes make this subtyping happen automatically.

```rust
// A shared reference is COVARIANT: a longer lifetime can stand in for a shorter one.
#[allow(clippy::needless_lifetimes)] // the name 'short is only here for teaching
fn print_it<'short>(text: &'short str) {
    println!("{text}");
}

fn main() {
    let forever: &'static str = "lives for the whole program";
    print_it(forever); // &'static str is accepted where &'short str is expected

    let owned = String::from("a local string");
    let shorter: &str = &owned;
    print_it(shorter);
}
```

```text
lives for the whole program
a local string
```

**What happens:** `print_it` accepts a reference of *some* short lifetime. A `&'static str` is shortened automatically to that lifetime, so the call is fine. Nothing is copied or converted at run time; it is purely a compile-time idea.

Whether this shortening is allowed inside a bigger type is decided by **variance**. There are three kinds:

| Variance | Meaning | Example |
|---|---|---|
| **Covariant** | A longer lifetime may replace a shorter one inside the type. | `&'a T`, `Box<T>`, `Vec<T>`, `Option<T>` |
| **Invariant** | No replacement is allowed in either direction; the lifetime must match exactly. | `&mut T` (in `T`), `Cell<T>`, `RefCell<T>` |
| **Contravariant** | The opposite direction: a shorter lifetime may replace a longer one. | Function arguments: `fn(&'a str)` is contravariant in `'a` |

Why is `&mut T` invariant in `T`? Because you can write through it. Suppose Rust allowed `&mut &'static str` to be treated as `&mut &'short str`. You could then store a short-lived reference through it, and the original variable (which promises `'static`) would end up with a dangling reference. The compiler prevents this:

```rust
fn assign<'a>(target: &mut &'a str, value: &'a str) {
    *target = value;
}

fn main() {
    let mut slot: &'static str = "static text";
    let local = String::from("local text");
    assign(&mut slot, &local);
    println!("{slot}");
}
```

```text
error[E0597]: `local` does not live long enough
  --> src/main.rs:8:23
   |
 6 |     let mut slot: &'static str = "static text";
   |                   ------------ type annotation requires that `local` is borrowed for `'static`
 7 |     let local = String::from("local text");
   |         ----- binding `local` declared here
 8 |     assign(&mut slot, &local);
   |                       ^^^^^^ borrowed value does not live long enough
 9 |     println!("{slot}");
10 | }
   | - `local` dropped here while still borrowed

For more information about this error, try `rustc --explain E0597`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

**What happens:**

1. `slot` has the type `&'static str`. `&mut slot` is a `&mut &'static str`.
2. `assign<'a>` requires that both the target and the value use the *same* `'a`. Because `&mut` is invariant, `'a` must be exactly `'static` for the target.
3. The value `&local` must then also be `'static`, which is impossible, so the compiler reports `local does not live long enough`.

You will not often write variance, but now error messages such as "type annotation requires that ... is borrowed for `'static`" make sense.

## 7. Higher-ranked trait bounds: `for<'a>`

Look at this bound: `F: Fn(&str) -> &str`. A closure of this kind must work for **every** lifetime of the argument: today it may be called with a string that lives for one second, tomorrow with a string that lives for an hour. The long form of the bound is:

```rust
F: for<'a> Fn(&'a str) -> &'a str
```

`for<'a>` reads "for all lifetimes `'a`". It is a **higher-ranked trait bound** (HRTB). Most of the time the compiler writes it for you when you use the shorthand.

```rust
// `for<'a>` reads "for every lifetime 'a".
fn apply<F>(f: F) -> usize
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    let first = String::from("  padded  ");
    let second = String::from("  other");
    // f is called with two different lifetimes, one per local String
    f(&first).len() + f(&second).len()
}

// The shorthand `Fn(&str) -> &str` means exactly the same thing.
fn apply_short<F: Fn(&str) -> &str>(f: F) -> usize {
    f("  x  ").len()
}

// A common real use: "a reference to C can be iterated, whatever its lifetime".
fn total<C>(collection: &C) -> i32
where
    for<'a> &'a C: IntoIterator<Item = &'a i32>,
{
    collection.into_iter().sum()
}

fn main() {
    println!("{}", apply(|s| s.trim()));
    println!("{}", apply_short(|s| s.trim()));
    println!("{}", total(&vec![1, 2, 3]));
    println!("{}", total(&[10, 20]));
}
```

```text
11
1
6
30
```

**What happens:**

1. `apply` calls `f` twice with two different local strings. Each call has its own lifetime, so `f` must work for all of them. That is why `for<'a>` is needed.
2. `apply_short` uses the shorthand `Fn(&str) -> &str`. It means exactly the same.
3. `total` uses an HRTB that you must write by hand: "for every lifetime `'a`, a `&'a C` can be turned into an iterator over `&'a i32`". It works for `Vec<i32>` and for arrays.
4. The numbers: `"  padded  "` trims to 6 characters and `"  other"` to 5, so 11; `"x"` gives 1; 1 + 2 + 3 = 6; 10 + 20 = 30.

If you forget the HRTB where it is needed, the compiler tells you:

```text
error[E0637]: `&` without an explicit lifetime name cannot be used here
 --> src/main.rs:3:5
  |
3 |     &C: IntoIterator<Item = &i32>,
  |     ^ explicit lifetime name needed here
  |
help: consider introducing a higher-ranked lifetime here
  |
3 |     for<'a> &'a C: IntoIterator<Item = &i32>,
  |     +++++++  ++
```

Closures have a special trap. If you define a closure in a separate `let` with an annotated parameter, the compiler does not know that its result is tied to the argument, and it cannot make the closure higher-ranked:

```rust
fn apply<F>(f: F) -> usize
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    f("  padded  ").len()
}

fn main() {
    let first_word = |s: &str| s.split(' ').next().unwrap_or("");
    println!("{}", apply(first_word));
}
```

```text
error: lifetime may not live long enough
 --> src/main.rs:9:32
  |
9 |     let first_word = |s: &str| s.split(' ').next().unwrap_or("");
  |                          -   - ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ returning this value requires that `'1` must outlive `'2`
  |                          |   |
  |                          |   return type of closure is &'2 str
  |                          let's call the lifetime of this reference `'1`
```

The fix is to write the closure directly where it is passed (`apply(|s| s.split(' ').next().unwrap_or(""))`), because then the expected signature is known and the compiler infers it correctly. You will use this in Exercise 3.

## 8. `impl Trait` and captured lifetimes (edition 2024)

A function can return `impl Trait`: a hidden type that you do not name ([Lesson 06.1](01-trait-objects-and-dynamic-dispatch.md)). The question is: which lifetimes may the hidden type contain? In **edition 2024** the answer is simple: **all** lifetimes that are in scope, including the ones from the parameters. In edition 2021 you had to write `+ '_` or a `use<'_>` bound, or you got error `E0700`.

```rust
// Edition 2024: `impl Trait` in the return type captures ALL lifetimes in scope.
fn letters(text: &str) -> impl Iterator<Item = char> {
    text.chars()
}

// `use<>` says: capture nothing. The result is independent of `text`.
fn owned_letters(text: &str) -> impl Iterator<Item = char> + use<> {
    text.chars().collect::<Vec<char>>().into_iter()
}

fn main() {
    let upper: String = letters("abc").map(|c| c.to_ascii_uppercase()).collect();
    println!("{upper}");

    let iter = {
        let temp = String::from("temp");
        owned_letters(&temp) // `temp` is dropped after this block
    };
    println!("{}", iter.count());
}
```

```text
ABC
4
```

**What happens:**

1. `letters` returns `text.chars()`, which borrows `text`. The return type does not mention a lifetime, but in edition 2024 the hidden type is allowed to capture the lifetime of `text` anyway.
2. `owned_letters` returns an iterator over a new `Vec<char>`. It does not need `text` after it returns. The bound `+ use<>` says "capture no lifetimes". That lets the caller drop `temp` while still holding the iterator, as `main` does.
3. In edition 2021 the first function does not compile (`E0700: hidden type for impl Iterator captures lifetime that does not appear in bounds`). That is the old rule that 2024 replaced.

## 9. What lifetimes cannot do

- **Self-referential structs.** A struct that owns a `String` and also holds a `&str` into that `String` cannot be written with plain lifetimes: the reference would need a lifetime that is "as long as the struct itself", and moving the struct would invalidate the pointer. Solutions: store indexes (ranges) instead of references, store the data and the view separately, or, in advanced cases, use `Pin` (Lesson 06.9) and `unsafe` code. Crates such as `ouroboros` package the tricks.
- **Reading from the future.** Lifetimes are about "valid until" and never "valid from". They cannot describe when a value will be created.

## 10. A decoder for lifetime errors

| You see | It means | First thing to try |
|---|---|---|
| `E0106 missing lifetime specifier` | The elision rules cannot pick an output lifetime. | Name the lifetime and tie the result to the right parameter, or return an owned value (`String`). |
| `E0597 ... does not live long enough` | A reference is used after its data was dropped. | Look at the two marks: where it is dropped and where it is used. Make the data live longer or copy it. |
| `E0515 cannot return reference to local variable` | You return a reference to something created inside the function. | Return the owned value (`String`, `Vec`) instead. |
| `E0373 closure may outlive the current function` | A closure that may run later borrows a local variable. | Use `move`, or a scoped thread. |
| `lifetime may not live long enough` | Your signature promises a longer lifetime than the code can give. | Add or fix a lifetime relation (`'a: 'b`) or change the returned type. |
| `E0521 borrowed data escapes outside of function` | A reference tries to reach somewhere that requires `'static`. | Own the data (clone it) or pass an owned value. |

The most common one in everyday code:

```text
error[E0515]: cannot return reference to local variable `text`
 --> src/main.rs:3:5
  |
3 |     &text
  |     ^^^^^ returns a reference to data owned by the current function

For more information about this error, try `rustc --explain E0515`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

## Common mistakes

### 1. One `'a` on everything

Makes signatures too strict. Use separate lifetimes for unrelated references, or let elision do it.

### 2. Returning a reference tied to `&self` when the data has its own lifetime

`fn text(&self) -> &str` ties the result to the `&self` borrow. Return `&'a str` if the struct holds `&'a str`.

### 3. Adding `'static` to make an error disappear

`T: 'static` means "no borrowed data". Own the data instead, or use a scoped thread when you just want to share a borrow.

### 4. Confusing `T: 'static` with "lives forever"

An owned `String` is `'static`, and it is freed when its owner drops it.

### 5. Closures defined in a separate `let` with `&` arguments

The signature is not inferred as higher-ranked. Write the closure at the place where it is passed.

## Try it yourself

1. Open `~/projects/rust/code/p06-lifetimes`.
2. Copy `words.rs` (section 4) into `src/main.rs` and run it.
3. Add a method `fn remaining(&self) -> &'a str` to `Words<'a>` that returns the unread rest of the text without leading spaces (`self.rest.trim_start()`). In `main`, call `next()` twice on a `let mut words = Words { rest: &text };` and print `words.remaining()`. Expect `brown fox`.
4. Change the return type of `remaining` to `&str` (without `'a`) and try to keep the result after `words` has gone out of scope (create `words` inside an inner block and use the result outside). Read the error: that is the lifetime of `&self` at work. Restore `&'a str`.
5. Run `cargo fmt` and `cargo clippy`, and commit: `git add code/p06-lifetimes` and `git commit -m "feat(p06): borrowed word iterator"`.

## Exercises

### Exercise 1 — Two lifetimes (easy)

Write `fn pick_primary<'a>(primary: &'a str, _fallback: &str) -> &'a str` that returns the first argument. Write a test where `fallback` is created in an inner block and dropped before the result is used.

<details>
<summary>Hint</summary>

The second parameter has its own anonymous lifetime (elision rule 1), so the result is not tied to it. In the test, declare `let result;` before the block and assign inside.

</details>

<details>
<summary>Answer</summary>

```rust
// The result is borrowed from `primary` only.
fn pick_primary<'a>(primary: &'a str, _fallback: &str) -> &'a str {
    primary
}

fn main() {
    let name = String::from("Ana");
    let answer;
    {
        let temporary = String::from("temporary");
        answer = pick_primary(&name, &temporary);
    }
    println!("{answer}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn result_can_outlive_the_second_argument() {
        let primary = String::from("main");
        let result;
        {
            let fallback = String::from("fallback");
            result = pick_primary(&primary, &fallback);
        }
        assert_eq!(result, "main");
    }
}
```

```text
Ana
```

```text

running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: only `primary` is connected to the result. If you wrote `'a` on both parameters, the test would not compile (section 3).

</details>

### Exercise 2 — A config reader that returns long-lived values (medium)

Write `struct Config<'a>` that holds `Vec<&'a str>` (the lines of a text), with `parse(text: &'a str) -> Config<'a>` and `get(&self, key: &str) -> Option<&'a str>`. A line looks like `key = value`. Test that the value returned by `get` can still be used after the `Config` is dropped.

<details>
<summary>Hint</summary>

`text.lines().collect()` builds the vector. In `get`, use `line.split_once('=')`. The return type is `Option<&'a str>`, not `Option<&str>`: the value comes from the text, not from `self`.

</details>

<details>
<summary>Answer</summary>

```rust
struct Config<'a> {
    lines: Vec<&'a str>,
}

impl<'a> Config<'a> {
    fn parse(text: &'a str) -> Config<'a> {
        Config {
            lines: text.lines().collect(),
        }
    }

    // The returned &str borrows from the ORIGINAL text ('a), not from `self`.
    fn get(&self, key: &str) -> Option<&'a str> {
        self.lines.iter().find_map(|line| {
            let (k, v) = line.split_once('=')?;
            if k.trim() == key {
                Some(v.trim())
            } else {
                None
            }
        })
    }
}

fn main() {
    let text = String::from("host = example.org\nport = 8080");
    let port = {
        let config = Config::parse(&text);
        config.get("port")
    }; // `config` is dropped here, but `port` still points into `text`
    println!("{port:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn value_outlives_the_config() {
        let text = String::from("host = example.org\nport = 8080");
        let port = {
            let config = Config::parse(&text);
            config.get("port")
        };
        assert_eq!(port, Some("8080"));
    }

    #[test]
    fn missing_key_gives_none() {
        let text = String::from("a = 1");
        assert_eq!(Config::parse(&text).get("b"), None);
    }
}
```

```text
Some("8080")
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the `'a` in the return type says "this reference points into the original text". The `Config` can be dropped, but the text is alive, so the reference stays valid.

</details>

### Exercise 3 — A higher-ranked closure (medium)

Write `fn apply_to_all<'s, F>(items: &'s [String], f: F) -> Vec<&'s str> where F: for<'a> Fn(&'a str) -> &'a str`. It applies `f` to every item. Test it with `|s| s.trim()` and with `|s| s.split(' ').next().unwrap_or("")`.

<details>
<summary>Hint</summary>

`items.iter().map(|item| f(item)).collect()`. Write the closures directly in the call, not in a separate `let`.

</details>

<details>
<summary>Answer</summary>

```rust
fn apply_to_all<'s, F>(items: &'s [String], f: F) -> Vec<&'s str>
where
    F: for<'a> Fn(&'a str) -> &'a str,
{
    items.iter().map(|item| f(item)).collect()
}

fn main() {
    let items = vec![String::from("  a "), String::from("b  "), String::from("c")];
    let trimmed = apply_to_all(&items, |s| s.trim());
    println!("{trimmed:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn trims_every_item() {
        let items = vec![String::from("  a "), String::from("b  ")];
        assert_eq!(apply_to_all(&items, |s| s.trim()), vec!["a", "b"]);
    }

    #[test]
    fn works_with_another_closure() {
        let items = vec![String::from("hello world")];
        let result = apply_to_all(&items, |s| s.split(' ').next().unwrap_or(""));
        assert_eq!(result, vec!["hello"]);
    }
}
```

```text
["a", "b", "c"]
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `f` is called with the lifetime of each item, so it must be valid for every lifetime. The returned strings borrow from `items` (`'s`).

</details>

### Exercise 4 — A parallel sum without cloning (hard)

Write `fn parallel_sum(numbers: &[i64]) -> i64` that splits the slice in two halves and sums them in two scoped threads. Test it against the sequential sum for 101 numbers, for an empty slice and for a slice with one element.

<details>
<summary>Hint</summary>

`numbers.split_at(numbers.len() / 2)`, then `thread::scope(|scope| { let a = scope.spawn(|| left.iter().sum::<i64>()); ... })`. Join both handles and add the results.

</details>

<details>
<summary>Answer</summary>

```rust
use std::thread;

fn parallel_sum(numbers: &[i64]) -> i64 {
    let middle = numbers.len() / 2;
    let (left, right) = numbers.split_at(middle);
    thread::scope(|scope| {
        let a = scope.spawn(|| left.iter().sum::<i64>());
        let b = scope.spawn(|| right.iter().sum::<i64>());
        a.join().unwrap() + b.join().unwrap()
    })
}

fn main() {
    let data: Vec<i64> = (1..=1000).collect();
    println!("sum = {}", parallel_sum(&data));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sums_like_the_sequential_version() {
        let data: Vec<i64> = (1..=101).collect();
        assert_eq!(parallel_sum(&data), data.iter().sum::<i64>());
    }

    #[test]
    fn empty_and_single() {
        assert_eq!(parallel_sum(&[]), 0);
        assert_eq!(parallel_sum(&[5]), 5);
    }
}
```

```text
sum = 500500
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: scoped threads may borrow `left` and `right` because the scope waits for them. No `Arc`, no `clone`, no `'static`. For tiny inputs threads cost more than they save; measure before you parallelize ([Lesson 05.12](../05-rust-intermediate/12-benchmarking-basics.md)).

</details>

## Quiz

1. State the three lifetime elision rules.

<details>
<summary>Answer</summary>

(1) Each reference parameter gets its own lifetime. (2) If there is exactly one input lifetime, all output lifetimes get it. (3) If there is a `&self` or `&mut self` parameter, all output lifetimes get the lifetime of `self`.

</details>

2. Why is `fn longest(a: &str, b: &str) -> &str` an error?

<details>
<summary>Answer</summary>

There are two input lifetimes and no `self`, so no elision rule can pick the output lifetime. You must write it (for example `<'a>` on both parameters and the result) or return an owned `String`.

</details>

3. What does `'long: 'short` mean?

<details>
<summary>Answer</summary>

`'long` outlives `'short`: it lasts at least as long. A reference with lifetime `'long` can be used where `'short` is expected.

</details>

4. Does `T: 'static` mean that `T` lives for the whole program?

<details>
<summary>Answer</summary>

No. It means `T` has no borrowed data that could expire. An owned `String` satisfies it and is freed normally.

</details>

5. Why does `thread::spawn` require a `'static` closure, and how does `thread::scope` avoid it?

<details>
<summary>Answer</summary>

A spawned thread can outlive the function that created it, so it must not borrow local data. `thread::scope` guarantees that all threads started in the scope finish before the scope ends, so they may borrow local variables.

</details>

6. Why is `&mut T` invariant in `T`?

<details>
<summary>Answer</summary>

Because you can write through it. If `&mut &'static str` could be shortened to `&mut &'short str`, you could store a short-lived reference into a place that promises `'static`, creating a dangling reference.

</details>

7. What does `for<'a> Fn(&'a str) -> &'a str` mean?

<details>
<summary>Answer</summary>

The function or closure must work for every possible lifetime `'a` of its argument, and the result has the same lifetime as the argument. The shorthand `Fn(&str) -> &str` means the same.

</details>

8. In edition 2024, which lifetimes can a returned `impl Trait` capture, and how do you opt out?

<details>
<summary>Answer</summary>

All lifetimes in scope, including those of the parameters. To opt out, add `+ use<>` (or list the lifetimes you really want, such as `use<'a>`).

</details>

## Summary

- A lifetime describes how long a reference is valid. Annotations never extend life; they only describe relations so the compiler can check them.
- Elision rules: one lifetime per reference parameter; a single input lifetime goes to the output; `&self` gives its lifetime to the output.
- Use separate lifetimes for unrelated references. Return `&'a T` (not a borrow of `&self`) when the data comes from a reference the struct holds.
- `'long: 'short` is an outlives bound. `T: 'static` means "no borrowed data", not "lives forever".
- `thread::spawn` needs `'static`; `thread::scope` lets threads borrow local data.
- `&'a T` is covariant; `&mut T` and `Cell<T>` are invariant in `T`; function arguments are contravariant.
- `for<'a>` means "for all lifetimes". Write closures directly where they are passed so the compiler infers the higher-ranked signature.
- In edition 2024, `impl Trait` captures all lifetimes in scope; `use<>` opts out.

## Sources and further reading

- [The Rust Programming Language — 10.3 Validating References with Lifetimes](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html) — the basics that this lesson builds on, including the elision rules.
- [The Rust Reference — Lifetime elision](https://doc.rust-lang.org/reference/lifetime-elision.html) — the exact rules, including trait objects and `impl` headers.
- [The Rustonomicon — Subtyping and Variance](https://doc.rust-lang.org/nomicon/subtyping.html) — the full explanation of variance with the table of types.
- [The Rustonomicon — Higher-Rank Trait Bounds](https://doc.rust-lang.org/nomicon/hrtb.html) — why `for<'a>` exists.
- [The Rust Reference — Subtyping and variance](https://doc.rust-lang.org/reference/subtyping.html) — the formal version.
- [std::thread::scope](https://doc.rust-lang.org/std/thread/fn.scope.html) — scoped threads in the standard library.

---

[🏠 Start](../START-HERE.md) · [← 06.4 Interior mutability](04-interior-mutability.md) · [📋 Phase 06](README.md) · [06.6 Unsafe Rust and Miri →](06-unsafe-rust-and-miri.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
