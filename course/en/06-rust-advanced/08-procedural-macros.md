<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 06.7 Declarative macros](07-declarative-macros.md) · [📋 Phase 06](README.md) · [06.9 Introduction to async/await →](09-async-await-introduction.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/06-rust-advanced/08-procedural-macros.md) <!-- nav -->

# Lesson 06.8 — Procedural macros

> **Phase 06 — Rust advanced** · Lesson 8 of 12 · ⏱ about 3 hours (2 sittings)

## What you will learn

- What a procedural macro is, the three kinds (function-like, derive, attribute), and why they live in their own crate.
- How to read code with `syn` and write code with `quote`.
- How to write all three kinds, step by step, in a small Cargo workspace.
- How to report errors that point at the exact place in the user's code.
- How to debug and test a procedural macro (`cargo expand`, unit tests, `trybuild`).

## Before you start

- You finished [Lesson 06.7 — Declarative macros](07-declarative-macros.md). You must know what a macro and a token are.
- You know traits and generics ([Lesson 05.2](../05-rust-intermediate/02-traits.md), [Lesson 05.1](../05-rust-intermediate/01-generics.md)) and how to add crates ([Lesson 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md)).
- Install `cargo-expand` as shown in Lesson 06.7 (section 8).

## New words

| Word | Meaning |
|---|---|
| **procedural macro** | A macro written as a normal Rust function. It runs inside the compiler, receives code as a list of tokens and returns new code as tokens. |
| **function-like macro** | A procedural macro called like `name!(...)`. |
| **derive macro** | A procedural macro used as `#[derive(Name)]` above a struct or enum. It adds new code (usually an `impl`) next to the type. |
| **attribute macro** | A procedural macro used as `#[name]` above an item such as a function. It receives the item and replaces it with new code. |
| **TokenStream** | The type for "a list of tokens". A macro receives one and returns one. |
| **token** | The smallest piece of source code: a name, a number, a symbol like `+`. |
| **AST** (Abstract Syntax Tree) | The tree that describes the meaning of code, for example "a struct named User with two fields". |
| **API** (Application Programming Interface) | What a library offers to its users: its functions, types and messages. |
| **syn** | A crate that parses a `TokenStream` into an AST you can inspect with normal Rust code. |
| **quote** | A crate that builds a `TokenStream` from code written inside `quote! { ... }`, with `#variable` holes. |
| **proc-macro2** | A crate with its own copy of the token types. It works in ordinary programs and tests, not only inside the compiler. |
| **span** | The place in the source code where a token came from. Errors use spans to point at the right line. |
| **workspace** | A group of Cargo packages that live in one folder tree and share one `Cargo.lock` and one `target` folder. |
| **crate type** | What a crate builds into: a binary, a library, or (here) a procedural macro library. |
| **compile time** | The time when `cargo build` runs the compiler, before your program runs. |

## 1. What a procedural macro is

In Lesson 06.7 you wrote macros as *patterns*. That is enough for many jobs, but a pattern cannot look inside a `struct`, count its fields, read their names or run real logic. A **procedural macro** can, because it is a real Rust function:

```text
 your source code            the compiler                 your macro (a Rust function)
┌─────────────────┐   tokens   ┌─────────────┐  TokenStream  ┌──────────────────────┐
│ #[derive(Foo)]  │ ─────────► │  finds the  │ ────────────► │ fn derive_foo(input) │
│ struct User {…} │            │  macro call │               │   -> TokenStream     │
└─────────────────┘            └─────────────┘               └──────────┬───────────┘
                                      ▲                                 │ new tokens
                                      └─────────────────────────────────┘
                     the compiler compiles the original code AND the new code
```

A procedural macro:

1. receives code as a `TokenStream`;
2. does whatever Rust code can do (loops, `if`, strings, calculations);
3. returns a new `TokenStream`, which the compiler compiles in place of (or next to) the original.

There are three kinds:

| Kind | How the user writes it | What it receives | What it does |
|---|---|---|---|
| Function-like | `my_macro!(...)` | the tokens between the brackets | replaces the call with new code |
| Derive | `#[derive(MyTrait)]` on a struct/enum | the struct/enum | **adds** new code (usually an `impl`); the original stays |
| Attribute | `#[my_attr]` or `#[my_attr(args)]` on an item | the arguments and the item | **replaces** the item with new code |

