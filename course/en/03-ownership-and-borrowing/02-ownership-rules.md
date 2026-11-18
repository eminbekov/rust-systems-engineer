<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 03.1 Memory: the stack and the heap](01-memory-stack-and-heap.md) · [📋 Phase 03](README.md) · [03.3 Move, copy and clone →](03-move-copy-and-clone.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/03-ownership-and-borrowing/02-ownership-rules.md) <!-- nav -->

# Lesson 03.2 — The ownership rules

> **Phase 03 — Ownership and borrowing** · Lesson 2 of 8 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- The three ownership rules, in plain words.
- What a **scope** is and how Rust frees a value when its owner leaves the scope.
- In which order values are dropped, and how to *see* it happen.
- What happens to ownership when you pass a value to a function or return it from one.
- Why these rules remove leaks, dangling pointers and double frees.

## Before you start

- You finished [Lesson 03.1 — Memory: the stack and the heap](01-memory-stack-and-heap.md). You know that a `String` is a small handle on the stack plus letters on the heap, and that heap memory must be freed exactly once.
- You know how to write functions ([Lesson 02.4](../02-programming-basics/04-functions.md)) and how variables and blocks `{ }` work ([Lesson 02.1](../02-programming-basics/01-variables-and-mutability.md)).
- Create a project: `cargo new code/p03-ownership` (from `~/projects/rust`). Each example below can be pasted into `src/main.rs`.

## New words

| Word | Meaning |
|---|---|
| **value** | A piece of data in memory, such as the number `42` or the text `"hello"`. |
| **owner** | The variable that is responsible for a value. Only the owner may free it. |
| **ownership** | The set of rules that says who owns each value and when it is freed. |
| **scope** | The part of the code where a variable exists. In Rust, it starts at `let` and ends at the closing `}` of the block that contains it. |
| **drop** | To free a value (and give back everything it owns). Rust *drops* a value when its owner goes out of scope. |
| **move** | To transfer ownership from one variable to another. After a move, the old variable can no longer be used. (Lesson 03.3 studies moves in depth.) |
| **RAII** (Resource Acquisition Is Initialization) | The idea that a value owns a resource (memory, a file, a lock) and releases it automatically when the value is dropped. |
| **struct** | A way to make your own type that groups several values. It is used here only inside a small helper; Lesson 04.1 teaches it properly. |
| **trait** | A set of abilities that a type can have. `Drop` is the trait of "what to do when I am dropped". Lesson 05.2 teaches traits. |

## 1. The three rules

Here are the ownership rules. They are short, and you should be able to say them from memory:

> 1. **Every value has exactly one owner** — a variable.
> 2. **There can be only one owner at a time.**
> 3. **When the owner goes out of scope, the value is dropped** (freed).

An everyday picture: a value is a **hotel room**, and the owner has the **only key card**. You may give the card to someone else (then you cannot enter any more), and when the card holder checks out, the room is cleaned and becomes free. Two people can never hold the one master key at once, so nobody can clean the room while another person is still sleeping in it.

Compare with Lesson 03.1: the three memory bugs came from freeing at the wrong time or twice. Rule 3 says *when* to free (exactly once, when the owner goes away), and rule 2 says *who* does it (only one owner, so only one free).

## 2. Scope: where a variable lives

A **scope** is the region of code in which a variable is valid. In Rust, a block `{ ... }` creates a scope. A variable is valid from the line where it is created until the closing `}`.

```rust
fn main() {
    {
        let s = String::from("hello");
        println!("inside: {s}");
    } // s goes out of scope here

    println!("outside: {s}");
}
```

**What happens:**

1. The inner block starts. `let s = String::from("hello");` creates a `String` (three numbers on the stack, letters on the heap). `s` is its owner.
2. `println!` uses `s`. This is fine, because we are still in the scope.
3. The closing `}` ends the scope. `s` is gone, so Rust **drops** the `String`: the heap memory is freed.
4. The last `println!` tries to use `s` outside its scope. The compiler refuses:

```text
error[E0425]: cannot find value `s` in this scope
 --> src/main.rs:7:25
  |
7 |     println!("outside: {s}");
  |                         ^
  |
help: the binding `s` is available in a different scope in the same function
 --> src/main.rs:3:13
  |
3 |         let s = String::from("hello");
  |             ^
```

This error says that the name `s` does not exist here. The fix is to use `s` only inside the block, or to create the variable outside it.

## 3. Seeing the drop happen

Normally the drop is invisible. To *see* it, we use a small helper type that prints a message when it is dropped. You do not need to understand the helper yet. Treat it as a black box and just copy it:

```rust
struct Noisy(String);

impl Drop for Noisy {
    fn drop(&mut self) {
        println!("  drop: {}", self.0);
    }
}

fn noisy(name: &str) -> Noisy {
    Noisy(name.to_string())
}
```

In one sentence each:

- `struct Noisy(String);` makes a new type `Noisy` that holds one `String` (its name). Structs are Lesson 04.1.
- `impl Drop for Noisy { ... }` says: "when a `Noisy` is dropped, run this code". `Drop` is a *trait* (Lesson 05.2). The word `self.0` is the name stored inside.
- `noisy("a")` is a small helper function that builds a `Noisy` with the name `"a"`. (The `&str` in its signature is text borrowed from somewhere; Lesson 03.6 explains it.)

Now we can watch scopes:

```rust
fn main() {
    println!("start of main");
    let _first = noisy("first");
    {
        println!("  entering inner block");
        let _inner = noisy("inner");
        println!("  leaving inner block");
    }
    let _second = noisy("second");
    println!("end of main");
}
```

**What happens:**

1. `let _first = ...` creates a `Noisy`. A name that starts with `_` tells Rust "I know this variable is unused; do not warn me".
2. The inner block creates `_inner`. When the block ends, `_inner` goes out of scope, and its `drop` runs **at the closing brace**.
3. `_second` is created after that.
4. When `main` ends, `_second` and `_first` go out of scope and are dropped.

```text
start of main
  entering inner block
  leaving inner block
  drop: inner
end of main
  drop: second
  drop: first
```

Look at the order at the end of `main`: **`second` was dropped before `first`**. Variables are dropped in the **reverse order** of their creation, just like plates taken from the top of a stack. With three variables it is even clearer:

```rust
fn main() {
    let _a = noisy("a");
    let _b = noisy("b");
    let _c = noisy("c");
    println!("end of main");
}
```

```text
end of main
  drop: c
  drop: b
  drop: a
```

The reverse order is useful. If `b` was built using something from `a`, then `b` is dropped first, while `a` still exists.

> 📏 **Convention:** this "owner goes out of scope, the value is cleaned up automatically" behaviour is called **RAII**. It is not only for memory: a file is closed, a lock is released, and a network connection is shut down in the same way, when their owners are dropped. You never write `free` or `close` by hand in normal Rust code.

## 4. Passing a value to a function moves ownership

What happens when you pass a `String` to a function? The function gets a **new owner**: its parameter. The old variable gives up ownership. We say that the value is **moved**.

```rust
fn takes_ownership(item: Noisy) {
    println!("  inside takes_ownership: {}", item.0);
} // item is dropped here

fn gives_ownership() -> Noisy {
    println!("  inside gives_ownership");
    noisy("made-by-function")
}

fn takes_and_gives_back(item: Noisy) -> Noisy {
    println!("  inside takes_and_gives_back: {}", item.0);
    item
}

fn main() {
    let x = noisy("x");
    println!("calling takes_ownership");
    takes_ownership(x);
    println!("back in main");

    let y = gives_ownership();
    println!("main now owns {}", y.0);

    let z = takes_and_gives_back(y);
    println!("main owns {} again", z.0);
    println!("end of main");
}
```

**What happens:**

