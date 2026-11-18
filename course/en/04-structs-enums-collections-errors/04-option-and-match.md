<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 04.3 Enums](03-enums.md) · [📋 Phase 04](README.md) · [04.5 if let, let else and patterns →](05-if-let-let-else-and-patterns.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/04-structs-enums-collections-errors/04-option-and-match.md) <!-- nav -->

# Lesson 04.4 — Option and match

> **Phase 04 — Structs, enums, collections and errors** · Lesson 4 of 12 · ⏱ about 3 hours (2 sittings)

## What you will learn

- Why "there might be no value" is a big source of bugs, and how Rust solves it with `Option<T>`.
- How to create an `Option` and how to get the value out safely.
- The most useful `Option` methods: `unwrap_or`, `map`, `and_then`, `filter`, `ok_or`, `as_ref` and friends.
- `match` in depth: ranges, several patterns, guards, `@` bindings, tuples and nested patterns.
- Why `unwrap` is dangerous in real programs and what to use instead.

## Before you start

- You finished [Lesson 04.3 — Enums](03-enums.md). You know variants with data and a first form of `match`.
- You remember how a `String` is moved ([Lesson 03.3 — Move, copy and clone](../03-ownership-and-borrowing/03-move-copy-and-clone.md)) and what `&` references are ([Lesson 03.4](../03-ownership-and-borrowing/04-references-and-borrowing.md)).

## New words

| Word | Meaning |
|---|---|
| **`Option<T>`** | An enum with two variants: `Some(value)` (there is a value of type `T`) and `None` (there is no value). |
| **`T`** (type parameter) | A placeholder for "any type". `Option<i32>` means "an optional `i32`"; `Option<String>` means "an optional `String`". You will write your own in [Lesson 05.1 — Generics](../05-rust-intermediate/01-generics.md). |
| **null** | A special "no value" marker used by many languages. Using it by mistake crashes programs. Rust has no null. |
| **`unwrap`** | A method that returns the value inside `Some`, or **panics** (stops the program with an error) on `None`. |
| **panic** | A deliberate crash of the program (or of the current thread) with an error message. |
| **`expect`** | Like `unwrap`, but you give the panic message. |
| **closure** | A small unnamed function written as `\|x\| x + 1`. You will learn them properly in [Lesson 05.4 — Closures](../05-rust-intermediate/04-closures.md); here you can read `\|w\| w * 2` as "a function that takes `w` and returns `w * 2`". |
| **guard** | An extra `if` condition on a `match` arm: `x if x > 5 => ...`. |
| **binding** | Giving a name to a part of a matched value, such as `n` in `Some(n)`. |
| **range pattern** | A pattern that matches a span of values: `1..=9` means 1 to 9 including 9. |
| **partial move** | Moving only a part (a field or a payload) out of a value, which leaves the whole value unusable. |
| **`Result`** | The enum for operations that can fail with an error: `Ok(value)` or `Err(error)`. Covered in [Lesson 04.10](10-error-handling.md). |

## 1. The problem: "no value"

Many functions can fail to produce an answer. Finding a word in text: the word may be absent. Dividing: the divisor may be zero. Asking a list for its fifth element: the list may have three.

Older languages use a special marker, often called **null**, for "no value". Every variable might secretly be null, and using such a variable crashes the program. Tony Hoare, who introduced null into a language in 1965, later called it his "billion-dollar mistake". The problem is that the type does not warn you: a variable of type "number" may or may not hold a number, and nothing in the code says which.

Rust has no null. Instead, "might be missing" is part of the **type**. The standard library defines an enum:

```text
enum Option<T> {
    None,
    Some(T),
}
```

`T` stands for "any type". `Option<i32>` is either `Some(5)` (an `i32` is there) or `None` (no `i32`). `Option` is so common that `Option`, `Some` and `None` are available everywhere without any `use`.

Because `Option<i32>` and `i32` are different types, the compiler **forces** you to deal with the "no value" case before you can use the number.