You have used derive macros already: `#[derive(Debug)]` is one (built into the compiler), and `serde`'s `#[derive(Serialize)]` ([Lesson 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md)) is a derive macro written by someone else with exactly the tools of this lesson. In section 7 you will see attribute macros too.

### One rule: its own crate

A crate that contains procedural macros must be a special kind of library with `proc-macro = true` in its `Cargo.toml`. It is compiled and *run by the compiler*, so it must be built before the code that uses it. That is why you always need two packages: one with the macros, and one that uses them. A Cargo **workspace** keeps both in one place.

## 2. Setting up the workspace

Create the workspace folder and two packages:

```sh
mkdir -p ~/projects/rust/code/p06-procmacros && cd ~/projects/rust/code/p06-procmacros
printf '[workspace]\nresolver = "3"\nmembers = []\n' > Cargo.toml
cargo new --lib toolbox-macros
cargo new app
```

Cargo adds `toolbox-macros` and `app` to `members` for you. Now edit `toolbox-macros/Cargo.toml`:

```toml
[package]
name = "toolbox-macros"
version = "0.1.0"
edition = "2024"

[lib]
proc-macro = true

[dependencies]
syn = { version = "3.0", features = ["full"] }
quote = "1.0"
proc-macro2 = "1.0"
```

and `app/Cargo.toml`:

```toml
[package]
name = "app"
version = "0.1.0"
edition = "2024"

[dependencies]
toolbox-macros = { path = "../toolbox-macros" }
```

**What happens:**

1. `[lib] proc-macro = true` tells Cargo to build this crate as a procedural macro library.
2. `syn` parses tokens. The feature `"full"` lets it parse every kind of Rust item, including functions (we need that for the attribute macro). Without `"full"`, `syn` parses only "derive input" (structs, enums, unions).
3. `quote` and `proc-macro2` are the helpers described in "New words". Both are used by almost every procedural macro.
4. In `app`, `path = "../toolbox-macros"` makes Cargo use the local package, not crates.io.

Run `cargo build` in the workspace folder to download and compile the dependencies. (`syn` 3.x is current at the time of writing. Many tutorials were written for `syn` 2; the code in this lesson is checked with 3.)

## 3. A function-like macro

Our first macro takes a string literal and turns it into upper case **at compile time**: `shout_const!("welcome")` becomes `"WELCOME"` before the program runs.

`toolbox-macros/src/lib.rs`:

```rust
use proc_macro::TokenStream;
use quote::quote;
use syn::{parse_macro_input, LitStr};

/// `shout_const!("hello")` becomes the string literal `"HELLO"`,
/// computed while the compiler runs.
#[proc_macro]
pub fn shout_const(input: TokenStream) -> TokenStream {
    let text = parse_macro_input!(input as LitStr);
    let upper = text.value().to_uppercase();
    quote! { #upper }.into()
}
```

`app/src/main.rs`:

```rust
use toolbox_macros::shout_const;

const BANNER: &str = shout_const!("welcome");

fn main() {
    println!("{}", BANNER);
    println!("{}", shout_const!("compile time"));
}
```

**What happens:**

1. `#[proc_macro]` marks `shout_const` as a function-like macro. The function name is the macro name. The signature is always `fn(TokenStream) -> TokenStream`.
2. `parse_macro_input!(input as LitStr)` asks `syn` to parse the input tokens as a string literal (`LitStr`). If the input is not a string literal, the macro reports a compile error and returns early.
3. `text.value()` gives the real `String` inside the literal. `to_uppercase()` is ordinary Rust.
4. `quote! { #upper }` builds tokens. `#upper` is a *hole*: `quote` puts the value of the variable `upper` there. A `String` becomes a string literal in the output tokens.
5. `.into()` converts from `proc_macro2::TokenStream` (what `quote!` makes) to `proc_macro::TokenStream` (what the compiler expects).
6. In `main.rs`, the compiler replaces `shout_const!("welcome")` with `"WELCOME"` and compiles the program.

```text
WELCOME
COMPILE TIME
```

If the user passes something that is not a string literal, `syn` produces an error that points at it:

```rust
use toolbox_macros::shout_const;

fn main() {
    println!("{}", shout_const!(42));
}
```

```text
error: expected string literal
 --> src/main.rs:4:33
  |
4 |     println!("{}", shout_const!(42));
  |                                 ^^
```

The arrow points at `42`, the exact input that was wrong. That is the **span** at work. You got this good error message for free from `parse_macro_input!`.

> 💡 **Tip:** `proc_macro` is a crate that is built into the compiler. You do not add it to `Cargo.toml`. `proc_macro2`, `syn` and `quote` are normal crates from crates.io.