1. `x` owns a `Noisy` called "x".
2. `takes_ownership(x)` moves the value into the parameter `item`. Now `item` is the owner and `x` is empty. When the function ends, `item` goes out of scope, so the value is dropped **at the end of the function**, not in `main`.
3. `gives_ownership()` creates a value and returns it. The returned value is moved out to the caller: `y` becomes its owner. Nothing is dropped here.
4. `takes_and_gives_back(y)` moves the value in, and returns it right away. The last line `item` (no semicolon) is the return value, so the value moves out again, into `z`.
5. At the end of `main`, `z` owns the value and drops it. Variables `x` and `y` own nothing, so they drop nothing.

```text
calling takes_ownership
  inside takes_ownership: x
  drop: x
back in main
  inside gives_ownership
main now owns made-by-function
  inside takes_and_gives_back: made-by-function
main owns made-by-function again
end of main
  drop: made-by-function
```

Each value was dropped exactly once. Nothing was leaked, nothing was freed twice.

### What the compiler says if you use a moved value

The old variable is no longer the owner, so it must not be used. Rust checks this when compiling:

```rust
fn print_it(s: String) {
    println!("{s}");
}

fn main() {
    let name = String::from("Ada");
    print_it(name);
    println!("{name}");
}
```

```text
error[E0382]: borrow of moved value: `name`
 --> src/main.rs:8:16
  |
6 |     let name = String::from("Ada");
  |         ---- move occurs because `name` has type `String`, which does not implement the `Copy` trait
7 |     print_it(name);
  |              ---- value moved here
8 |     println!("{name}");
  |                ^^^^ value borrowed here after move
  |
note: consider changing this parameter type in function `print_it` to borrow instead if owning the value isn't necessary
 --> src/main.rs:1:16
  |
1 | fn print_it(s: String) {
  |    --------    ^^^^^^ this parameter takes ownership of the value
  |    |
  |    in this function
help: consider cloning the value if the performance cost is acceptable
  |
7 |     print_it(name.clone());
  |                  ++++++++
```

How to read it:

- `error[E0382]` is the error code. `E0382` is "use of a moved value". You can run `rustc --explain E0382` to read a long explanation.
- The labels point to three places: where the value was created (`move occurs because ...`), where it was moved (`value moved here`), and where you tried to use it (`value borrowed here after move`).
- The `note` suggests that the function should *borrow* instead of taking ownership. That is the real fix, and Lesson 03.4 teaches it.
- The `help` suggests `.clone()` (make a full copy). It works, but costs time and memory. Lesson 03.3 explains when it is right.

### Moving inside a loop

A move can also happen in a place you do not notice at first. A loop runs its body many times, but a value can be moved only once:

```rust
fn print_it(s: String) {
    println!("{s}");
}

fn main() {
    let name = String::from("Ada");
    for _ in 0..2 {
        print_it(name);
    }
}
```

```text
error[E0382]: use of moved value: `name`
 --> src/main.rs:8:18
  |
6 |     let name = String::from("Ada");
  |         ---- move occurs because `name` has type `String`, which does not implement the `Copy` trait
7 |     for _ in 0..2 {
  |     ------------- inside of this loop
8 |         print_it(name);
  |                  ^^^^ value moved here, in previous iteration of loop
```

The first pass moves `name` into `print_it`. On the second pass, `name` has no value. Rust sees this *before* the program runs.

## 5. Handing the value back is tedious

Until Lesson 03.4 we only have one tool: give the value away, and take it back if we still need it. It works:

```rust
fn add_exclamation(mut s: String) -> String {
    s.push('!');
    s
}

fn text_and_length(s: String) -> (String, usize) {
    let length = s.len();
    (s, length)
}

fn main() {
    let greeting = String::from("hello");
    let greeting = add_exclamation(greeting);
    println!("{greeting}");

    let (greeting, length) = text_and_length(greeting);
    println!("{greeting} has {length} bytes");
}
```

**What happens:**

