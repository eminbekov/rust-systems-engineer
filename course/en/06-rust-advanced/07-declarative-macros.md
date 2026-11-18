<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 06.6 Unsafe Rust and Miri](06-unsafe-rust-and-miri.md) · [📋 Phase 06](README.md) · [06.8 Procedural macros →](08-procedural-macros.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/06-rust-advanced/07-declarative-macros.md) <!-- nav -->

# Lesson 06.7 — Declarative macros

> **Phase 06 — Rust advanced** · Lesson 7 of 12 · ⏱ about 3.5 hours (2 sittings)

## What you will learn

- What a macro is, and how it is different from a function.
- How to write your own macro with `macro_rules!`: patterns, fragments such as `$x:expr`, and repetitions such as `$( ... ),*`.
- How macros call themselves (recursion), and what "hygiene" means.
- How to generate whole items (`struct`, `impl`) with a macro, and how to share a macro between crates.
- How to look at the code a macro produces with `cargo expand`, and how to read macro error messages.

## Before you start

- You finished [Lesson 06.6 — Unsafe Rust and Miri](06-unsafe-rust-and-miri.md).
- You know traits and generics: [Lesson 05.2 — Traits](../05-rust-intermediate/02-traits.md).
- You know how to write tests: [Lesson 05.6 — Testing in depth](../05-rust-intermediate/06-testing-in-depth.md).
- Create a project for this lesson: `cd ~/projects/rust && cargo new code/p06-macros` (run it inside your learning repository from Phase 01).

## New words