## 4. A derive macro

Now a derive macro. We want this to work:

```rust
#[derive(FieldNames)]
struct User {
    name: String,
    age: u32,
}

// User::type_name()   -> "User"
// User::field_names() -> ["name", "age"]
```

Add to `toolbox-macros/src/lib.rs` (the `use` line at the top now lists more names):

```rust
use proc_macro::TokenStream;
use proc_macro2::TokenStream as TokenStream2;
use quote::quote;
use syn::{parse_macro_input, Data, DeriveInput, Fields, LitStr};

/// Derive macro: adds `type_name()` and `field_names()` to a struct.
#[proc_macro_derive(FieldNames)]
pub fn derive_field_names(input: TokenStream) -> TokenStream {
    expand_field_names(input.into()).into()
}

fn expand_field_names(input: TokenStream2) -> TokenStream2 {
    let ast: DeriveInput = match syn::parse2(input) {
        Ok(ast) => ast,
        Err(error) => return error.to_compile_error(),
    };
    let name = &ast.ident;

    let fields = match &ast.data {
        Data::Struct(data) => match &data.fields {
            Fields::Named(named) => &named.named,
            _ => {
                return syn::Error::new_spanned(name, "FieldNames needs a struct with named fields")
                    .to_compile_error();
            }
        },
        _ => {
            return syn::Error::new_spanned(name, "FieldNames can only be used on structs")
                .to_compile_error();
        }
    };

    let field_names = fields.iter().map(|f| f.ident.as_ref().unwrap().to_string());
    let (impl_generics, ty_generics, where_clause) = ast.generics.split_for_impl();

    quote! {
        impl #impl_generics #name #ty_generics #where_clause {
            pub fn type_name() -> &'static str {
                stringify!(#name)
            }

            pub fn field_names() -> &'static [&'static str] {
                &[#(#field_names),*]
            }
        }
    }
}
```

**What happens:**

1. `#[proc_macro_derive(FieldNames)]` makes the function the implementation of `#[derive(FieldNames)]`. The name in the brackets is what users write.
2. The derive function only converts types (`.into()`) and calls `expand_field_names`. All real work is in a normal function that uses `proc_macro2` types. This split makes testing possible (section 8).
3. `DeriveInput` is the AST of "a struct, enum or union". Its main parts are `ident` (the type name), `generics` and `data` (the fields or variants).
4. We `match` on `ast.data`. If it is a struct with *named* fields, we get the list of fields. For an enum or a tuple struct we return an error.
5. `syn::Error::new_spanned(name, "message")` creates an error that points at the tokens of `name`. `.to_compile_error()` turns it into tokens that make the compiler print this error. We **return** these tokens instead of panicking.
6. `fields.iter().map(...)` makes a list of the field names as `String`s.
7. `split_for_impl()` splits the generics into the three pieces an `impl` block needs: `impl<T>`, `Type<T>` and the `where` clause. This makes the macro work for `struct Pair<T> { ... }` too.
8. In `quote!`, `#name` inserts the type name. `#(#field_names),*` is a *repetition*, like in Lesson 06.7: it writes every item of `field_names`, separated by commas.

Use it in `app/src/main.rs`:

```rust
use toolbox_macros::FieldNames;

#[derive(FieldNames)]
struct User {
    name: String,
    age: u32,
}

#[derive(FieldNames)]
struct Pair<T> {
    left: T,
    right: T,
}

fn main() {
    println!("{} has {:?}", User::type_name(), User::field_names());
    println!("{} has {:?}", Pair::<u8>::type_name(), Pair::<u8>::field_names());

    let user = User { name: "Aziza".to_string(), age: 31 };
    println!("{} is {}", user.name, user.age);
    let pair = Pair { left: 1, right: 2 };
    println!("{}", pair.left + pair.right);
}
```

```text
User has ["name", "age"]
Pair has ["left", "right"]
Aziza is 31
3
```

### What the macro sees

What does a `DeriveInput` really look like? You can print it with `eprintln!("{:#?}", ast);` inside the macro (it needs the `"extra-traits"` feature of `syn` for `Debug`). Here is the start of the real output for `User`, shortened:

```text
DeriveInput {
    attrs: [],
    vis: Visibility::Inherited,
    ident: Ident {
        ident: "User",
        span: #0 bytes(134..138),
    },
    generics: Generics { lt_token: None, params: [], gt_token: None, where_clause: None },
    data: Data::Struct {
        struct_token: Token![struct],
        fields: Fields::Named {
            brace_token: Brace,
            named: [
                Field {
                    ident: Some(Ident { ident: "name", ... }),
                    ty: Type::Path { ... "String" ... },
                    ...
                },
                Token![,],
                Field { ident: Some(Ident { ident: "age", ... }), ... },
```

Reading this tree is the main skill of writing procedural macros: you find the part you need (`data → fields → named → ident`) and write a `match` that walks to it.

### Errors that point at the right place

Try the macro on something it does not support:

```rust
use toolbox_macros::FieldNames;

#[derive(FieldNames)]
enum Color {
    Red,
    Green,
}

fn main() {}
```

```text
error: FieldNames can only be used on structs
 --> src/main.rs:4:6
  |
4 | enum Color {
  |      ^^^^^
```

The same for `struct Meters(f64);` gives `error: FieldNames needs a struct with named fields`, pointing at `Meters`.

> 📏 **Convention:** A procedural macro must never `panic!` for bad user input. Return `syn::Error::...to_compile_error()` with a clear message and the right span. A panic works, but the user sees `error: proc macro panicked` and your text only as a "help" line.

### What the compiler really compiles

`cargo expand` (Lesson 06.7) works for procedural macros too. For the `app` package, it shows:

```text
struct User {
    name: String,
    age: u32,
}
impl User {
    pub fn type_name() -> &'static str {
        "User"
    }
    pub fn field_names() -> &'static [&'static str] {
        &["name", "age"]
    }
}
struct Pair<T> {
    left: T,
    right: T,
}
impl<T> Pair<T> {
    pub fn type_name() -> &'static str {
        "Pair"
    }
    ...
}
```

The original struct is still there (derive macros **add**), and the `impl` is new. For `Pair<T>`, `split_for_impl` produced `impl<T> Pair<T>`.

## 5. An attribute macro

An attribute macro receives **two** token streams: the arguments in the brackets of the attribute, and the item that has the attribute. It returns the code that replaces the item.

We write `#[log_call]`: it makes a function print a line each time it is called.

First add `ItemFn` to the `use syn::{ ... }` line at the top of the file. Then add the macro:

```rust
/// Attribute macro: prints a line each time the function is called.
#[proc_macro_attribute]
pub fn log_call(_args: TokenStream, item: TokenStream) -> TokenStream {
    let mut function = parse_macro_input!(item as ItemFn);
    let name = function.sig.ident.to_string();
    let body = &function.block;
    function.block = syn::parse_quote!({
        ::std::println!("[log] calling {}", #name);
        #body
    });
    quote! { #function }.into()
}
```

Use it:

```rust
use toolbox_macros::log_call;

#[log_call]
fn add(a: i32, b: i32) -> i32 {
    a + b
}

fn main() {
    println!("{}", add(2, 3));
}
```

**What happens:**

1. `#[proc_macro_attribute]` marks a macro used as `#[log_call]`. The first parameter is the arguments (empty here, so `_args`). The second is the item.
2. `ItemFn` is the AST of a whole function: signature (`sig`), visibility, attributes and body (`block`). It needs `syn`'s `"full"` feature.
3. We read the function's name from `function.sig.ident`.
4. `syn::parse_quote!` is like `quote!`, but it parses the tokens into a `syn` type, here a block. The new block first prints, then runs the old body (`#body`).
5. `quote! { #function }` turns the changed function back into tokens. The compiler compiles *this* instead of the original function.
6. We wrote `::std::println!` with a leading `::`. The macro's output is pasted into the user's code, where the user might have their own `println`. Starting with `::std` always means the real standard library.

```text
[log] calling add
5
```

> 📏 **Convention:** In generated code, use full paths (`::std::vec::Vec`, `::core::option::Option`), not short names. Procedural macros are **not hygienic** like `macro_rules!` for names: generated code can clash with the user's names.

## 6. Debugging a procedural macro

1. **`cargo expand`** shows the final code (see above). Look here first.
2. **`eprintln!` inside the macro** prints while the compiler runs. The text appears in the output of `cargo build`. Use `eprintln!("{:#?}", ast)` (with the `"extra-traits"` feature) to see the tree. Remove it before you commit.
3. **A small test crate** with one struct. Make it fail fast and read the first error: macro errors cascade.
4. **`cargo build -p app`** rebuilds only what changed. When you edit the macro crate, all crates that use it are rebuilt.