1. `add_exclamation` takes ownership, changes the string (`mut s` means the parameter may be changed), and returns it. In `main`, `let greeting = ...` creates a new variable that shadows the old one (Lesson 02.1).
2. `text_and_length` wants to return two things, so it returns a tuple: the string and its length. We unpack the tuple with `let (greeting, length) = ...`.

```text
hello!
hello! has 6 bytes
```

This is correct, but clumsy: every function that only wants to *look* at a value must give it back. Rust solves this with **references**, and they are the topic of Lessons 03.4 and 03.5.

## 6. More facts about drops

**Assigning a new value drops the old one immediately.** The variable stays, but the value it owned is no longer needed:

```rust
fn main() {
    let mut item = noisy("old");
    println!("item is {}", item.0);
    println!("assigning a new value");
    item = noisy("new");
    println!("assigned; item is {}", item.0);
    println!("end of main");
}
```

```text
item is old
assigning a new value
  drop: old
assigned; item is new
end of main
  drop: new
```

**A collection owns its items.** When the owner of a `Vec` (a growable list, Lesson 04.7) is dropped, all the items inside it are dropped too, in order:

```rust
fn main() {
    let mut list = Vec::new();
    list.push(noisy("one"));
    list.push(noisy("two"));
    list.push(noisy("three"));
    println!("the list has {} items", list.len());
    println!("end of main");
}
```

```text
the list has 3 items
end of main
  drop: one
  drop: two
  drop: three
```

Ownership forms a tree:

```text
list (owner on the stack)
 └── heap buffer of the Vec
      ├── Noisy "one"   └── String "one"   (heap)
      ├── Noisy "two"   └── String "two"   (heap)
      └── Noisy "three" └── String "three" (heap)
```

Dropping the root drops every branch. That is why you rarely think about freeing in Rust: free the owner and everything below it goes with it.

**You can end a value's life early with `drop`.** The function `drop(x)` takes ownership of `x` and does nothing with it. The parameter goes out of scope at once, so the value is cleaned up right away:

```rust
fn main() {
    let big = noisy("big-resource");
    println!("using {}", big.0);
    drop(big);
    println!("the resource is already gone");
    println!("end of main");
}
```

```text
using big-resource
  drop: big-resource
the resource is already gone
end of main
```

After `drop(big)`, the name `big` is moved-from, so using it is an error (`E0382`, "borrow of moved value"). You do not need `drop` often. It is useful when a value holds something valuable, such as a big buffer, a file or a lock, and you want to release it before the end of the scope.

> 🔒 **Security:** automatic cleanup is also a safety feature. Secrets such as passwords and keys should not stay in memory longer than needed. Because a value is dropped at a known, predictable point, it is easy to reason about how long it lives. (Real secret-handling libraries go further and wipe the memory; Phase 10 touches this.)

> 🧪 **Testing:** a test can check that a function returns ownership correctly, simply by calling it and using the result. If the function swallowed the value, the test would not compile. Compile errors are the cheapest tests of all.

## Common mistakes

**1. Using a variable after passing it to a function by value.** This is `E0382`, shown above. Ask yourself: "Does this function need to *own* the value, or only look at it?" If it only looks, borrow (Lesson 03.4).

**2. Moving a value inside a loop.** Also `E0382`, shown above. Move the call out of the loop, or make the function borrow.

**3. Thinking that a value is dropped after its *last use*.** It is not. A value is dropped when its owner's **scope ends**, even if you stopped using it long ago. (Only *borrows*, in the next lessons, end at the last use.)

**4. Thinking shadowing drops the old value.** When you write `let x = ...;` twice, the first value is still alive until the end of the scope; the name is just hidden. Exercise 4 shows it.

**5. Reading `drop(x)` as "forget `x`" and then using `x` again.**

```rust
fn main() {
    let s = String::from("hello");
    drop(s);
    println!("{s}");
}
```

