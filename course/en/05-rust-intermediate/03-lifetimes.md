<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 05.2 Traits](02-traits.md) · [📋 Phase 05](README.md) · [05.4 Closures →](04-closures.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/05-rust-intermediate/03-lifetimes.md) <!-- nav -->

# Lesson 05.3 — Lifetimes

> **Phase 05 — Rust intermediate** · Lesson 3 of 14 · ⏱ about 4 hours (2 sittings)

## What you will learn

- What a lifetime is, and which problem the compiler solves with it (dangling references).
- How to read and write lifetime annotations such as `'a` in functions and structs.
- The three *elision rules* that let you leave most annotations out.
- What `'static` means, and how to fix the typical lifetime errors.
- How lifetimes let you write fast "zero-copy" code that returns pieces of its input.

## Before you start

- You finished [Lesson 05.2 — Traits](02-traits.md). Lifetimes are a kind of generic parameter, so
  [Lesson 05.1 — Generics](01-generics.md) matters here too.
- You understand references and the borrowing rules ([Lesson 03.4](../03-ownership-and-borrowing/04-references-and-borrowing.md)
  and [Lesson 03.5](../03-ownership-and-borrowing/05-mutable-references-and-the-borrow-rules.md))
  and string slices ([Lesson 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md)).
- You have the package `code/p05-lessons` from Lesson 05.1.

## New words

| Word | Meaning |
|---|---|
| **lifetime** | The part of the program during which a reference is valid, from where it is created to its last use. The value it points to must live at least that long. |
| **lifetime annotation** | A name like `'a` (an apostrophe and a short name) written on references to describe how their lifetimes relate. |
| **lifetime parameter** | The declaration of such a name in angle brackets: the `'a` in `fn longest<'a>(…)`. |
| **dangling reference** | A reference to a value that no longer exists. Rust makes it impossible. |
| **borrow checker** | The part of the compiler that checks all the borrowing and lifetime rules (Lesson 03.4). |
| **lifetime elision** | The compiler's rules for filling in obvious lifetimes, so you do not have to write them. |
| **`'static`** | The longest lifetime: the whole run of the program. String literals have it. |
| **`'_`** | The "anonymous lifetime": a placeholder that says "some lifetime, fill it in by the elision rules". |
| **zero-copy** | Reading data by pointing into the original instead of copying it into new strings. |
| **NLL** (Non-Lexical Lifetimes) | The compiler's way of ending a borrow at the reference's *last use*, not at the end of the block. |

## 1. The problem: a reference to something that is gone

In Lesson 03.4 you learned that a reference borrows a value and must not outlive it. Here is the
mistake the compiler is protecting you from:

```rust
fn main() {
    let r;
    {
        let x = 5;
        r = &x;
    }
    println!("r: {r}");
}
```

```text
error[E0597]: `x` does not live long enough
 --> examples/dangling.rs:5:13
  |
4 |         let x = 5;
  |             - binding `x` declared here
5 |         r = &x;
  |             ^^ borrowed value does not live long enough
6 |     }
  |     - `x` dropped here while still borrowed
7 |     println!("r: {r}");
  |                   - borrow later used here
```

**What happens:**

1. `r` is declared in the outer block. `x` lives in the inner block `{ … }`.
2. At the closing `}` of the inner block `x` is dropped (Lesson 03.2: a value is dropped when its
   owner goes out of scope).
3. `r` still points to the memory where `x` was. Using it afterwards would read garbage. The
   compiler stops this at build time.

Here is the same story as a picture. The **lifetime** of a reference is the stretch of code during
which it is used; the value must live at least as long:

```text
fn main() {
    let r;                  // ──────────────┐  r is used until here
    {                       //               │
        let x = 5;          // ─────┐        │
        r = &x;             //      │ x      │  <- r borrows x,
    }                       // ─────┘ ends   │     but x ends first!
    println!("r: {r}");     // ──────────────┘
}
```

So, in general, **a lifetime is the answer to the question "how long is this reference allowed to
be used?"**. Most of the time the compiler works it out alone. This lesson is about the cases where
it needs your help.