## 7. Where you see this in real projects

- `serde` is two crates: `serde` (traits `Serialize` and `Deserialize`) and `serde_derive` (the derive macros). The `serde` crate re-exports the derives (when you turn on its `derive` feature), so users write `use serde::Serialize;` and get both. This is the standard way to ship a trait with its derive macro, because a `proc-macro` crate can export *only* macros, not traits.
- `clap` (`#[derive(Parser)]`), `thiserror` (`#[derive(Error)]`) and `tokio` (`#[tokio::main]`, an attribute macro you will use in [Lesson 06.9](09-async-await-introduction.md)) are procedural macros too.

> 🔒 **Security:** A procedural macro is a program that the compiler runs on your computer while you build. It can read your files and use the network, with your permissions, exactly like a `build.rs` script. A malicious macro in a dependency is a *supply-chain* risk (an attack through the libraries you use). We study this in the security phase. For now: add only crates you trust, and look at the crate's popularity and maintainers.

> ⚡ **Performance:** `syn` with `"full"` and `"extra-traits"` is slow to compile. Enable only the features you need. A derive macro adds compile time for every crate that uses it, but has no run-time cost: the generated code is normal code.

## 8. Testing a procedural macro

There are three levels, from cheap to precise.

**1. Integration tests in the user crate.** Put a test file in `app/tests/`. It uses the macro as a user would:

```rust
use toolbox_macros::{shout_const, FieldNames};

#[derive(FieldNames)]
struct Point {
    x: i32,
    y: i32,
}

#[test]
fn derive_lists_field_names() {
    assert_eq!(Point::type_name(), "Point");
    assert_eq!(Point::field_names(), ["x", "y"]);
}

#[test]
fn shout_const_makes_upper_case() {
    assert_eq!(shout_const!("rust"), "RUST");
}

#[test]
fn point_fields_are_usable() {
    let p = Point { x: 1, y: 2 };
    assert_eq!(p.x + p.y, 3);
}
```

**2. Unit tests of the expansion.** Because `expand_field_names` takes and returns `proc_macro2` types, it can be called from an ordinary `#[test]` inside the macro crate. Add to the end of `toolbox-macros/src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn struct_gets_both_functions() {
        let input = quote! { struct Point { x: i32, y: i32 } };
        let output = expand_field_names(input).to_string();
        assert!(output.contains("type_name"));
        assert!(output.contains("\"x\""));
        assert!(output.contains("\"y\""));
    }

    #[test]
    fn enum_is_rejected_with_a_compile_error() {
        let input = quote! { enum Color { Red, Green } };
        let output = expand_field_names(input).to_string();
        assert!(output.contains("compile_error"));
        assert!(output.contains("only be used on structs"));
    }
}
```

Functions that use `proc_macro::TokenStream` (without the `2`) panic when called outside a macro call, so we keep the logic in `proc_macro2` functions.

**3. Compile-fail tests with `trybuild`.** You want to be sure the error message is good. The `trybuild` crate compiles small files and compares the compiler's output with a saved `.stderr` file. In `app/Cargo.toml`:

```toml
[dev-dependencies]
trybuild = "1.0"
```

`app/tests/ui.rs`:

```rust
#[test]
fn ui() {
    let t = trybuild::TestCases::new();
    t.compile_fail("tests/ui/*.rs");
}
```

Put the `enum Color` example in `app/tests/ui/enum.rs`. The first run writes the compiler output to `app/wip/enum.stderr` and the test fails with `successfully created new stderr files`. Read it, and if it is what you want, move it:

```sh
mv app/wip/enum.stderr app/tests/ui/enum.stderr
cargo test
```

```text
running 1 test
test tests/ui/enum.rs ... ok
test ui ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.13s
```

> 🧪 **Testing:** Test the happy path in `tests/`, the logic with `proc_macro2` unit tests, and the error messages with `trybuild`. Your users read those messages, so they are part of your API (Application Programming Interface: what your code offers to the people who use it).

## Common mistakes

**1. Forgetting `proc-macro = true`.**

```text
error: the `proc_macro` attribute is only usable with crates of the `proc-macro` crate type
 --> src/lib.rs:4:3
  |
4 | #[proc_macro]
  |   ^^^^^^^^^^
```

Fix: add `[lib] proc-macro = true` to the macro crate's `Cargo.toml`.

**2. Exporting normal items from a macro crate.**