| Word | Meaning |
|---|---|
| **macro** | Code that writes code. The compiler runs the macro while it compiles and replaces the macro call with the code the macro produced. You already use macros: `println!`, `vec!`, `assert_eq!`. |
| **declarative macro** | A macro written with `macro_rules!`. You describe patterns of input and the code to produce for each pattern. |
| **procedural macro** | A macro written as a Rust function that works on code as data. You write one in [Lesson 06.8](08-procedural-macros.md). |
| **macro expansion** | The step where the compiler replaces a macro call with the code the macro produces. |
| **token** | The smallest piece of source code the compiler understands: a name, a number, a symbol such as `+` or `,`. |
| **token tree** (`tt`) | One token, or a group of tokens inside matching brackets `()`, `[]` or `{}`. |
| **fragment specifier** | The part after the colon in `$x:expr`. It says what kind of code the macro variable must match (an expression, a name, a type, …). |
| **repetition** | The syntax `$( ... ),*` that matches or produces a piece of code many times. |
| **hygiene** | The rule that names created inside a macro cannot clash with names in the code that calls the macro. |
| **AST** (Abstract Syntax Tree) | The tree the compiler builds from your source code, for example "a sum of two numbers". |
| **DSL** (Domain-Specific Language) | A tiny language made for one job, for example `calc!(add 2, 3)` in this lesson. |
| **DRY** (Don't Repeat Yourself) | The principle that the same knowledge should be written in one place only. |

## 1. What a macro is

A **function** runs when your program runs. It receives values and returns a value.

A **macro** runs earlier, while the compiler compiles your program. It receives *pieces of source code* and returns *new source code*. The compiler then compiles that new code as if you had typed it yourself.

You have used macros since your first program. The `!` after a name tells you it is a macro call, not a function call:

```rust
fn main() {
    let numbers = vec![1, 2, 3];
    println!("{:?}", numbers);
}
```

`vec![1, 2, 3]` is a macro. A function cannot do what it does, because a function in Rust has a fixed number of parameters. A macro can accept any number of items, and `println!` can check its format string before your program starts.

Macros are useful for three jobs:

1. **A variable number of arguments** (`vec!`, `println!`).
2. **Removing repeated code** that functions and generics cannot remove, for example writing the same `impl` for ten types (DRY).
3. **Small custom syntax** (a mini language, a DSL).

Rust has two families of macros. This lesson teaches the first one.

| Family | How you write it | Lesson |
|---|---|---|
| Declarative | `macro_rules! name { ... }` with patterns | this lesson |
| Procedural | a Rust function that receives and returns tokens | [Lesson 06.8](08-procedural-macros.md) |

> 📏 **Convention:** Reach for a function first. If you need different types, use generics and traits. Use a macro only when a function cannot do the job. Macros are harder to read, harder to debug and give worse error messages.

## 2. Your first macro

A declarative macro looks like a `match`. Each *arm* has a pattern on the left and the code to produce on the right.

```rust
macro_rules! say_hello {
    () => {
        println!("Hello from a macro!");
    };
}

fn main() {
    say_hello!();
    say_hello!();
}
```

**What happens:**

1. `macro_rules! say_hello { ... }` defines a macro named `say_hello`. You call it with `say_hello!`.
2. The macro has one arm. The pattern `()` on the left means "the call has nothing between the brackets".
3. The code in `{ ... }` on the right is what the compiler pastes in place of every call.
4. The compiler replaces both `say_hello!();` calls with `println!("Hello from a macro!");` and then compiles the result.

```text
Hello from a macro!
Hello from a macro!
```

> 💡 **Tip:** You may call a macro with `()`, `[]` or `{}`. `say_hello!()`, `say_hello![]` and `say_hello!{}` mean the same. The habit is: `vec![...]` with square brackets, everything else with round brackets.

## 3. Patterns and fragments

To accept input, you write **macro variables**. A macro variable starts with `$` and has a *fragment specifier* after a colon: `$x:expr` means "match any expression and call it `$x`".

```rust
macro_rules! square {
    ($x:expr) => {
        $x * $x
    };
}

fn main() {
    println!("{}", square!(7));
    println!("{}", square!(1 + 2));
}
```

**What happens:**

1. The pattern `($x:expr)` matches one expression. For `square!(7)`, `$x` is `7`. The macro produces `7 * 7`.
2. For `square!(1 + 2)`, the compiler parses `1 + 2` as **one** expression first. So the result is `(1 + 2) * (1 + 2)`, not `1 + 2 * 1 + 2`. A macro in Rust works on the *tree* of the code, not on plain text, so you do not need extra brackets.

```text
49
9
```

These are the fragment specifiers you will use most:

| Specifier | Matches | Example input |
|---|---|---|
| `expr` | an expression | `1 + 2`, `foo(x)`, `"text"` |
| `ident` | a name (identifier) | `my_var`, `Point` |
| `ty` | a type | `i32`, `Vec<String>` |
| `literal` | a literal value | `5`, `"hi"`, `true` |
| `pat` | a pattern | `Some(x)`, `1..=5` |
| `stmt` | a statement | `let a = 1` |
| `block` | a block in braces | `{ a + b }` |
| `item` | an item: function, struct, `impl`… | `fn f() {}` |
| `path` | a path | `std::collections::HashMap` |
| `vis` | a visibility, maybe empty | `pub`, `pub(crate)` |
| `lifetime` | a lifetime | `'a` |
| `tt` | a single token tree: the "anything" fragment | `+`, `foo`, `( a, b )` |

> ⚠️ **Common mistake:** After `$x:expr` the compiler allows only `=>`, `,` or `;`. This rule keeps the macro unambiguous. So you write `$a:expr, $b:expr` with a comma between them. Writing `$a:expr $b:expr` is a compile error.

### Several arms and fixed words

A macro can have many arms. The compiler tries them from top to bottom and uses the first one that matches. Arms can contain fixed words, so you can build a tiny language:

```rust
macro_rules! calc {
    (add $a:expr, $b:expr) => {
        $a + $b
    };
    (mul $a:expr, $b:expr) => {
        $a * $b
    };
    (neg $a:expr) => {
        -$a
    };
}

fn main() {
    println!("{}", calc!(add 2, 3));
    println!("{}", calc!(mul 4, 5));
    println!("{}", calc!(neg 6));
}
```

**What happens:**

1. `calc!(add 2, 3)` does not match the `mul` arm or the `neg` arm. It matches the first arm: the fixed word `add`, then an expression `2`, a comma and an expression `3`.
2. The macro produces `2 + 3`. The other two calls work the same way.

```text
5
20
-6
```

## 4. Repetitions

How does `vec![1, 2, 3]` accept any number of items? With a **repetition**: `$( ... ) separator quantifier`.

- The part inside `$( ... )` is repeated.
- The *separator* is optional: usually `,` or `;`.
- The *quantifier* says how many times: `*` means zero or more, `+` means one or more, `?` means zero or one.

Let us write our own `vec!`:

```rust
macro_rules! my_vec {
    () => {
        Vec::new()
    };
    ($($item:expr),+ $(,)?) => {{
        let mut v = Vec::new();
        $( v.push($item); )+
        v
    }};
}

fn main() {
    let empty: Vec<i32> = my_vec![];
    let numbers = my_vec![1, 2, 3];
    let words = my_vec!["a", "b", "c",];
    println!("{:?}", empty);
    println!("{:?}", numbers);
    println!("{:?}", words);
}
```

**What happens:**

1. The first arm `()` handles the empty call and produces `Vec::new()`.
2. In the second arm, `$($item:expr),+` means "one or more expressions, separated by commas". Every match is stored in `$item`.
3. `$(,)?` means "an optional comma at the end", so `my_vec!["a", "b", "c",]` is allowed.
4. On the right side, `$( v.push($item); )+` repeats the code in the brackets once for each matched item.
5. The body is wrapped in **double** braces `{{ ... }}`. The outer pair is the macro syntax. The inner pair makes a block, so the macro can contain several statements (`let`, `push`, and the final `v`) and still be one expression.

```text
[]
[1, 2, 3]
["a", "b", "c"]
```

### Repetition with two variables

Repetition works with several variables at once. Here is a `hashmap!` macro:

```rust
use std::collections::HashMap;

macro_rules! hashmap {
    ($($key:expr => $value:expr),* $(,)?) => {{
        let mut map = HashMap::new();
        $( map.insert($key, $value); )*
        map
    }};
}

fn main() {
    let ages = hashmap! {
        "Aziza" => 31,
        "Bobur" => 27,
    };
    let mut names: Vec<_> = ages.iter().collect();
    names.sort();
    println!("{:?}", names);
}
```

**What happens:**

1. Each repetition matches `key => value`. The `=>` is a fixed token that you chose; it is only decoration.
2. For every pair the compiler writes one `map.insert(key, value);`. Variables that repeat together (`$key` and `$value`) must repeat the same number of times.
3. We collect the pairs into a `Vec` and sort it, because a `HashMap` has no fixed order and we want stable output.

```text
[("Aziza", 31), ("Bobur", 27)]
```

## 5. Macros that call themselves

A macro may call itself. This is how you handle "the first item, then the rest". Here is a `max!` macro for any number of values:

```rust
macro_rules! max {
    ($x:expr) => {
        $x
    };
    ($x:expr, $($rest:expr),+) => {{
        let a = $x;
        let b = max!($($rest),+);
        if a > b { a } else { b }
    }};
}

fn main() {
    println!("{}", max!(4));
    println!("{}", max!(3, 9, 5));
    println!("{}", max!(1.5, 0.5));
}
```

**What happens:**

1. `max!(4)` matches the first arm (one expression, nothing after it) and produces `4`.
2. `max!(3, 9, 5)` matches the second arm: `$x` is `3`, and `$rest` holds `9` and `5`.
3. The macro produces code that calls `max!(9, 5)`. The compiler expands that again, until one value is left.
4. The compiler expands everything before type checking, so the same macro works for integers and for floating-point numbers.

```text
4
9
1.5
```

The arm order matters. The shorter pattern is first, because the second arm needs a comma and at least one more value.

> ⚡ **Performance:** A macro has no cost at run time: the produced code is ordinary code. The cost is in compile time and in the size of the code. Deep recursion also has a limit (the default `recursion_limit` is 128 levels).

To see the code the compiler really builds, use `cargo expand` (section 8). The expansion of `max!(3, 9, 5)` is:

```rust
{
    let a = 3;
    let b = {
        let a = 9;
        let b = 5;
        if a > b { a } else { b }
    };
    if a > b { a } else { b }
}
```

Notice that there are two variables named `a`. Why does this work? That is the next section.

## 6. Hygiene

Macros in many languages break when a name inside the macro clashes with a name outside. Rust macros are **hygienic** for local variables: a variable that the macro creates is invisible to the caller, and the caller's variables are invisible to the macro's own code.

```rust
macro_rules! double_it {
    ($e:expr) => {{
        let x = 2;
        $e * x
    }};
}

fn main() {
    let x = 10;
    println!("{}", double_it!(x + 1));
}
```

**What happens:**

1. The caller has `x = 10`. The macro also creates its own `x = 2`.
2. The expression `x + 1` that you wrote belongs to the caller, so its `x` is the caller's `x`, which is `10`.
3. The macro's `x` is a different variable. The result is `(10 + 1) * 2 = 22`.

```text
22
```

Hygiene works in the other direction too. A `let` inside a macro does not leak out:

```rust
macro_rules! make_var {
    () => {
        let secret = 5;
    };
}

fn main() {
    make_var!();
    println!("{}", secret);
}
```

```text
error[E0425]: cannot find value `secret` in this scope
 --> src/main.rs:9:20
  |
9 |     println!("{}", secret);
  |                    ^^^^^^ not found in this scope
  |
help: an identifier with the same name is defined here, but is not accessible due to macro hygiene
```

The compiler even tells you the reason: "macro hygiene". The fix is to let the caller choose the name and pass it in as an `ident`: `make_var!(secret)` with the pattern `($name:ident) => { let $name = 5; }`.

Hygiene covers local variables and loop labels. **Items** (functions, structs, traits) that a macro defines *are* visible to the caller. That is how the next section works.

## 7. Generating items

A macro can produce whole items. Here is the classic use: the same `impl` for many types.

```rust
trait Describe {
    fn describe(&self) -> String;
}

macro_rules! impl_describe {
    ($($t:ty),+ $(,)?) => {
        $(
            impl Describe for $t {
                fn describe(&self) -> String {
                    format!("{} of type {}", self, stringify!($t))
                }
            }
        )+
    };
}

impl_describe!(i32, u8, f64, bool);

fn main() {
    println!("{}", 42_i32.describe());
    println!("{}", 7_u8.describe());
    println!("{}", 2.5_f64.describe());
    println!("{}", true.describe());
}
```

**What happens:**

1. `$t:ty` matches a type. The macro repeats the whole `impl` block once per type: four `impl` blocks from one line (DRY).
2. `stringify!($t)` is a built-in macro that turns code into a string literal. `stringify!(i32)` is `"i32"`.
3. `format!("{} ...", self, ...)` needs `self` to implement `Display`. All four types do, so the macro compiles for all of them.

```text
42 of type i32
7 of type u8
2.5 of type f64
true of type bool
```

You can also generate a struct, by taking its name as an `ident`:

```rust
macro_rules! make_struct {
    ($name:ident { $($field:ident : $ty:ty),* $(,)? }) => {
        #[derive(Debug)]
        struct $name {
            $($field: $ty),*
        }

        impl $name {
            fn field_names() -> &'static [&'static str] {
                &[$(stringify!($field)),*]
            }
        }
    };
}

make_struct!(Point { x: i32, y: i32 });

fn main() {
    let p = Point { x: 3, y: 4 };
    println!("{:?}", p);
    println!("sum = {}", p.x + p.y);
    println!("{:?}", Point::field_names());
}
```

**What happens:**

1. `$name:ident` matches `Point`. Each repetition `$field:ident : $ty:ty` matches `x: i32` and `y: i32`.
2. The macro writes `struct Point { x: i32, y: i32 }` and an `impl` block with a function that lists the field names as strings.
3. Because items are not hygienic, `main` can use `Point` even though the macro created it.

```text
Point { x: 3, y: 4 }
sum = 7
["x", "y"]
```

## 8. Looking inside: `cargo expand`

When a macro does not behave, look at the code it produces. `cargo expand` prints your crate after all macros are expanded.

Set it up once (the `nightly` toolchain is used only to print the expansion; your project still builds with stable Rust):

```sh
rustup toolchain install nightly --profile minimal
cargo install cargo-expand
```

Then, inside your project:

```sh
cargo expand
```

For the `my_vec!` example from section 4, the relevant part of the output is:

```text
fn main() {
    let empty: Vec<i32> = Vec::new();
    let numbers = {
        let mut v = Vec::new();
        v.push(1);
        v.push(2);
        v.push(3);
        v
    };
    ...
}
```

You see exactly what the compiler sees. Even `println!` is a macro: in the full output it becomes a call to `::std::io::_print(format_args!(...))`.

## 9. Using a macro in other places

`macro_rules!` has two scope rules that surprise beginners.

**Rule 1 — define before use.** A macro must appear in the file *above* the code that calls it:

```rust
fn main() {
    greet!();
}

macro_rules! greet {
    () => {
        println!("hi");
    };
}
```

```text
error: cannot find macro `greet` in this scope
 --> src/main.rs:2:5
  |
2 |     greet!();
  |     ^^^^^ consider moving the definition of `greet` before this call
```

The fix is to move `macro_rules! greet` above `main`.

**Rule 2 — export with `#[macro_export]`.** A normal macro is private to its file/module. To use it from other crates, mark it `#[macro_export]`. It is then placed at the root of the crate. Inside it, write `$crate::` before the crate's own items, so they are found wherever the macro is used:

```rust
/// Turns text into upper case and adds an exclamation mark.
pub fn yell(text: &str) -> String {
    format!("{}!", text.to_uppercase())
}

/// Calls `yell` for you. `$crate` always means "this crate",
/// even when the macro is used from another crate.
#[macro_export]
macro_rules! shout {
    ($text:expr) => {
        $crate::yell($text)
    };
}
```

If this code is the library `macros07`, another crate (or a binary in the same package) writes:

```rust
use macros07::shout;

fn main() {
    println!("{}", shout!("hello"));
}
```

```text
HELLO!
```

> 🔒 **Security:** A macro can hide dangerous code, including `unsafe` blocks. Before you use a macro from a crate you do not know, read what it expands to (`cargo expand`). We come back to this in the security phase.

## 10. Error messages and `compile_error!`

Macro errors say "no rules expected". They happen when a call matches none of the arms:

```rust
macro_rules! square {
    ($x:expr) => {
        $x * $x
    };
}

fn main() {
    println!("{}", square!(1, 2));
}
```

```text
error: no rules expected `,`
 --> src/main.rs:8:29
  |
1 | macro_rules! square {
  | ------------------- when calling this macro
...
8 |     println!("{}", square!(1, 2));
  |                             ^ no rules expected this token in macro call
  |
note: while trying to match meta-variable `$x:expr`
```

**What it means:** the macro matched `1` as `$x`, then found a comma, but the pattern ended after `$x`. A call with too few arguments gives a similar message: `unexpected end of macro invocation`.

You can give your users a better message with a last "catch-all" arm that calls the built-in `compile_error!`:

```rust
macro_rules! only_number {
    ($x:literal) => {
        $x
    };
    ($($t:tt)*) => {
        compile_error!("only_number! takes exactly one literal, like only_number!(5)")
    };
}

fn main() {
    println!("{}", only_number!(5));
    println!("{}", only_number!(1 + 2));
}
```

```text
error: only_number! takes exactly one literal, like only_number!(5)
  --> src/main.rs:6:9
   |
 6 |         compile_error!("only_number! takes exactly one literal, like only_number!(5)")
   |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
...
12 |     println!("{}", only_number!(1 + 2));
   |                    ------------------- in this macro invocation
```

**What happens:** `5` matches the first arm. `1 + 2` is not a single literal, so the compiler tries the next arm. `$($t:tt)*` matches *anything*, so that arm always runs and stops the build with your message. Keep this "catch-all" arm last.

> 🧪 **Testing:** Test a macro like any other code: call it in a `#[test]` and use `assert_eq!`. Test the edge cases of the *patterns*: the empty call, one item, many items, a trailing comma. For "this call must fail to compile" you can write a documentation test marked `compile_fail` (documentation tests are explained in [Lesson 05.7 — Documentation](../05-rust-intermediate/07-documentation.md)).

## Common mistakes

**1. Calling a macro without `!`.** `say_hello()` looks for a function. Always write the `!`: `say_hello!()`.

**2. Missing comma between fragments.** `($a:expr $b:expr)` is rejected when the macro is defined:

```text
error: `$a:expr` is followed by `$b:expr`, which is not allowed for `expr` fragments
  = note: allowed there are: `=>`, `,` or `;`
```

Put a fixed token between them: `($a:expr, $b:expr)`. The same rule is why a call like `my_vec![1 2 3]` (no commas) fails with the error "no rules expected" and the token `2`.

**3. Forgetting the double braces.** If your macro body has several statements and you wrote `=> { let a = 1; a + $x }`, the macro tries to produce *statements*, which does not work where an expression is needed:

```text
error: expected expression, found `let` statement
error: macro expansion ignores `a` and any tokens following
  = note: the usage of `one_plus!` is likely invalid in expression context
```

Use `=> {{ let a = 1; a + $x }}`: the inner braces make the whole body one block, and a block is an expression.

**4. Putting the general arm first.** The compiler uses the first arm that matches. If `($($t:tt)*)` is first, no other arm is ever used. Order arms from the most specific to the most general.

**5. Using a macro before it is defined** (section 9). Move the definition up.

**6. Expecting a macro's `let` to be visible outside** (section 6). Pass the name in as an `ident`.

**7. Using a macro when a function would do.** If you can write a plain function or a generic function, do that. Compare: `fn square<T: std::ops::Mul<Output = T> + Copy>(x: T) -> T { x * x }` does the job of `square!` with better error messages.

## Try it yourself

1. Open your project: `cd ~/projects/rust/code/p06-macros`.
2. Replace `src/main.rs` with the `my_vec!` example from section 4. Run it:

   ```sh
   cargo run
   ```

   You should see `[]`, `[1, 2, 3]` and `["a", "b", "c"]` on three lines.
3. Install the tool from section 8 (if not yet installed) and run `cargo expand`. Find the code that replaced `my_vec![1, 2, 3]`.
4. Break the macro on purpose: call `my_vec![1 2 3]` (no commas). Read the error. Which token does the compiler say it did not expect? (Answer: `2`, the second number, because after `1` the pattern wants a comma.)
5. Save your work with Git, as you did in Phase 01:

   ```sh
   git add code/p06-macros
   git commit -m "feat(p06-macros): add my_vec macro"
   ```

## Exercises

### Exercise 1 — `min!` (easy)

Write a macro `min!` that returns the smallest of one or more values. `min!(8, 3, 5)` is `3`. `min!(2.5, 7.0)` is `2.5`. Use the `max!` macro from section 5 as an example.

<details>
<summary>Hint</summary>

Copy `max!`. You need only one change: the comparison in the `if`.

</details>

<details>
<summary>Answer</summary>

```rust
macro_rules! min {
    ($x:expr) => {
        $x
    };
    ($x:expr, $($rest:expr),+) => {{
        let a = $x;
        let b = min!($($rest),+);
        if a < b { a } else { b }
    }};
}

fn main() {
    println!("{}", min!(8));
    println!("{}", min!(8, 3, 5));
    println!("{}", min!(2.5, 7.0));
}
```

```text
8
3
2.5
```

Why it works: the first arm stops the recursion when one value is left. The second arm takes the first value and asks `min!` for the smallest of the rest.

</details>

### Exercise 2 — `set!` with tests (medium)

Write a macro `set!` that builds a `HashSet` from a list of expressions, like `hashmap!` does for maps. It must allow a trailing comma. Write two tests: one that checks duplicates are removed, and one that checks a trailing comma works.

<details>
<summary>Hint</summary>

Start from `hashmap!` in section 4. A set has no values, so use `s.insert($item)`. Put the tests in `#[cfg(test)] mod tests` and write `use super::*;` at the top of the module so the tests can see `HashSet`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::HashSet;

macro_rules! set {
    ($($item:expr),* $(,)?) => {{
        let mut s = HashSet::new();
        $( s.insert($item); )*
        s
    }};
}

fn main() {
    let s = set![3, 1, 3, 2, 1];
    let mut sorted: Vec<_> = s.into_iter().collect();
    sorted.sort();
    println!("{:?}", sorted);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn set_removes_duplicates() {
        let s = set![1, 1, 2];
        assert_eq!(s.len(), 2);
        assert!(s.contains(&1));
        assert!(s.contains(&2));
    }

    #[test]
    fn trailing_comma_is_allowed() {
        let one = set![5,];
        assert_eq!(one.len(), 1);
    }
}
```

`cargo run` prints `[1, 2, 3]`. `cargo test` prints `test result: ok. 2 passed`.

Why it works: `$(,)?` accepts one optional comma at the end. A `HashSet` ignores an `insert` of a value it already has, so duplicates disappear.

</details>

### Exercise 3 — `string_enum!` (medium)

Write a macro `string_enum!` so that `string_enum!(Color { Red, Green, Blue });` creates an `enum Color` with a method `as_str(&self) -> &'static str` (returning `"Red"`, …) and a function `all() -> &'static [Color]`. Print every color and its text.

<details>
<summary>Hint</summary>

The enum name is an `ident`. The variants are `$($variant:ident),+`. Inside `match self`, write `$name::$variant => stringify!($variant)` once per variant. A reference to an array of unit variants such as `&[Color::Red, Color::Green]` can be `'static`, because the compiler stores such constant data in the program itself.

</details>

<details>
<summary>Answer</summary>

```rust
macro_rules! string_enum {
    ($name:ident { $($variant:ident),+ $(,)? }) => {
        #[derive(Debug, Clone, Copy, PartialEq)]
        enum $name {
            $($variant),+
        }

        impl $name {
            fn as_str(&self) -> &'static str {
                match self {
                    $($name::$variant => stringify!($variant)),+
                }
            }

            fn all() -> &'static [$name] {
                &[$($name::$variant),+]
            }
        }
    };
}

string_enum!(Color { Red, Green, Blue });

fn main() {
    for color in Color::all() {
        println!("{:?} -> {}", color, color.as_str());
    }
}
```

```text
Red -> Red
Green -> Green
Blue -> Blue
```

Why it works: the macro repeats `$variant` in three places: the enum body, the `match` arms and the slice. One list of names now gives all three, which is the DRY idea.

</details>

### Exercise 4 — `ensure!` with tests (hard)

Write `ensure!(condition, "message {}", value)`. If the condition is false, the macro must `return Err(String)` from the *calling function*, with a message formatted like `format!`. Use it in `fn check_age(age: i32) -> Result<i32, String>` that rejects negative ages and ages of 150 or more. Write a test for the success case and for one failure message.

<details>
<summary>Hint</summary>

After the condition you need "everything else, any number of tokens", to pass to `format!`. The fragment for that is `$($arg:tt)+`. Then write `format!($($arg)+)`. The `return` inside the macro returns from the function where the macro is called.

</details>

<details>
<summary>Answer</summary>

```rust
macro_rules! ensure {
    ($cond:expr, $($arg:tt)+) => {
        if !$cond {
            return Err(format!($($arg)+));
        }
    };
}

fn check_age(age: i32) -> Result<i32, String> {
    ensure!(age >= 0, "age {} is negative", age);
    ensure!(age < 150, "age {} is too large", age);
    Ok(age)
}

fn main() {
    println!("{:?}", check_age(30));
    println!("{:?}", check_age(-4));
    println!("{:?}", check_age(200));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_age_passes() {
        assert_eq!(check_age(30), Ok(30));
    }

    #[test]
    fn negative_age_fails_with_message() {
        assert_eq!(check_age(-4), Err("age -4 is negative".to_string()));
    }
}
```

```text
Ok(30)
Err("age -4 is negative")
Err("age 200 is too large")
```

Why it works: a macro is pasted into the calling function, so `return` returns from `check_age`. A function could not do this. This is one of the real reasons to choose a macro.

</details>

### Exercise 5 — count the tokens (hard)

Write a macro `count!` that returns how many token trees it receives: `count!()` is `0`, `count!(a b c)` is `3`, `count!(+ - * / %)` is `5`. Use recursion and `tt`.

<details>
<summary>Hint</summary>

Two arms. The first one matches nothing and gives `0usize`. The second one matches `$head:tt $($tail:tt)*` and gives `1usize + count!($($tail)*)`.

</details>

<details>
<summary>Answer</summary>

```rust
macro_rules! count {
    () => { 0usize };
    ($head:tt $($tail:tt)*) => { 1usize + count!($($tail)*) };
}

fn main() {
    println!("{}", count!());
    println!("{}", count!(a b c));
    println!("{}", count!(+ - * / %));
}
```

```text
0
3
5
```

Why it works: each step removes one token tree (`$head`), adds `1`, and counts the tail. The empty call ends the recursion. For a very long input this hits the recursion limit, so real code uses other tricks, but the idea is the same.

</details>

## Quiz

1. What is the difference between a function and a macro?

<details>
<summary>Answer</summary>

A function runs when the program runs and receives values. A macro runs while the compiler compiles, receives pieces of source code and returns new source code that replaces the macro call.

</details>

2. What does `$x:expr` mean? What does the `expr` part tell the compiler?

<details>
<summary>Answer</summary>

`$x` is a macro variable. `expr` is the fragment specifier: it says that `$x` must match a whole expression. The compiler parses the expression first, so `1 + 2` stays one unit.

</details>

3. What do `*`, `+` and `?` mean after `$( ... )`?

<details>
<summary>Answer</summary>

`*` means zero or more repetitions, `+` means one or more, and `?` means zero or one.

</details>

4. Why does `double_it!(x + 1)` use the caller's `x`, although the macro also declares `let x = 2`?

<details>
<summary>Answer</summary>

Because of hygiene. Local variables created inside a macro live in their own "name space". The `x` in `x + 1` was written by the caller, so it means the caller's `x`. The macro's `x` is a different variable.

</details>

5. Where must a `macro_rules!` definition be placed, and what makes it usable from another crate?

<details>
<summary>Answer</summary>

It must be written above the code that uses it. To use it from another crate, put `#[macro_export]` on it. Inside, refer to the crate's own items through `$crate::`.

</details>

6. A call fails with "no rules expected `,`". What does it mean?

<details>
<summary>Answer</summary>

The call did not match any arm of the macro: the compiler found a comma at a place where no arm allows one. Check the call against the patterns, or add an arm for that shape.

</details>

7. Name one job that a macro can do and a function cannot.

<details>
<summary>Answer</summary>

Examples: accept a variable number of arguments (`vec!`), generate several `impl` blocks or a whole `struct`, or `return` from the calling function (`ensure!`).

</details>

## Summary

- A macro is code that writes code. It runs at compile time, and the compiler compiles what it produces.
- `macro_rules!` works like `match`: arms of `pattern => code`, tried from top to bottom.
- Macro variables have fragment specifiers: `expr`, `ident`, `ty`, `literal`, `tt` and others.
- `$( ... ),*` and `$( ... ),+` repeat code. `$(,)?` allows a trailing comma.
- Macros can call themselves, which handles "the first item, then the rest".
- Hygiene keeps local names of the macro and of the caller apart. Items that a macro defines are visible to the caller.
- `#[macro_export]` and `$crate` share a macro between crates. `cargo expand` shows what a macro produced.
- Prefer functions and generics. Use a macro when you need variable arguments, generated items, or control flow like `return`.

## Sources and further reading

- [The Rust Programming Language — 20.5 Macros](https://doc.rust-lang.org/book/ch20-05-macros.html) — the official book chapter on declarative and procedural macros.
- [The Rust Reference — Macros By Example](https://doc.rust-lang.org/reference/macros-by-example.html) — the exact rules for fragments, repetitions and scoping.
- [The Little Book of Rust Macros](https://veykril.github.io/tlborm/) — a long, practical guide with advanced patterns.
- [Rust by Example — macro_rules!](https://doc.rust-lang.org/rust-by-example/macros.html) — short examples of designators, repetition and DSLs.
- [cargo-expand](https://github.com/dtolnay/cargo-expand) — install instructions and options for the expansion tool.

---

[🏠 Start](../START-HERE.md) · [← 06.6 Unsafe Rust and Miri](06-unsafe-rust-and-miri.md) · [📋 Phase 06](README.md) · [06.8 Procedural macros →](08-procedural-macros.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