```rust
fn checked_div(a: i32, b: i32) -> Option<i32> {
    if b == 0 {
        None
    } else {
        Some(a / b)
    }
}

fn main() {
    for (a, b) in [(10, 2), (7, 0), (-9, 3)] {
        match checked_div(a, b) {
            Some(result) => println!("{a} / {b} = {result}"),
            None => println!("{a} / {b} is impossible"),
        }
    }
}
```

**What happens:**

1. `checked_div` returns `Option<i32>`. The signature itself tells the reader: "this may not give a number".
2. When the divisor is zero we return `None`. Otherwise we wrap the answer: `Some(a / b)`.
3. The `for` loop goes over an array of pairs. `(a, b)` takes each pair apart into two variables (a pattern, [Lesson 04.5](05-if-let-let-else-and-patterns.md)).
4. `match` has two arms, one per variant. `Some(result)` binds the inner number to the name `result`. The compiler would reject the `match` if the `None` arm were missing.

```text
10 / 2 = 5
7 / 0 is impossible
-9 / 3 = -3
```

### You cannot use an `Option` as if it were the value

In this example and the next one, only the lines inside `fn main() { … }` are shown; the line numbers in the error messages count from the top of `src/main.rs`, where line 1 is `fn main() {`.

```rust
let x: i32 = 5;
let y: Option<i32> = Some(5);
let sum = x + y;
```

```text
error[E0277]: cannot add `Option<i32>` to `i32`
 --> src/main.rs:4:17
  |
4 |     let sum = x + y;
  |                 ^ no implementation for `i32 + Option<i32>`
  |
  = help: the trait `Add<Option<i32>>` is not implemented for `i32`
```

(The real message continues with a list of other types that support `+`; it is shortened here.) This is the whole point: the compiler stops you from adding a possibly missing number. You must first take the number out, and decide what to do when it is not there.

### `None` needs a type

```rust
let nothing = None;
```

```text
error[E0282]: type annotations needed for `Option<_>`
 --> src/main.rs:2:9
  |
2 |     let nothing = None;
  |         ^^^^^^^   ---- type must be known at this point
  |
help: consider giving `nothing` an explicit type, where the type for type parameter `T` is specified
  |
2 |     let nothing: Option<T> = None;
  |                +++++++++++
```

`None` alone does not say "an optional what?". Write `let nothing: Option<i32> = None;`. Usually the type is clear from later use, and no annotation is needed.

## 2. Getting the value out

### With `match`

`match` is the most general way and you saw it above. You can run any code in each arm and the arms give a value:

```text
let text = match maybe { Some(n) => format!("got {n}"), None => String::from("nothing") };
```

### With `unwrap`, `expect` and friends

For short cases there are methods. The safe ones give a fallback:

```rust
fn main() {
    let some: Option<i32> = Some(7);
    let none: Option<i32> = None;

    println!("{}", some.unwrap_or(0));
    println!("{}", none.unwrap_or(0));
    println!("{}", none.unwrap_or_default());
    println!("{} {}", some.is_some(), none.is_none());
    println!("{}", some.expect("a number must be present"));
}
```

**What happens:**

1. `unwrap_or(0)` returns the inner value, or `0` when there is none.
2. `unwrap_or_default()` returns the type's default value (`0` for numbers, an empty `String` for text, see `Default` in [Lesson 04.2](02-methods-and-associated-functions.md)).
3. `is_some()` and `is_none()` answer yes/no questions without giving the value.
4. `expect("...")` returns the value, and if there is none, it **panics** with your message. Here a value exists, so it is just `7`.

```text
7
0
0
true true
7
```

The dangerous ones are `unwrap()` and `expect()` on a `None`:

```rust
fn main() {
    let none: Option<i32> = None;
    println!("before");
    let n = none.unwrap();
    println!("after {n}");
}
```