## 2. Lifetime annotations in functions

Write a function that returns the longer of two string slices:

```rust
fn longest(x: &str, y: &str) -> &str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}
```

```text
error[E0106]: missing lifetime specifier
 --> examples/longest_err.rs:1:33
  |
1 | fn longest(x: &str, y: &str) -> &str {
  |               ----     ----     ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but the signature does not say whether it is borrowed from `x` or `y`
help: consider introducing a named lifetime parameter
  |
1 | fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
  |           ++++     ++          ++          ++
```

**What it means:** the function returns a reference. A reference must point into something owned by
somebody else. The signature does not say whether the result points into `x` or into `y`, and the
compiler needs to know, because the answer decides how long the result may be used. Here the answer
depends on the run-time lengths, so the compiler cannot guess. You must describe it:

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let first = String::from("lifetimes");
    let second = String::from("rust");
    let result = longest(first.as_str(), second.as_str());
    println!("the longest is {result}");
}
```

```text
the longest is lifetimes
```

**What happens:**

1. `<'a>` declares a **lifetime parameter** named `'a`. It works like the `<T>` in Lesson 05.1, but
   for lifetimes instead of types. By convention lifetime names are short, lowercase and start with
   an apostrophe.
2. `x: &'a str` and `y: &'a str` mean: "both references are valid for at least the lifetime `'a`".
3. `-> &'a str` means: "the result is valid for that same `'a`".
4. Put together: **the result lives no longer than the shorter-living of `x` and `y`**. `'a` becomes
   the overlap of the two input lifetimes.

> 💡 **Tip:** Read `<'a>` as "there is some span of time, call it `a`". The annotations connect
> references to each other. They do not give anything more time.

### Annotations describe; they do not change anything

A lifetime annotation never makes a value live longer. It only tells the compiler how the lifetimes
of different references relate, so it can check that you use the result safely. To see the check
work, use `longest` with two strings whose lives end at different times:

```rust
fn main() {
    let first = String::from("lifetimes");
    let result;
    {
        let second = String::from("rust");
        result = longest(first.as_str(), second.as_str());
    }
    println!("the longest is {result}");
}
```

```text
error[E0597]: `second` does not live long enough
  --> examples/longest_scope_err.rs:14:42
   |
13 |         let second = String::from("rust");
   |             ------ binding `second` declared here
14 |         result = longest(first.as_str(), second.as_str());
   |                                          ^^^^^^ borrowed value does not live long enough
15 |     }
   |     - `second` dropped here while still borrowed
16 |     println!("the longest is {result}");
   |                               ------ borrow later used here
```

**What it means:** `result` might point to `second` (we know it does not, but the signature only
promises "the shorter of the two"). `second` is dropped at `}`, and `result` is used after that.
The fix is to use `result` inside the block where `second` is still alive:

```rust
fn main() {
    let first = String::from("lifetimes");
    {
        let second = String::from("rust");
        let result = longest(first.as_str(), second.as_str());
        println!("the longest is {result}");
    }
}
```

```text
the longest is lifetimes
```

### Only connect what is really connected

Not every parameter needs the same lifetime. If the result can only come from `x`, say so, and the
other argument is free:

```rust
fn first_of<'a>(x: &'a str, _y: &str) -> &'a str {
    x
}

fn main() {
    let a = String::from("kept");
    let result;
    {
        let b = String::from("temporary");
        result = first_of(&a, &b);
    }
    println!("{result}");
}
```

```text
kept
```

**What happens:** the result is tied only to `x`'s lifetime `'a`. The temporary `b` may die before
`result` is used. The name `_y` starts with an underscore to tell the compiler "unused on purpose".
This shows the idea to remember: **the signature is a promise about what the function does**, and
callers rely on the signature only (never on the body).

### You cannot return a reference to a local value

```rust
fn shout<'a>(word: &'a str) -> &'a str {
    let loud = word.to_uppercase();
    loud.as_str()
}
```

```text
error[E0515]: cannot return value referencing local variable `loud`
 --> examples/return_local_err.rs:3:5
  |