```text
error[E0382]: borrow of moved value: `s`
 --> src/main.rs:4:16
  |
2 |     let s = String::from("hello");
  |         - move occurs because `s` has type `String`, which does not implement the `Copy` trait
3 |     drop(s);
  |          - value moved here
4 |     println!("{s}");
  |                ^ value borrowed here after move
```

`drop` is a normal function. It takes ownership, so `s` is moved. The fix: do not use `s` after `drop(s)`.

## Try it yourself

1. In `~/projects/rust`, run `cargo new code/p03-ownership` (if you have not already). Open `src/main.rs`.
2. Paste the `Noisy` helper at the top of the file. Then add the "three variables" program (`_a`, `_b`, `_c`) as `main`.
3. Run `cargo run`. Check that the drop order is `c`, `b`, `a`.
4. Add an inner block `{ ... }` around `_b` and `_c`. Run again. Which drops moved earlier? Why?
5. Now write a function `fn consume(n: Noisy) { println!("consuming {}", n.0); }` and call it with `_a`. Where does `drop: a` appear in the output now?
6. Commit your work:

   ```bash
   git add code/p03-ownership
   git commit -m "p03: ownership experiments with Noisy"
   ```

## Exercises

### Exercise 1 — Predict the drop order (easy)

What does this program print? Write down your answer before you run it. (Use the `Noisy` helper from section 3.)

```rust
fn main() {
    let _a = noisy("a");
    {
        let _b = noisy("b");
        let _c = noisy("c");
        println!("inner block ends");
    }
    let _d = noisy("d");
    println!("main ends");
}
```

<details>
<summary>Hint</summary>

A scope's variables are dropped, in reverse order, at its closing brace. Do the inner block first, then `main`.

</details>

<details>
<summary>Answer</summary>

```text
inner block ends
  drop: c
  drop: b
main ends
  drop: d
  drop: a
```

Why it works: at the end of the inner block `_c` (the newest) is dropped first, then `_b`. `_a` lives in the outer scope, so it is dropped last, after `_d`.

</details>

### Exercise 2 — Fix the moved value (easy)

This program does not compile. Fix it **using only what you know from this lesson** (do not clone):

```rust
fn print_twice(s: String) {
    println!("{s}");
    println!("{s}");
}

fn main() {
    let message = String::from("ownership");
    print_twice(message);
    println!("length: {}", message.len());
}
```

<details>
<summary>Hint</summary>

If the function needs to give the value back, it can return it. Change the return type and use the returned value in `main`.

</details>

<details>
<summary>Answer</summary>

```rust
fn print_twice(s: String) -> String {
    println!("{s}");
    println!("{s}");
    s
}

fn main() {
    let message = String::from("ownership");
    let message = print_twice(message);
    println!("length: {}", message.len());
}
```

```text
ownership
ownership
length: 9
```

Why it works: ownership goes into `print_twice` and comes back out as the return value. The new `message` (which shadows the old one) is the owner again. In Lesson 03.4 you will see the simpler fix: borrow the string instead of taking it.

</details>

### Exercise 3 — Write a function and tests (medium)

Write `fn longer(a: String, b: String) -> String` that takes ownership of both strings and returns the longer one (the first one if they have the same length). The other string is dropped when the function ends. Write two tests: one for a clear winner and one for a tie.

<details>
<summary>Hint</summary>

Compare `b.len() > a.len()`. Use `if ... { b } else { a }` as the last expression of the function. In the tests, create the strings with `String::from(...)` and compare the result with `assert_eq!(result, "hello")` (a `String` can be compared with text).

</details>

<details>
<summary>Answer</summary>