```text
before

thread 'main' (3975970) panicked at src/main.rs:4:18:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

The program stops at that line. "after" is never printed. The number in parentheses is the thread id and changes on every run. With `expect("config must contain a port")` the message is your own text, which is far more helpful when you debug.

> 🔒 **Security:** a `panic` on input that comes from outside — a file, the keyboard, a network request — is a bug that an attacker can trigger on purpose to crash your program (a *denial of service*). Never `unwrap` on data you do not control. Use `match`, `unwrap_or`, or return the problem to the caller.

> 📏 **Convention:** `unwrap` and `expect` are fine in three places: in tests, in quick experiments, and when you can prove the value is there (then prefer `expect("why this cannot happen")` so the proof is written down). Everywhere else, handle `None`.

## 3. Working on the value without opening the box

Often you do not want to open the `Option` at all; you want to *transform* what is inside and keep the "maybe" for later. These methods take a **closure** — a tiny function. Read `|w| w * 2` as "given `w`, give `w * 2`".

```rust
fn main() {
    let width: Option<u32> = Some(4);
    let nothing: Option<u32> = None;

    // map: change the value inside, if there is one
    println!("{:?}", width.map(|w| w * 2));
    println!("{:?}", nothing.map(|w| w * 2));

    // and_then: the step itself may give None
    let half = |n: u32| if n % 2 == 0 { Some(n / 2) } else { None };
    println!("{:?}", Some(10).and_then(half));
    println!("{:?}", Some(7).and_then(half));

    // filter: keep the value only when a test passes
    println!("{:?}", Some(15).filter(|n| *n > 10));
    println!("{:?}", Some(5).filter(|n| *n > 10));

    // ok_or: turn None into an error value (Result)
    let r1: Result<u32, &str> = width.ok_or("no width given");
    let r2: Result<u32, &str> = nothing.ok_or("no width given");
    println!("{:?} {:?}", r1, r2);

    // unwrap_or_else: compute the fallback only when needed
    println!("{}", nothing.unwrap_or_else(|| 10 + 20));
}
```

**What happens:**

1. `map` applies the function to the inside of a `Some` and wraps the result again. `None` stays `None`; the function does not run.
2. `and_then` is for a step that itself may fail. `half` returns `Some` for even numbers and `None` for odd ones. With `map` you would get a messy `Option<Option<u32>>`; `and_then` flattens it.
3. `filter` turns `Some(x)` into `None` when `x` fails the test. In the closure `n` is a reference to the number, so `*n` reads the value.
4. `ok_or("...")` converts `Option` into `Result`: `Some(v)` becomes `Ok(v)`, and `None` becomes `Err("...")`. This is the bridge to error handling ([Lesson 04.10](10-error-handling.md)).
5. `unwrap_or_else(|| ...)` runs the closure only when the value is missing. Use it instead of `unwrap_or` when the fallback is expensive to compute.

```text
Some(8)
None
Some(5)
None
Some(15)
None
Ok(4) Err("no width given")
30
```

> 💡 **Tip:** you do not need to memorise all methods. Remember the idea (`Option` has many helpers) and look up the list in the documentation of [`Option`](https://doc.rust-lang.org/std/option/enum.Option.html) when you need one. RustRover's code completion also shows them after you type a dot.

## 4. Where `Option` comes from

You will meet `Option` constantly in the standard library:

```rust
fn main() {
    let numbers = [10, 20, 30];

    println!("{:?}", numbers.get(1));
    println!("{:?}", numbers.get(5));
    println!("{:?}", numbers.first());
    println!("{:?}", "hello".find('l'));
    println!("{:?}", "hello".find('z'));
    println!("{:?}", '7'.to_digit(10));
    println!("{:?}", 'x'.to_digit(10));
    println!("{:?}", "42".parse::<i32>().ok());
    println!("{:?}", "4x2".parse::<i32>().ok());
}
```

**What happens:**

1. `numbers.get(index)` returns `Some(&element)`, or `None` when the index is too large. Compare with `numbers[5]`, which **panics**. Use `get` when the index comes from outside.
2. `first()` is the first element, or `None` for an empty list.
3. `"hello".find('l')` returns the position (byte index) of the first match: `Some(2)`; a missing character gives `None`.
4. `to_digit(10)` converts a character to its numeric value in base 10, or `None` if it is not a digit.
5. `parse::<i32>()` really returns a `Result`; `.ok()` converts `Ok(v)` into `Some(v)` and `Err(_)` into `None`, throwing the error details away.

```text
Some(20)
None
Some(10)
Some(2)
None
Some(7)
None
Some(42)
None
```

(`get` and `first` return a reference into the array; `{:?}` prints it as the plain value.)

## 5. `match` in depth

You have seen `match` on enums. A `match` can examine any value, and the patterns can be rich.

```rust
fn classify(n: i32) -> &'static str {
    match n {
        i32::MIN..=-1 => "negative",
        0 => "zero",
        1..=9 => "one digit",
        10 | 20 | 30 => "round number",
        x if x % 2 == 0 => "big and even",
        _ => "big and odd",
    }
}