3 |     loud.as_str()
  |     ----^^^^^^^^^
  |     |
  |     returns a value referencing data owned by the current function
  |     `loud` is borrowed here
```

**What it means:** `loud` is created inside the function and dropped when the function ends. A
reference to it would dangle. No lifetime annotation can fix this. The fix is to **return the owned
value**, so the caller becomes its owner:

```rust
fn shout(word: &str) -> String {
    word.to_uppercase()
}

fn main() {
    println!("{}", shout("hi"));
}
```

```text
HI
```

A similar mistake is `fn make_greeting() -> &String { … }` with no parameters at all. The compiler
says `error[E0106]: missing lifetime specifier … this function's return type contains a borrowed
value, but there is no value for it to be borrowed from`, and suggests the right fix: "you are more
likely to want to return an owned value".

### A signature that lies

If the body returns something that the signature does not allow, the compiler catches it:

```rust
fn pick_first<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
    if x.is_empty() {
        y
    } else {
        x
    }
}
```

```text
error: lifetime may not live long enough
 --> examples/mismatch_err.rs:3:9
  |
1 | fn pick_first<'a, 'b>(x: &'a str, y: &'b str) -> &'a str {
  |               --  -- lifetime `'b` defined here
  |               |
  |               lifetime `'a` defined here
2 |     if x.is_empty() {
3 |         y
  |         ^ function was supposed to return data with lifetime `'a` but it is returning data with lifetime `'b`
```

**What it means:** the signature promises a result tied to `x`, but the body may return `y`. The fix
is to use the same lifetime for both (as in `longest`) or to return only `x`.

## 3. Structs that hold references

A struct can store a reference, but then it needs a lifetime parameter: the struct must not outlive
the data it points to.

```rust
#[derive(Debug)]
struct Excerpt<'a> {
    part: &'a str,
}

impl<'a> Excerpt<'a> {
    fn level(&self) -> usize {
        self.part.len()
    }

    fn announce(&self, note: &str) -> &str {
        println!("note: {note}");
        self.part
    }
}

fn main() {
    let novel = String::from("Call me Ishmael. Some years ago...");
    let first_sentence = novel.split('.').next().unwrap();
    let e = Excerpt { part: first_sentence };
    println!("{:?}", e);
    println!("level = {}", e.level());
    println!("{}", e.announce("reading"));
}
```

**What happens:**

1. `struct Excerpt<'a>` declares the lifetime parameter, and `part: &'a str` uses it. It says: "an
   `Excerpt` cannot live longer than the string it points into".
2. `impl<'a> Excerpt<'a>` declares `'a` again for the `impl` block (the same pattern as `impl<T>
   Point<T>` in Lesson 05.1) and uses it in `Excerpt<'a>`.
3. `novel.split('.')` cuts the text at each dot and `.next().unwrap()` takes the first piece
   (Lesson 05.5 explains `.next()`). The piece is a `&str` pointing **into** `novel`; no new string
   was allocated.
4. `e` holds that reference, so `e` must not outlive `novel`. The compiler checks this.
5. `announce` takes `&self` and `note: &str` and returns `&str`. We did not annotate it; the next
   section explains why that works.

```text
Excerpt { part: "Call me Ishmael" }
level = 15
note: reading
Call me Ishmael
```

Without the parameter, the compiler tells you what to do:

```rust
struct Excerpt {
    part: &str,
}
```

```text
error[E0106]: missing lifetime specifier
 --> examples/excerpt_err.rs:2:11
  |
2 |     part: &str,
  |           ^ expected named lifetime parameter
  |
help: consider introducing a named lifetime parameter
  |
1 ~ struct Excerpt<'a> {
2 ~     part: &'a str,
  |
```

> 📏 **Convention:** Beginners often reach for references inside structs. If it makes the code hard,
> store an owned `String` instead. A struct with a reference is right when the struct is a short-lived
> "view" (a parser result, a token, a line of a log), as in Section 6.

## 4. Lifetime elision: why you rarely write `'a`

You have written many functions with references and never needed `'a`. For example:

```rust
fn first_word(s: &str) -> &str {
    match s.find(' ') {
        Some(i) => &s[..i],
        None => s,
    }
}

// the same function with the lifetimes written out
fn first_word_explicit<'a>(s: &'a str) -> &'a str {
    match s.find(' ') {
        Some(i) => &s[..i],
        None => s,
    }
}

fn main() {
    println!("{}", first_word("hello brave world"));
    println!("{}", first_word_explicit("single"));
}
```

```text
hello
single
```

Both functions are identical to the compiler. Early Rust required the long form every time, but
the same patterns appeared again and again, so the compiler learned to fill them in. These are the
**lifetime elision rules**. The compiler applies them in order, and if after all three the result's
lifetime is still unknown, it reports `E0106` and asks you to annotate.

| # | Rule | Example (what you write → what the compiler sees) |
|---|---|---|
| 1 | Every reference parameter gets its **own** lifetime. | `fn f(a: &str, b: &str)` → `fn f<'a, 'b>(a: &'a str, b: &'b str)` |
| 2 | If there is **exactly one** input lifetime, it is given to all output references. | `fn f(a: &str) -> &str` → `fn f<'a>(a: &'a str) -> &'a str` |
| 3 | If one input is `&self` or `&mut self`, **its** lifetime is given to all output references. | `fn announce(&self, n: &str) -> &str` → the result gets the lifetime of `self` |

Apply the rules to the examples:

- `first_word(s: &str) -> &str`: rule 1 gives `s` a lifetime, rule 2 gives it to the result. ✓
- `longest(x: &str, y: &str) -> &str`: rule 1 gives two different lifetimes; rule 2 does not apply
  (two inputs); rule 3 does not apply (no `self`). The result is unknown. ✗ so you must annotate.
- `Excerpt::announce(&self, note: &str) -> &str`: rule 3 applies, so the result is tied to `self`. ✓

The anonymous lifetime `'_` says "a lifetime, deduced by the rules". It is useful in types where
the `<'a>` would be noise:

```rust
fn parse_line(line: &str) -> Option<LogLine<'_>>
```

reads "returns a `LogLine` that borrows from the input", without naming the lifetime.

## 5. The special lifetime `'static`

`'static` is the lifetime of the entire program run. **String literals** have it, because the text
is stored inside the program file itself and exists as long as the program does:

```rust
fn motto() -> &'static str {
    "make it work, make it right, make it fast"
}

fn main() {
    let s: &'static str = "string literals live for the whole program";
    println!("{s}");
    println!("{}", motto());
}
```

```text
string literals live for the whole program
make it work, make it right, make it fast
```

> ⚠️ **Common mistake:** When the compiler complains about lifetimes, do not "fix" it by writing
> `'static` everywhere. That promises the data lives for the whole program, which is almost never
> true for data you computed; you will only get another error. First ask: *who owns this data, and
> for how long?* Usually the right fix is to return an owned value (`String`) or to keep the owner
> alive longer.

You will meet `'static` again in Lesson 05.10: a thread started with `thread::spawn` may outlive the
function that created it, so the data it uses must be owned (or `'static`).

## 6. A real use: zero-copy parsing

Systems programs often read big inputs (logs, network packets, files). Copying every field into a
new `String` costs memory and time. With lifetimes, a parser can return **slices pointing into the
original text**:

```rust
#[derive(Debug)]
struct LogLine<'a> {
    level: &'a str,
    module: &'a str,
    message: &'a str,
}

fn parse_line(line: &str) -> Option<LogLine<'_>> {
    let (level, rest) = line.split_once(' ')?;
    let (module, message) = rest.split_once(": ")?;
    Some(LogLine { level, module, message })
}

fn main() {
    let text = String::from("ERROR net: connection refused");
    match parse_line(&text) {
        Some(line) => {
            println!("{:?}", line);
            println!("level={} module={} message={}", line.level, line.module, line.message);
        }
        None => println!("not a log line"),
    }
    println!("{:?}", parse_line("garbage").is_none());
}
```

**What happens:**

1. `split_once(' ')` splits a `&str` at the first space and returns `Option<(&str, &str)>`: both
   parts are slices of the original (Lesson 03.6). `?` returns `None` if there is no space.