```rust
fn longer(a: String, b: String) -> String {
    if b.len() > a.len() { b } else { a }
}

fn main() {
    let result = longer(String::from("short"), String::from("much longer"));
    println!("{result}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn returns_the_longer_one() {
        let result = longer(String::from("hi"), String::from("hello"));
        assert_eq!(result, "hello");
    }

    #[test]
    fn tie_returns_the_first() {
        let result = longer(String::from("aa"), String::from("bb"));
        assert_eq!(result, "aa");
    }
}
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: both parameters are owned by the function. It returns one of them (moving it out), and the other one is dropped at the closing brace of the function.

</details>

### Exercise 4 — Shadowing is not dropping (hard)

Predict the output. In which order are the four values dropped, and when?

```rust
fn main() {
    let x = noisy("first x");
    println!("the first x is {}", x.0);
    let x = noisy("second x");
    println!("shadowed; the current x is {}", x.0);

    let list = vec![noisy("p"), noisy("q")];
    println!("made the list");
    drop(list);
    println!("main ends");
}
```

<details>
<summary>Hint</summary>

Shadowing only hides a name. Both `Noisy` values are alive until the end of `main`. `drop(list)` runs before that and drops `p` then `q` (in order).

</details>

<details>
<summary>Answer</summary>

```text
the first x is first x
shadowed; the current x is second x
made the list
  drop: p
  drop: q
main ends
  drop: second x
  drop: first x
```

Why it works: `drop(list)` ends the list's life early, and it drops its items in order. The two `x` values are dropped at the end of `main` in reverse order of creation: "second x" first, then "first x". The shadowed value was never dropped early.

</details>

## Quiz

1. State the three ownership rules.

<details>
<summary>Answer</summary>

Every value has an owner; there can be only one owner at a time; when the owner goes out of scope, the value is dropped.

</details>

2. When exactly is a local variable dropped?

<details>
<summary>Answer</summary>

At the closing brace of the block where it was declared (the end of its scope), unless its value was moved elsewhere before that.

</details>

3. In which order are local variables dropped?

<details>
<summary>Answer</summary>

In reverse order of declaration: the last variable created is the first one dropped.

</details>

4. What happens to the old variable when you pass a `String` to a function by value?

<details>
<summary>Answer</summary>

Ownership moves into the function's parameter. The old variable no longer owns anything, and using it is a compile error (`E0382`).

</details>

5. Why can Rust never free the same heap memory twice in safe code?

<details>
<summary>Answer</summary>

Because a value has only one owner at a time, and only the owner drops the value. After a move, the old variable cannot drop it.

</details>

6. What does `drop(x)` do, and why is `x` unusable afterwards?

<details>
<summary>Answer</summary>

`drop` is an ordinary function that takes ownership of `x`. Its parameter goes out of scope at once, so the value is cleaned up. Since the value was moved into the function, `x` can no longer be used.

</details>

## Summary

- Ownership has three rules: one owner per value, one owner at a time, drop when the owner leaves its scope.
- A scope is a block between braces. Variables are dropped at the end of their scope, in reverse order.
- Passing a value to a function by value **moves** ownership; returning a value moves it back out.
- Using a variable after its value was moved is the compile error `E0382`.
- Assigning a new value to a variable drops the old one at once; dropping an owner drops everything it owns.
- Giving values away and taking them back works but is clumsy. Borrowing (next lessons) is the clean solution.

## Sources and further reading

- [The Rust Programming Language — 4.1 What Is Ownership?](https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html) — the official chapter: the rules, scope, `String`, move, and ownership with functions.
- [Rust By Example — Ownership and moves](https://doc.rust-lang.org/rust-by-example/scope/move.html) — short examples of moving values.
- [Rust By Example — RAII](https://doc.rust-lang.org/rust-by-example/scope/raii.html) — the idea of automatic cleanup with `Drop`.
- [The Rust Reference — Destructors](https://doc.rust-lang.org/reference/destructors.html) — the exact rules of drop order (an advanced read for later).
- [Error code E0382](https://doc.rust-lang.org/error_codes/E0382.html) — the official explanation of "use of moved value".

---

[🏠 Start](../START-HERE.md) · [← 03.1 Memory: the stack and the heap](01-memory-stack-and-heap.md) · [📋 Phase 03](README.md) · [03.3 Move, copy and clone →](03-move-copy-and-clone.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