fn describe_pair(pair: (i32, i32)) -> String {
    match pair {
        (0, 0) => String::from("origin"),
        (x, 0) => format!("on the x axis at {x}"),
        (0, y) => format!("on the y axis at {y}"),
        (x, y) => format!("at ({x}, {y})"),
    }
}

fn age_group(age: u32) -> String {
    match age {
        n @ 0..=12 => format!("child, {n} years"),
        n @ 13..=19 => format!("teenager, {n} years"),
        n => format!("adult, {n} years"),
    }
}

fn main() {
    for n in [-5, 0, 7, 20, 44, 51] {
        println!("{n}: {}", classify(n));
    }
    println!("{}", describe_pair((0, 0)));
    println!("{}", describe_pair((5, 0)));
    println!("{}", describe_pair((0, -2)));
    println!("{}", describe_pair((3, 4)));
    println!("{}", age_group(9));
    println!("{}", age_group(15));
    println!("{}", age_group(40));
}
```

**What happens:**

1. `i32::MIN..=-1` is a **range pattern**: from the smallest `i32` up to and including `-1`. `1..=9` is the same for 1 to 9. The `..=` means "up to and including". Ranges work for integers and characters.
2. `10 | 20 | 30` matches any of three values.
3. `x if x % 2 == 0` is an arm with a **guard**: the pattern `x` matches any number, but the arm is used only if the condition after `if` is true. Otherwise the next arm is tried. The compiler cannot look inside a guard, so a `match` still needs an arm without a guard at the end (`_`).
4. In `describe_pair` the value is a **tuple**, and the patterns are tuples too. A literal like `0` must match exactly; a name like `x` matches anything and binds it. The first arm that fits wins, so the order matters: `(0, 0)` is tested before `(x, 0)`.
5. `n @ 0..=12` is a **binding with a test**: check the range *and* keep the value under the name `n`. The last arm `n => ...` takes everything left.

```text
-5: negative
0: zero
7: one digit
20: round number
44: big and even
51: big and odd
origin
on the x axis at 5
on the y axis at -2
at (3, 4)
child, 9 years
teenager, 15 years
adult, 40 years
```

Patterns also nest. This is useful for an `Option` inside an `Option`, or an enum inside a struct:

```rust
fn main() {
    let maybe_number = Some(Some(3));
    match maybe_number {
        Some(Some(n)) if n > 2 => println!("big inner {n}"),
        Some(Some(n)) => println!("small inner {n}"),
        Some(None) => println!("outer present, inner missing"),
        None => println!("nothing"),
    }

    let scores = [Some(90), None, Some(55)];
    let mut total = 0;
    for score in scores {
        match score {
            Some(s) => total += s,
            None => println!("skipped a missing score"),
        }
    }
    println!("total = {total}");
}
```

**What happens:**

1. `Some(Some(n)) if n > 2` looks two levels deep and adds a guard. All four possible shapes are covered, so the compiler is satisfied.
2. In the loop, a `match` arm may be a statement-like expression: `total += s` has the type `()` (nothing), and so does the `println!` in the other arm. All arms of one `match` must have the **same type**.

```text
big inner 3
skipped a missing score
total = 145
```

> 💡 **Tip:** if you forget the `None` arm, the compiler is on your side:
>
> ```text
> error[E0004]: non-exhaustive patterns: `None` not covered
>  --> src/main.rs:3:11
>   |
> 3 |     match maybe {
>   |           ^^^^^ pattern `None` not covered
>   |
>   = note: the matched value is of type `Option<i32>`
> help: ensure that all possible cases are being handled by adding a match arm with a wildcard pattern or an explicit pattern as shown
>   |
> 4 ~         Some(n) => println!("got {n}"),
> 5 ~         None => todo!(),
>   |
> ```

## 6. `match` and ownership: the partial move

When you match on an `Option<String>` by value, the `String` is **moved** out into the pattern variable. Then the original is partly gone:

```rust
fn main() {
    let name: Option<String> = Some(String::from("Ferris"));

    match name {
        Some(n) => println!("hello, {n}"),
        None => println!("hello, stranger"),
    }

    println!("{:?}", name);
}
```

```text
error[E0382]: borrow of partially moved value: `name`
 --> src/main.rs:9:22
  |
5 |         Some(n) => println!("hello, {n}"),
  |              - value partially moved here
...
9 |     println!("{:?}", name);
  |                      ^^^^ value borrowed here after partial move
  |
  = note: partial move occurs because value has type `String`, which does not implement the `Copy` trait
help: borrow this binding in the pattern to avoid moving the value
  |
5 |         Some(ref n) => println!("hello, {n}"),
  |              +++
```

The compiler shows `ref n`, an old way to borrow inside a pattern. In modern Rust prefer one of these three fixes:

```rust
fn main() {
    let name: Option<String> = Some(String::from("Ferris"));

    // fix 1: match on a reference
    match &name {
        Some(n) => println!("hello, {n}"),
        None => println!("hello, stranger"),
    }

    // fix 2: as_ref() turns Option<String> into Option<&String>
    if let Some(n) = name.as_ref() {
        println!("length {}", n.len());
    }

    // fix 3: as_deref() gives Option<&str>
    let shown: &str = name.as_deref().unwrap_or("stranger");
    println!("shown = {shown}");

    println!("{:?}", name);
}
```

**What happens:**

1. `match &name` matches on a reference. Inside the arm `n` is `&String`, a borrow, so `name` stays whole.
2. `name.as_ref()` converts `&Option<String>` into `Option<&String>`: the same shape, but holding borrows. Now `if let Some(n) = ...` ([Lesson 04.5](05-if-let-let-else-and-patterns.md)) only borrows.
3. `name.as_deref()` goes one step further and gives `Option<&str>`, which combines nicely with `unwrap_or("stranger")`.
4. The final line shows that `name` was never moved.

```text
hello, Ferris
length 6
shown = Ferris
Some("Ferris")
```

Rule of thumb: **to look into an `Option`, borrow it** (`&opt`, `opt.as_ref()`, `opt.as_deref()`); to **consume** it, match it by value.

## 7. Testing code that returns `Option`

> 🧪 **Testing:** test both sides: a case that gives `Some(...)` and a case that gives `None`. Edge cases (empty text, empty list, zero) usually produce `None`, so they must be in your tests. `assert_eq!(f(x), None)` works because `Option<T>` supports `==` when `T` does and prints with `{:?}` when `T` does.

```rust
fn first_word_len(text: &str) -> Option<usize> {
    text.split_whitespace().next().map(|w| w.len())
}

fn main() {
    println!("{:?}", first_word_len("hello world"));
    println!("{:?}", first_word_len("   "));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_first_word() {
        assert_eq!(first_word_len("hello world"), Some(5));
    }

    #[test]
    fn empty_text_has_no_word() {
        assert_eq!(first_word_len(""), None);
        assert_eq!(first_word_len("   "), None);
    }
}
```

**What happens:**

1. `split_whitespace()` cuts the text at spaces and tabs; `.next()` takes the first piece, as an `Option` (an iterator ends with `None`; iterators come in [Lesson 05.5](../05-rust-intermediate/05-iterators.md)).
2. `.map(|w| w.len())` turns the word into its length, and keeps `None` as it is.
3. The tests cover the normal case and two "nothing there" cases.

```text
Some(5)
None
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Common mistakes

> ⚠️ **Common mistake:** using `unwrap()` "just to make it compile". It does compile, and then crashes at the worst moment. Decide what should happen on `None`: a default (`unwrap_or`), an error for the caller (`ok_or`), or a message (`match`).

> ⚠️ **Common mistake:** treating `Option<i32>` like `i32` (`E0277`, section 1). Open the box first.

> ⚠️ **Common mistake:** forgetting the `None` arm (`E0004`). The compiler always catches this.

> ⚠️ **Common mistake:** a partial move by matching an `Option<String>` by value, then using it again (`E0382`, section 6). Match on `&opt` or use `as_ref()`.

> ⚠️ **Common mistake:** using `_` as the last arm so that you never see new cases. For `Option` there are only two variants, so write `None` explicitly.

> ⚠️ **Common mistake:** indexing with `numbers[i]` when `i` comes from outside; this panics if `i` is too large. Use `numbers.get(i)`.

## Try it yourself

1. Create a project:

   ```text
   cd ~/projects/rust
   cargo new code/p04-option-match
   cd code/p04-option-match
   ```

2. Put this into `src/main.rs`:

   ```rust
   fn find_index(items: &[&str], wanted: &str) -> Option<usize> {
       for (i, item) in items.iter().enumerate() {
           if *item == wanted {
               return Some(i);
           }
       }
       None
   }

   fn main() {
       let fruits = ["apple", "pear", "plum"];
       for name in ["pear", "kiwi"] {
           match find_index(&fruits, name) {
               Some(i) => println!("{name} is at index {i}"),
               None => println!("{name} is not in the list"),
           }
       }
   }
   ```

   (`iter().enumerate()` walks over a list and gives pairs `(index, item)`. Iterators are the topic of Lesson 05.5; for now it is a ready-made tool.)

3. Run `cargo run`. Expected output:

   ```text
   pear is at index 1
   kiwi is not in the list
   ```

4. Change the `match` in `main` so that a missing fruit prints a message and a found fruit prints its index plus one (use `map`). Then add a test module with one `Some` and one `None` case.
5. Commit:

   ```text
   cd ~/projects/rust
   git add code/p04-option-match
   git commit -m "feat(p04): option and match practice"
   ```

## Exercises

### Exercise 1 — Classify a number (easy)

Write `fn classify(n: i32) -> &'static str` using a single `match` with range patterns: negative numbers give `"negative"`, `0` gives `"zero"`, `1` to `9` give `"small"`, and everything bigger gives `"large"`. Test the edges: `-1`, `0`, `1`, `9`, `10`.

<details>
<summary>Hint</summary>

`i32::MIN..=-1 => ...` matches all negative numbers. Use `_` for the last arm.

</details>

<details>
<summary>Answer</summary>

```rust
fn classify(n: i32) -> &'static str {
    match n {
        i32::MIN..=-1 => "negative",
        0 => "zero",
        1..=9 => "small",
        _ => "large",
    }
}

fn main() {
    for n in [-3, 0, 4, 9, 10, 1000] {
        println!("{n} is {}", classify(n));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn edges() {
        assert_eq!(classify(-1), "negative");
        assert_eq!(classify(0), "zero");
        assert_eq!(classify(1), "small");
        assert_eq!(classify(9), "small");
        assert_eq!(classify(10), "large");
    }
}
```

Output of `cargo run`:

```text
-3 is negative
0 is zero
4 is small
9 is small
10 is large
1000 is large
```

Why it works: ranges cover whole spans in one arm, and the tests sit exactly on the borders between ranges, where off-by-one mistakes hide.

</details>

### Exercise 2 — First even number (medium)

Write `fn first_even(numbers: &[i32]) -> Option<i32>` that returns the first even number, or `None`. Test: a list with an even number, a list without one, and an empty list.

<details>
<summary>Hint</summary>

Loop with `for &n in numbers` (the `&n` pattern copies the number out of the reference). Return `Some(n)` as soon as `n % 2 == 0`. After the loop return `None`.

</details>

<details>
<summary>Answer</summary>

```rust
fn first_even(numbers: &[i32]) -> Option<i32> {
    for &n in numbers {
        if n % 2 == 0 {
            return Some(n);
        }
    }
    None
}

fn main() {
    println!("{:?}", first_even(&[1, 3, 8, 10]));
    println!("{:?}", first_even(&[1, 3, 5]));
    println!("{:?}", first_even(&[]));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_the_first() {
        assert_eq!(first_even(&[1, 3, 8, 10]), Some(8));
    }

    #[test]
    fn none_when_no_even_number() {
        assert_eq!(first_even(&[1, 3, 5]), None);
        assert_eq!(first_even(&[]), None);
    }
}
```

Output of `cargo run`:

```text
Some(8)
None
None
```

Why it works: `return` leaves the function at the first hit. Reaching the end of the loop means no even number exists, which is exactly the case for `None`.

</details>

### Exercise 3 — Chain with `map` (medium)

You have `fn find_age(name: &str) -> Option<u32>` that knows `"alice"` (30) and `"bob"` (25) and returns `None` for others. Write `fn age_next_year(name: &str) -> Option<u32>` **without any `match` inside it**, using `map`. Test one known and one unknown name.

<details>
<summary>Hint</summary>

`find_age(name).map(|age| age + 1)`.

</details>

<details>
<summary>Answer</summary>

```rust
fn find_age(name: &str) -> Option<u32> {
    match name {
        "alice" => Some(30),
        "bob" => Some(25),
        _ => None,
    }
}

fn age_next_year(name: &str) -> Option<u32> {
    find_age(name).map(|age| age + 1)
}

fn main() {
    println!("{:?}", age_next_year("alice"));
    println!("{:?}", age_next_year("carol"));
    println!("{}", age_next_year("bob").unwrap_or(0));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn known_name() {
        assert_eq!(age_next_year("bob"), Some(26));
    }

    #[test]
    fn unknown_name() {
        assert_eq!(age_next_year("nobody"), None);
    }
}
```

Output of `cargo run`:

```text
Some(31)
None
26
```

Why it works: `map` runs the closure only for `Some`. For `None` the result is `None` automatically, so the "missing" case flows through without any extra code.

</details>

### Exercise 4 — Parse a pair (hard)

Write `fn parse_pair(text: &str) -> Option<(i32, i32)>` that accepts text like `"3,4"` or `" 10 , -2 "` and returns `Some((3, 4))` / `Some((10, -2))`. Anything else (`"3"`, `"a,b"`, `"1,"`, `""`) gives `None`. Test valid and invalid inputs.

<details>
<summary>Hint</summary>

`text.split_once(',')` returns `Option<(&str, &str)>`: the text before and after the first comma. Then match on a tuple of two parse results: `match (left.trim().parse::<i32>(), right.trim().parse::<i32>()) { (Ok(x), Ok(y)) => ..., _ => None }`. `trim()` removes spaces on both ends.

</details>

<details>
<summary>Answer</summary>

```rust
fn parse_pair(text: &str) -> Option<(i32, i32)> {
    match text.split_once(',') {
        Some((left, right)) => match (left.trim().parse::<i32>(), right.trim().parse::<i32>()) {
            (Ok(x), Ok(y)) => Some((x, y)),
            _ => None,
        },
        None => None,
    }
}

fn main() {
    for text in ["3,4", " 10 , -2 ", "3", "a,b", ""] {
        println!("{:?} -> {:?}", text, parse_pair(text));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_pairs() {
        assert_eq!(parse_pair("3,4"), Some((3, 4)));
        assert_eq!(parse_pair(" 10 , -2 "), Some((10, -2)));
    }

    #[test]
    fn invalid_pairs() {
        assert_eq!(parse_pair("3"), None);
        assert_eq!(parse_pair("a,b"), None);
        assert_eq!(parse_pair("1,"), None);
        assert_eq!(parse_pair(""), None);
    }
}
```

Output of `cargo run`:

```text
"3,4" -> Some((3, 4))
" 10 , -2 " -> Some((10, -2))
"3" -> None
"a,b" -> None
"" -> None
```

Why it works: the outer `match` handles "is there a comma?", the inner `match` on a tuple handles "did both halves parse?". The `_` arm collects every other combination. In [Lesson 04.10](10-error-handling.md) you will learn to say *why* it failed with `Result`.

</details>

## Quiz

1. What problem does `Option<T>` solve, and why does Rust not have `null`?

<details>
<summary>Answer</summary>

It makes "the value may be missing" visible in the type, so the compiler forces you to handle the missing case. With `null`, any variable may secretly be empty and crash the program at run time.

</details>

2. What are the two variants of `Option<T>`?

<details>
<summary>Answer</summary>

`Some(T)` holds a value; `None` holds nothing.

</details>

3. Why does `let x = None;` fail to compile?

<details>
<summary>Answer</summary>

The compiler cannot know the type `T` of the missing value (`E0282`). Give it a type: `let x: Option<i32> = None;`, or use `x` somewhere that fixes the type.

</details>

4. What is the difference between `unwrap_or(0)` and `unwrap()`?

<details>
<summary>Answer</summary>

`unwrap_or(0)` returns `0` when the value is missing. `unwrap()` panics when the value is missing.

</details>

5. When do you use `map` and when `and_then` on an `Option`?

<details>
<summary>Answer</summary>

`map` when the transformation always succeeds (`T -> U`). `and_then` when the transformation itself may give `None` (`T -> Option<U>`); it avoids a nested `Option<Option<U>>`.

</details>

6. What does `n @ 1..=9` do?

<details>
<summary>Answer</summary>

It tests whether the value is in the range 1 to 9 (inclusive) and, if so, binds the whole value to the name `n`.

</details>

7. Why does `match name { Some(n) => ..., None => ... }` on an `Option<String>` make `name` unusable afterwards, and how do you avoid it?

<details>
<summary>Answer</summary>

The `String` is moved out of the `Option` into `n` (a partial move). Match on a reference instead: `match &name`, or use `name.as_ref()` / `name.as_deref()`.

</details>

8. Name two places where `unwrap` is acceptable.

<details>
<summary>Answer</summary>

In tests, and in quick experiments. Also when the value is provably present (then prefer `expect` with a message explaining why).

</details>

## Summary

- `Option<T>` is an ordinary enum, `Some(T)` or `None`. It replaces `null` and puts "might be missing" into the type.
- You cannot use an `Option<T>` as a `T`; you must open it with `match`, `unwrap_or`, `if let` (next lesson) or other methods.
- `unwrap` and `expect` panic on `None`; keep them for tests and provable cases, never for outside input.
- `map`, `and_then`, `filter`, `ok_or`, `unwrap_or_else`, `as_ref` and `as_deref` let you work with `Option` without opening it.
- Many standard functions return `Option`: `get`, `first`, `find`, `to_digit`, `split_once`. Prefer `get` to `[i]` for outside indexes.
- `match` supports literals, ranges (`1..=9`), alternatives (`|`), guards (`if`), bindings (`n @ ...`), tuples and nesting; every arm has the same type, and the match must be exhaustive.
- To look into an `Option` without consuming it, borrow it: `&opt`, `opt.as_ref()`, `opt.as_deref()`.
- Test both `Some` and `None` cases and the edges between ranges.

## Sources and further reading

- [The Rust Programming Language — 6.1 Defining an Enum (the Option section)](https://doc.rust-lang.org/book/ch06-01-defining-an-enum.html) — the official explanation of `Option` and null.
- [The Rust Programming Language — 6.2 The match Control Flow Construct](https://doc.rust-lang.org/book/ch06-02-match.html) — `match`, patterns that bind values, and exhaustiveness.
- [The Rust Programming Language — 19.3 Pattern Syntax](https://doc.rust-lang.org/book/ch19-03-pattern-syntax.html) — the complete list of pattern forms (ranges, guards, `@`).
- [std::option::Option](https://doc.rust-lang.org/std/option/enum.Option.html) — the full list of `Option` methods with examples.
- [Rust by Example — Option](https://doc.rust-lang.org/rust-by-example/std/option.html) — runnable examples.
- [Rust by Example — match](https://doc.rust-lang.org/rust-by-example/flow_control/match.html) — runnable examples of the pattern forms from section 5.

---

[🏠 Start](../START-HERE.md) · [← 04.3 Enums](03-enums.md) · [📋 Phase 04](README.md) · [04.5 if let, let else and patterns →](05-if-let-let-else-and-patterns.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