2. We split again on `": "` to separate the module from the message.
3. `LogLine { level, module, message }` uses the *field init shorthand* (Lesson 04.1): the variable
   names equal the field names.
4. The result borrows from `line` (`LogLine<'_>`). Nothing was copied; the three fields are just
   pointers and lengths into `text`.
5. `parse_line("garbage")` has no space, so it returns `None`.

```text
LogLine { level: "ERROR", module: "net", message: "connection refused" }
level=ERROR module=net message=connection refused
true
```

The price is that the result is tied to its source. If the source goes away, the compiler refuses:

```rust
fn main() {
    let text = String::from("WARN disk: almost full");
    let line = parse_line(&text).unwrap();
    drop(text);
    println!("{:?}", line);
}
```

```text
error[E0505]: cannot move out of `text` because it is borrowed
  --> examples/logline_err.rs:15:10
   |
13 |     let text = String::from("WARN disk: almost full");
   |         ---- binding `text` declared here
14 |     let line = parse_line(&text).unwrap();
   |                           ----- borrow of `text` occurs here
15 |     drop(text);
   |          ^^^^ move out of `text` occurs here
16 |     println!("{:?}", line);
   |                      ---- borrow later used here
```

**What it means:** `drop(text)` would destroy the text that `line` points into. In languages without
this check, this is a classic *use-after-free* bug and a security hole. In Rust, it is a compile error.

> ⚡ **Performance:** Zero-copy parsing is the reason fast Rust tools (log processors, parsers,
> network proxies) can handle huge inputs with almost no memory allocation. Whenever you can return
> `&str` slices instead of `String`s, you save one allocation per field. But *measure first* (Lesson
> 05.12); clear code with `String` is fine when speed does not matter.

### Borrows end at the last use (NLL)

The compiler does not keep a borrow alive until the end of the block, only until the reference's
**last use**. This is called NLL (Non-Lexical Lifetimes):

```rust
fn main() {
    let mut text = String::from("hi");
    let r = &text;
    println!("{r}");
    text.push('!');
    println!("{text}");
}
```

```text
hi
hi!
```

**What happens:** `r` borrows `text` immutably. `r` is last used in the first `println!`, so the
borrow is over. `text.push('!')` needs a mutable borrow, which is now allowed. In very old versions
of Rust this program did not compile.

## 7. Lifetimes, generics and trait bounds together

Lifetime parameters and type parameters live in the same angle brackets, lifetimes first:

```rust
use std::fmt::Display;

fn longest_with_note<'a, T>(x: &'a str, y: &'a str, note: T) -> &'a str
where
    T: Display,
{
    println!("note: {note}");
    if x.len() >= y.len() {
        x
    } else {
        y
    }
}

fn main() {
    println!("{}", longest_with_note("alpha", "be", "comparing"));
}
```

```text
note: comparing
alpha
```

**What happens:** `<'a, T>` declares a lifetime and a type. `note` has its own type `T` (here `&str`)
and no connection with `'a`, because only `x` and `y` can end up in the result. Lesson 06.5
returns to lifetimes with harder cases (bounds like `T: 'a`, lifetimes in traits).

## Common mistakes

**1. Missing lifetime on a function result** (`E0106`, Section 2). Add `<'a>` and connect the inputs
that the result can borrow from.

**2. Missing lifetime in a struct field** (`E0106`, Section 3). Add `<'a>` to the struct and the `impl`.

**3. Using the result after the data is gone** (`E0597`, Sections 1 and 2). Move the use inside the
scope of the owner, or make the owner live longer (declare it earlier).

**4. Returning a reference to a local variable** (`E0515`, Section 2). Return the owned value
(`String`, `Vec`, …).

**5. A signature promising more than the body can keep** (`lifetime may not live long enough`).
Make the lifetimes agree with what the body returns.

**6. Dropping or modifying the source while a view is alive** (`E0505`, Section 6). Keep the source
alive until the last use of the view.

## Try it yourself

This time you will practise two Git tools that are useful when code does not compile: a throw-away
**experiment branch**, and **stash**.