```text
error: `proc-macro` crate types currently cannot export any items other than functions tagged with `#[proc_macro]`, `#[proc_macro_derive]`, or `#[proc_macro_attribute]`
 --> src/lib.rs:4:1
  |
4 | pub const ANSWER: i32 = 42;
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

Fix: a macro crate may export only macros. Make helpers private (no `pub`), or put shared code in a third normal crate (this is why `serde` and `serde_derive` are separate).

**3. Returning the wrong `TokenStream` type.**

```text
error[E0308]: mismatched types
  --> src/lib.rs:12:5
   |
11 | pub fn five(_input: TokenStream) -> TokenStream {
   |                                     ----------- expected `TokenStream` because of return type
12 |     quote! { 5 }
   |     ^^^^^^^^^^^^ expected `TokenStream`, found `proc_macro2::TokenStream`
   |
help: call `Into::into` on this expression to convert `proc_macro2::TokenStream` into `TokenStream`
```

Fix: add `.into()`: `quote! { 5 }.into()`.

**4. Forgetting a `syn` feature.** Using `ItemFn` without the `"full"` feature:

```text
error[E0432]: unresolved import `syn::ItemFn`
note: found an item that was configured out
  |
  | #[cfg(feature = "full")]
  |       ---------------- the item is gated behind the `full` feature
```

Fix: `syn = { version = "3.0", features = ["full"] }`.

**5. Panicking on bad input.** A `panic!` or `unwrap()` in a macro gives the user:

```text
error: proc macro panicked
 --> src/main.rs:2:5
  |
2 |     oops::boom!();
  |     ^^^^^^^^^^^^^
  |
  = help: message: I do not like this input
```

It works, but it does not point at the real problem. Return a `syn::Error` with a span.

**6. Using a procedural macro in the same crate that defines it.** It does not work: the macro must be compiled before the code that uses it. Always use a second package (or a test in `tests/`).

## Try it yourself

1. Build the workspace from sections 2–5. Run `cargo run -p app` and check you see `WELCOME`, the field names and `[log] calling add`.
2. Run `cargo expand -p app` and find the `impl User` block.
3. Break it on purpose: change `shout_const!("welcome")` into `shout_const!(42)`. Read the error and find the span.
4. Run `cargo test --workspace`. You should see the integration tests, the unit tests and the `trybuild` test pass.
5. Commit your work (the learning repository from Phase 01):

   ```sh
   git add code/p06-procmacros
   git commit -m "feat(p06-procmacros): add shout_const, FieldNames and log_call macros"
   ```

   Do not commit the `target` folder. If `git status` shows it, add `target/` to your `.gitignore` first.

## Exercises

All exercises extend the `toolbox-macros` crate from this lesson.

### Exercise 1 — `const_len!` (easy)

Write a function-like macro `const_len!("hello")` that expands to the number of bytes of the string literal: `5`. Check it with `const N: usize = const_len!("hello");`.

<details>
<summary>Hint</summary>

Copy `shout_const`. Instead of `to_uppercase()`, call `text.value().len()`. A `usize` variable can be inserted with `#len` in `quote!`.

</details>

<details>
<summary>Answer</summary>

```rust
/// `const_len!("text")` becomes the number `4` (the length in bytes).
#[proc_macro]
pub fn const_len(input: TokenStream) -> TokenStream {
    let text = parse_macro_input!(input as LitStr);
    let len = text.value().len();
    quote! { #len }.into()
}
```

Test (in `app/tests/derive.rs`):

```rust
use toolbox_macros::const_len;

#[test]
fn const_len_counts_bytes() {
    const N: usize = const_len!("hello");
    assert_eq!(N, 5);
}
```

`cargo test` shows `test const_len_counts_bytes ... ok`.

Why it works: the length is computed by the compiler. `N` is a constant even though the macro ran real code.

</details>

### Exercise 2 — `field_count()` with a test (medium)

Extend `FieldNames` so it also generates `pub fn field_count() -> usize`. Add a unit test in the macro crate that checks the generated tokens contain `field_count`, and extend the integration test to check that `Point::field_count()` is `2`.

<details>
<summary>Hint</summary>

Inside the `quote!` block, add a second function whose body is `Self::field_names().len()`. In the test, `output.contains("field_count")` is enough.

</details>

<details>
<summary>Answer</summary>

Inside the `impl` in `quote!`, after `field_names`:

```rust
            pub fn field_count() -> usize {
                Self::field_names().len()
            }
```

Unit test in the macro crate:

```rust
    #[test]
    fn struct_gets_field_count() {
        let input = quote! { struct Point { x: i32, y: i32 } };
        let output = expand_field_names(input).to_string();
        assert!(output.contains("field_count"));
    }
```

In the integration test, add `assert_eq!(Point::field_count(), 2);` to `derive_lists_field_names`.

Why it works: `field_count` reuses the generated `field_names`, so there is only one place that knows the list.

</details>

### Exercise 3 — `#[log_call("DEBUG")]` (medium)

Make `log_call` accept an optional tag in brackets. `#[log_call("DEBUG")]` prints `[DEBUG] calling add`; plain `#[log_call]` still prints `[log] calling add`.

<details>
<summary>Hint</summary>

The first parameter of the attribute macro (`args`) holds the tokens in the brackets. Check `args.is_empty()` first. If it is not empty, parse it as a `LitStr` with `parse_macro_input!(args as LitStr)` and take `.value()`. Insert the tag with `#tag`.

</details>

<details>
<summary>Answer</summary>

```rust
#[proc_macro_attribute]
pub fn log_call(args: TokenStream, item: TokenStream) -> TokenStream {
    let tag = if args.is_empty() {
        "log".to_string()
    } else {
        parse_macro_input!(args as LitStr).value()
    };
    let mut function = parse_macro_input!(item as ItemFn);
    let name = function.sig.ident.to_string();
    let body = &function.block;
    function.block = syn::parse_quote!({
        ::std::println!("[{}] calling {}", #tag, #name);
        #body
    });
    quote! { #function }.into()
}
```

Test:

```rust
use toolbox_macros::log_call;

#[log_call("DEBUG")]
fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[test]
fn tagged_log_still_returns_value() {
    assert_eq!(add(2, 3), 5);
}
```

Run with `cargo test -- --nocapture` to see the line `[DEBUG] calling add`.

Why it works: the attribute arguments and the item are two separate token streams, so the macro can read both.

</details>

### Exercise 4 — a `trybuild` test for a tuple struct (medium)

Add a compile-fail test showing that `#[derive(FieldNames)] struct Meters(f64);` is rejected with the message `FieldNames needs a struct with named fields`.

<details>
<summary>Hint</summary>

Create `app/tests/ui/tuple.rs` with the struct and an empty `fn main() {}`. Run `cargo test`, then move the file from `app/wip/` to `app/tests/ui/` like for the enum.

</details>

<details>
<summary>Answer</summary>

`app/tests/ui/tuple.rs`:

```rust
use toolbox_macros::FieldNames;

#[derive(FieldNames)]
struct Meters(f64);

fn main() {}
```

`app/tests/ui/tuple.stderr` (this is the real output):

```text
error: FieldNames needs a struct with named fields
 --> tests/ui/tuple.rs:4:8
  |
4 | struct Meters(f64);
  |        ^^^^^^
```

`cargo test` then shows `test tests/ui/tuple.rs ... ok`.

Why it works: `trybuild` compiles every file that matches `tests/ui/*.rs` and compares the compiler output with the `.stderr` file next to it. If someone breaks the error message later, the test fails.

</details>

### Exercise 5 — a `VariantNames` derive for enums (hard)

Write `#[derive(VariantNames)]` that adds `pub fn variant_names() -> &'static [&'static str]` to an **enum**. For `enum Color { Red, Green, Blue }`, it returns `["Red", "Green", "Blue"]`. On a struct it must give a compile error with a clear message.

<details>
<summary>Hint</summary>

Match on `ast.data`: this time you need `Data::Enum(data)`, and `data.variants` is the list. Each variant has an `ident`. Use `let ... else` to return the error early: `let Data::Enum(data) = &ast.data else { return ... };`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Derive macro: adds `variant_names()` to an enum.
#[proc_macro_derive(VariantNames)]
pub fn derive_variant_names(input: TokenStream) -> TokenStream {
    let ast = parse_macro_input!(input as DeriveInput);
    let name = &ast.ident;

    let Data::Enum(data) = &ast.data else {
        return syn::Error::new_spanned(name, "VariantNames can only be used on enums")
            .to_compile_error()
            .into();
    };

    let variant_names = data.variants.iter().map(|v| v.ident.to_string());
    let (impl_generics, ty_generics, where_clause) = ast.generics.split_for_impl();

    quote! {
        impl #impl_generics #name #ty_generics #where_clause {
            pub fn variant_names() -> &'static [&'static str] {
                &[#(#variant_names),*]
            }
        }
    }
    .into()
}
```

Test:

```rust
use toolbox_macros::VariantNames;