1. Start the lesson branch and save the working `longest` program:

   ```bash
   cd ~/projects/rust
   git switch main
   git switch -c lesson/05-03-lifetimes
   ```

   Create `code/p05-lessons/examples/lifetimes_longest.rs` with the working `longest` program of
   Section 2, run `cargo run --example lifetimes_longest`, then commit:

   ```bash
   git add .
   git commit -m "feat(p05): longest with a lifetime parameter"
   ```

2. Now break things **on purpose** without risking your work. Create an experiment branch and make
   the scope error of Section 2 (`result` used outside the inner block). Run it and read the error.

   ```bash
   git switch -c experiment/break-lifetimes
   # edit the example, run it, read the error message
   git commit -am "wip: break lifetimes on purpose"
   ```

   When you have learned what you wanted, throw the whole experiment away:

   ```bash
   git switch lesson/05-03-lifetimes
   git branch -D experiment/break-lifetimes
   ```

   ```text
   Deleted branch experiment/break-lifetimes (was 24855d9).
   ```

   `-D` (capital) deletes a branch even if it was never merged. Use it only for experiments you want
   to forget. Your lesson branch has not been touched.

3. An experiment that was **not committed** can be discarded with `git restore`:

   ```bash
   echo "x" >> code/p05-lessons/src/main.rs
   git status --short
   git restore code/p05-lessons/src/main.rs
   git status --short
   ```

   The first `git status --short` shows ` M code/p05-lessons/src/main.rs` (modified); after
   `git restore` the second one shows nothing. ⚠️ `restore` cannot be undone: the changes are gone.

4. Imagine you are in the middle of a half-written change and must switch branches. Git refuses to
   switch if it would overwrite your edits. **`git stash`** puts your uncommitted changes on a shelf
   and gives you a clean working folder:

   ```bash
   echo "// half-finished idea" >> code/p05-lessons/src/main.rs
   git stash push -m "half-finished idea"
   git status --short
   git stash list
   git stash pop
   ```

   ```text
   Saved working directory and index state On lesson/05-03-lifetimes: half-finished idea
   stash@{0}: On lesson/05-03-lifetimes: half-finished idea
   ```

   `git status --short` printed nothing (clean). `git stash pop` brings the changes back and removes
   the shelf entry; its output ends with `Dropped refs/stash@{0} (…)`. Undo the pretend edit with
   `git restore code/p05-lessons/src/main.rs`.

5. Finish the lesson as before: commit the `Excerpt` and `LogLine` examples, merge into `main`,
   delete the lesson branch.

## Exercises

### Exercise 1 — Fix the signature (easy)

This function does not compile. Add lifetime annotations so that it returns the shorter of the two
strings (the first one if they have the same length). Add two tests.

```rust
fn shorter(a: &str, b: &str) -> &str {
    if a.len() <= b.len() {
        a
    } else {
        b
    }
}
```

<details>
<summary>Hint</summary>

The result may come from `a` or from `b`, so both parameters and the result need the same lifetime
name: `<'a>`, `a: &'a str`, `b: &'a str`, `-> &'a str`.

</details>

<details>
<summary>Answer</summary>

```rust
fn shorter<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() <= b.len() {
        a
    } else {
        b
    }
}

fn main() {
    println!("{}", shorter("tree", "forest"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn picks_the_shorter_text() {
        assert_eq!(shorter("tree", "forest"), "tree");
        assert_eq!(shorter("abc", "x"), "x");
    }

    #[test]
    fn equal_length_returns_the_first() {
        assert_eq!(shorter("ab", "cd"), "ab");
    }
}
```

```text
tree
```

Why it works: the same `'a` on both inputs and on the output means "the result is valid as long
as both inputs are". The compiler reported `E0106` before, because rule 2 of elision needs exactly
one input reference.

</details>

### Exercise 2 — A settings line (medium)

Write `struct Setting<'a> { key: &'a str, value: &'a str }` and
`fn parse_setting(line: &str) -> Option<Setting<'_>>` that splits `"port = 8080"` at `=` and removes
the spaces around both parts. A line without `=` gives `None`. Derive `Debug` and `PartialEq` and
write two tests.

<details>
<summary>Hint</summary>

`line.split_once('=')?` gives the two halves. `trim()` returns a `&str` without leading and
trailing whitespace; it still points into the original text, so no copy is made.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, PartialEq)]
struct Setting<'a> {
    key: &'a str,
    value: &'a str,
}

fn parse_setting(line: &str) -> Option<Setting<'_>> {
    let (key, value) = line.split_once('=')?;
    Some(Setting { key: key.trim(), value: value.trim() })
}

fn main() {
    println!("{:?}", parse_setting("port = 8080"));
    println!("{:?}", parse_setting("no equals sign"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_key_and_value() {
        assert_eq!(
            parse_setting("port = 8080"),
            Some(Setting { key: "port", value: "8080" })
        );
    }

    #[test]
    fn line_without_equals_is_none() {
        assert_eq!(parse_setting("just text"), None);
    }
}
```

```text
Some(Setting { key: "port", value: "8080" })
None
```

Why it works: elision rule 2 ties the `Setting` returned to the only input reference, `line`. All
four slices (`key`, `value` and their trimmed versions) point into `line`.

</details>

### Exercise 3 — Fix the scope error in two ways (medium)

The program below fails with `E0597` (this is the program from Section 2). Fix it **(a)** by changing
only `main`, and **(b)** by changing the function so it returns an owned `String` (keep the inner
block in `main` as it is).

```rust
fn main() {
    let first = String::from("lifetimes");
    let result;
    {
        let second = String::from("rust");
        result = longest(first.as_str(), second.as_str());
    }
    println!("the longest is {result}");
}
```

<details>
<summary>Hint</summary>

(a) The owner `second` must live as long as `result` is used: move the `println!` or the
declaration. (b) If the function returns a `String`, the result does not borrow from anything.
`x.to_string()` makes an owned copy of a `&str`.

</details>

<details>
<summary>Answer</summary>

(a) Declare both strings in the same block, so they live equally long (this is the program from
Section 2 that compiles):

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() {
        x
    } else {
        y
    }
}

fn main() {
    let first = String::from("lifetimes");
    let second = String::from("rust");
    let result = longest(first.as_str(), second.as_str());
    println!("the longest is {result}");
}
```

(b) Return an owned value:

```rust
fn longest_owned(x: &str, y: &str) -> String {
    if x.len() > y.len() {
        x.to_string()
    } else {
        y.to_string()
    }
}

fn main() {
    let first = String::from("lifetimes");
    let result;
    {
        let second = String::from("rust");
        result = longest_owned(&first, &second);
    }
    println!("the longest is {result}");
}
```

```text
the longest is lifetimes
```

Why it works: in (a) the owners outlive the use. In (b) the result no longer points to `second`, so
`second` may be dropped. The price is one allocation for the copy. This is the trade-off from
Section 6: references are faster, owned values are easier.

</details>

### Exercise 4 — Parse a config text without copying (hard)

Write `fn parse_config(text: &str) -> Vec<(&str, &str)>`. It reads the text line by line, skips
empty lines and lines starting with `#`, and for every line with `=` pushes the trimmed `(key, value)`
pair. Lines without `=` are ignored. Write tests for: comments, blank and broken lines; empty text;
a value that itself contains `=` (`url = a=b` gives `("url", "a=b")`).

<details>
<summary>Hint</summary>

`text.lines()` can be used in a `for` loop (Lesson 05.5 explains it). Shadow the loop variable with
`let line = line.trim();`. Use `line.is_empty()`, `line.starts_with('#')` and `continue`.
`split_once('=')` splits only at the **first** `=`, which handles the last test. The return type needs
no annotation: elision rule 2 applies.

</details>

<details>
<summary>Answer</summary>

```rust
fn parse_config(text: &str) -> Vec<(&str, &str)> {
    let mut pairs = Vec::new();
    for line in text.lines() {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }
        if let Some((key, value)) = line.split_once('=') {
            pairs.push((key.trim(), value.trim()));
        }
    }
    pairs
}

fn main() {
    let config = String::from("# server\nhost = example.org\n\nport=8080\nbroken line\n");
    for (key, value) in parse_config(&config) {
        println!("{key} -> {value}");
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn skips_comments_blank_and_broken_lines() {
        let text = "# c\nhost = a\n\nbroken\nport=1";
        assert_eq!(parse_config(text), vec![("host", "a"), ("port", "1")]);
    }

    #[test]
    fn empty_text_gives_no_pairs() {
        assert!(parse_config("").is_empty());
    }

    #[test]
    fn value_may_contain_equals() {
        assert_eq!(parse_config("url = a=b"), vec![("url", "a=b")]);
    }
}
```

```text
host -> example.org
port -> 8080
```

Why it works: every `&str` in the returned vector points into `text`, and the vector cannot outlive
`text`. No `String` was allocated for the keys and values, only the vector itself.

</details>

## Quiz

1. What problem do lifetimes solve?

<details>
<summary>Answer</summary>

They let the compiler prove that no reference outlives the value it points to, so dangling references and use-after-free bugs are impossible in safe Rust.

</details>

2. What does `fn f<'a>(x: &'a str, y: &'a str) -> &'a str` promise?

<details>
<summary>Answer</summary>

That the returned reference is valid for the lifetime `'a`, which is the overlap of both inputs, that is, as long as both `x` and `y` are valid. The result may point into either argument.

</details>

3. Does writing `'a` make a value live longer?

<details>
<summary>Answer</summary>

No. Annotations only describe the relationship between existing lifetimes, so the compiler can check your code. A value lives as long as its owner's scope says.

</details>

4. Why does `fn longest(x: &str, y: &str) -> &str` need annotations, but `fn first_word(s: &str) -> &str` does not?

<details>
<summary>Answer</summary>

Elision rule 2: with exactly one input reference, its lifetime is given to the output. With two input references (and no `self`) the compiler cannot know which one the result borrows from.

</details>

5. A function builds a `String` inside and wants to return a `&str` to it. What is the fix?

<details>
<summary>Answer</summary>

Return the `String` itself (an owned value). A reference to a local variable would dangle when the function ends (`E0515`).

</details>

6. When is it a good idea to put a reference (and so a lifetime) inside a struct?

<details>
<summary>Answer</summary>

When the struct is a short-lived view of data owned elsewhere, for example the result of parsing a line without copying. If the struct needs to live independently or for long, store owned data (`String`) instead.

</details>

## Summary

- A lifetime is how long a reference may be used. The compiler checks that it never exceeds the life of the value it points to.
- Lifetime annotations (`'a`) describe how the lifetimes of inputs and outputs relate; they never extend a life.
- Functions that return references must say which inputs the result borrows from, unless the three elision rules fill it in.
- A struct that holds a reference needs a lifetime parameter, repeated in its `impl` block.
- `'static` means "for the whole program"; string literals have it. Do not use it to silence errors.
- You cannot return a reference to a local variable: return the owned value instead.
- Zero-copy parsing returns slices into the input; the result is tied to the input's life.
- Git habit: break things on a throw-away experiment branch, discard with `git branch -D` or `git restore`, shelve half-done work with `git stash`.

## Sources and further reading

- [The Rust Programming Language — 10.3 Validating References with Lifetimes](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html) — the official chapter, with the same `longest` example family.
- [The Rust Reference — Lifetime elision](https://doc.rust-lang.org/reference/lifetime-elision.html) — the exact elision rules.
- [Rust by Example — Lifetimes](https://doc.rust-lang.org/rust-by-example/scope/lifetime.html) — short examples.
- [The Rustonomicon — Lifetimes](https://doc.rust-lang.org/nomicon/lifetimes.html) — a deeper look for later (Lesson 06.5).
- [Git documentation — git stash](https://git-scm.com/docs/git-stash) — shelving uncommitted changes.

---

[🏠 Start](../START-HERE.md) · [← 05.2 Traits](02-traits.md) · [📋 Phase 05](README.md) · [05.4 Closures →](04-closures.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