#[derive(VariantNames)]
enum Color {
    Red,
    Green,
    Blue,
}

#[test]
fn variant_names_for_enum() {
    assert_eq!(Color::variant_names(), ["Red", "Green", "Blue"]);
    let _ = (Color::Red, Color::Green, Color::Blue);
}
```

Why it works: it has the same shape as `FieldNames`. Only the part of the tree you walk is different: `Data::Enum` and `variants` instead of `Data::Struct` and `fields`.

</details>

## Quiz

1. What are the three kinds of procedural macros? Give one example of each.

<details>
<summary>Answer</summary>

Function-like (`shout_const!("x")`), derive (`#[derive(FieldNames)]`) and attribute (`#[log_call]`). Real examples: `serde`'s `Serialize` is a derive, and `#[tokio::main]` is an attribute.

</details>

2. Why does a procedural macro need its own crate?

<details>
<summary>Answer</summary>

The compiler runs the macro while compiling the code that uses it, so the macro must be compiled first, as a separate library with `proc-macro = true`. A crate of this type can export only macros.

</details>

3. What do `syn` and `quote` do?

<details>
<summary>Answer</summary>

`syn` parses a `TokenStream` into a tree (AST) that you can inspect with normal Rust code. `quote!` builds a `TokenStream` from Rust-like code with `#variable` holes.

</details>

4. What is the difference between `proc_macro::TokenStream` and `proc_macro2::TokenStream`? Why use the second?

<details>
<summary>Answer</summary>

The first exists only inside the compiler and cannot be used in ordinary tests. The second is a normal type that works anywhere. `quote!` and `syn` use `proc_macro2`. You convert with `.into()` at the edges of the macro function.

</details>

5. How should a macro report that the user used it wrongly?

<details>
<summary>Answer</summary>

Return `syn::Error::new_spanned(tokens, "message").to_compile_error()`. The compiler then prints the message and points at the given tokens. Do not use `panic!`.

</details>

6. Why does a derive macro use `split_for_impl()`?

<details>
<summary>Answer</summary>

It splits the generic parameters of the type into the three pieces an `impl` needs (`impl<T>`, `Type<T>`, and the `where` clause), so the generated `impl` also works for generic types.

</details>

7. Why can a procedural macro be a security risk?

<details>
<summary>Answer</summary>

It is a program that runs on your computer during the build, with your permissions. A malicious macro in a dependency could read files or use the network.

</details>

## Summary

- A procedural macro is a Rust function that turns a `TokenStream` into a new `TokenStream` at compile time.
- There are three kinds: function-like (`name!`), derive (`#[derive(Name)]`) and attribute (`#[name]`).
- They live in a library crate with `proc-macro = true`, which can export only macros. Use a workspace with a second package that uses them.
- `syn` parses code into an AST (`LitStr`, `DeriveInput`, `ItemFn`). `quote!` writes code back, with `#var` and `#(#items),*`.
- Report mistakes with `syn::Error::new_spanned(...).to_compile_error()`, not with `panic!`.
- In generated code use full paths such as `::std::println!`.
- Debug with `cargo expand` and `eprintln!`. Test with integration tests, `proc_macro2` unit tests and `trybuild`.
- A macro runs code on your machine at build time, so trust your macro dependencies.

## Sources and further reading

- [The Rust Programming Language — 20.5 Macros (procedural macros)](https://doc.rust-lang.org/book/ch20-05-macros.html) — the book's introduction to the three kinds, with a derive example.
- [The Rust Reference — Procedural macros](https://doc.rust-lang.org/reference/procedural-macros.html) — the exact rules for each kind.
- [syn on docs.rs](https://docs.rs/syn/3/syn/) — the full list of AST types, features and the `parse_macro_input!` macro.
- [quote on docs.rs](https://docs.rs/quote/latest/quote/) — the `quote!` syntax: `#var`, repetitions and `ToTokens`.
- [proc-macro-workshop](https://github.com/dtolnay/proc-macro-workshop) — exercises (a builder derive, a sorted attribute, and more) from the author of `syn` and `quote`.
- [trybuild](https://docs.rs/trybuild/latest/trybuild/) — compile-fail tests for macros.

---

[🏠 Start](../START-HERE.md) · [← 06.7 Declarative macros](07-declarative-macros.md) · [📋 Phase 06](README.md) · [06.9 Introduction to async/await →](09-async-await-introduction.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
