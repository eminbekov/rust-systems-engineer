<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 04.9 Hash maps](09-hash-maps.md) · [📋 Phase 04](README.md) · [04.11 Project: phone book →](11-project-phone-book.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/04-structs-enums-collections-errors/10-error-handling.md) <!-- nav -->

# Lesson 04.10 — Error handling

> **Phase 04 — Structs, enums, collections and errors** · Lesson 10 of 12 · ⏱ about 5 hours (3 sittings)

## What you will learn

- The difference between a bug (use `panic!`) and an expected failure (use `Result`).
- How to read and write `Result<T, E>`, and how `unwrap`, `expect`, `match` and helper methods deal with it.
- How the `?` operator passes an error up to the caller in one character.
- How to design your own error type with `Display`, `Error` and `From`, and when `Box<dyn Error>` is enough.
- How to test error cases and how a program reports failure to the shell with exit codes.

## Before you start

- You finished [Lesson 04.9 — Hash maps](09-hash-maps.md).
- You know enums, `Option` and `match` ([Lesson 04.3](03-enums.md), [Lesson 04.4](04-option-and-match.md)), `if let` and `let else` ([Lesson 04.5](05-if-let-let-else-and-patterns.md)), strings and `parse` ([Lesson 04.8](08-strings-in-depth.md)).
- Create a project and two small data files:

```bash
cd ~/projects/rust
cargo new code/p04-errors
cd code/p04-errors
printf '42\n' > number.txt
printf 'forty-two\n' > bad_number.txt
```

## New words

| Word | Meaning |
|---|---|
| **error** | Something that went wrong while the program runs: a file is missing, the user typed text where a number was expected, the network is down. |
| **bug** | A mistake in the program itself. The code is wrong, not the outside world. |
| **recoverable error** | An error the program can reasonably handle (try again, show a message, use a default). Rust models it with `Result`. |
| **panic** | A deliberate crash for a situation the program cannot continue from, almost always a bug. Started by `panic!` or by a failed `unwrap`. |
| **unwinding** | What a panic does by default: Rust walks back up through the running functions, cleaning up each one's values, and then ends the thread. |
| **backtrace** | A list of the functions that were running when the panic happened, from the innermost outward. |
| **`Result<T, E>`** | An enum with two variants: `Ok(T)` holds the success value, `Err(E)` holds the error value. |
| **`unwrap`** | A method that returns the value inside `Ok` / `Some`, and panics if there is an `Err` / `None`. |
| **`expect`** | Like `unwrap`, but you give the panic message. |
| **propagate** | To pass an error up to the caller instead of handling it where it happened. |
| **`?` operator** | A short way to propagate: on `Ok(v)` it gives `v`; on `Err(e)` it returns `Err(e)` from the current function. |
| **I/O** (Input/Output) | Reading from and writing to the outside world: files, the keyboard, the network. I/O is the most common source of errors. |
| **`impl` block** | A place where you write the methods of a type, or the implementation of a trait for a type (`impl Trait for Type`). |
| **`Display`** | The trait behind `{}`. It decides how a value looks for a human reader. You implement it to give your error a message. |
| **`Error` trait** | The standard trait (`std::error::Error`) that marks a type as an error. |
| **`From` trait** | The standard conversion trait. `impl From<A> for B` says how to turn an `A` into a `B`; the `?` operator uses it. |
| **`Box<dyn Error>`** | A box (a heap pointer) that can hold any value that is an `Error`. A catch-all error type. (`dyn` is explained in [Lesson 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md).) |
| **match guard** | An extra `if` condition after a pattern in a `match` arm: `Ok(n) if n > 0 => ...`. The arm is used only when the condition is true. |
| **stdout / stderr** (standard output / standard error) | Two output channels of a program. Normal results go to stdout (`println!`), error messages to stderr (`eprintln!`). |
| **exit code** | A number a program returns to the shell when it ends. `0` means success; any other number means failure. |

## 1. Two kinds of problems

Programs fail in two very different ways.

1. **A bug.** The programmer made a mistake. An index was out of range, a "can never happen"
   case happened. Nobody can fix this while the program runs; the right reaction is to stop
   loudly so the bug is found and fixed.
2. **An expected failure.** The program is correct, but the world is not: the file does not exist,
   the user typed `abc` where a number is needed, the disk is full. A good program must **expect**
   these and react: show a message, ask again, use a default.

Rust uses two different tools for these two kinds of problems, so you always see which kind you
have:

| Situation | Tool | Result |
|---|---|---|
| Bug, broken promise, "impossible" state | `panic!` | the program stops (with a message) |
| Failure that can happen in correct code | `Result<T, E>` | the caller gets a value and decides |
| A value that may simply be absent | `Option<T>` | `Some(v)` or `None` |

The compiler helps: a function that can fail says so **in its type**, and you cannot use the
success value without dealing with the possibility of an error.

## 2. `panic!`: stop on a bug

```rust
fn main() {
    println!("start");
    let data: Vec<i32> = Vec::new();
    if data.is_empty() {
        panic!("the data list must never be empty here");
    }
    println!("never printed");
}
```

```text
start

thread 'main' (3973753) panicked at src/main.rs:5:9:
the data list must never be empty here
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**What happens:**

1. `panic!("...")` prints the message, the thread name and the place in the code (file, line, column),
   then starts **unwinding**: Rust walks out of the running functions, freeing the values they own.
2. Because this was the main thread, the whole program ends. Its **exit code** is `101` (we
   checked: `echo $?` prints `101`). Exit codes are explained in section 10.
3. `never printed` never appears: the code after the panic does not run.

To see where the panic came from, run with the environment variable `RUST_BACKTRACE=1`:
`RUST_BACKTRACE=1 cargo run`. Rust then prints the **backtrace**, the list of function calls that led
to the panic; look for lines that name your own files.

Many mistakes cause panics without you writing `panic!`: an out-of-range index
([Lesson 04.7](07-vectors.md)), a missing key with `map[key]` ([Lesson 04.9](09-hash-maps.md)), division by zero for
integers, integer overflow in debug builds ([Lesson 02.2](../02-programming-basics/02-data-types.md)).

### `unwrap` and `expect`

`unwrap` and `expect` are shortcuts: "give me the value, and **panic** if there is none."

```rust
fn main() {
    let good: i32 = "42".parse().unwrap();
    println!("good = {good}");
    let bad: i32 = "forty-two".parse().unwrap();
    println!("bad = {bad}");
}
```

```text
good = 42

thread 'main' (3973991) panicked at src/main.rs:4:40:
called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

`parse` returns a `Result` (next section). `unwrap()` on the `Err` panics and prints the error value
in debug form. `expect` does the same, but you choose the message:

```rust
fn main() {
    let text = "17x";
    let n: i32 = text.parse().expect("the settings file should contain a number");
    println!("{n}");
}
```

```text

thread 'main' (3974235) panicked at src/main.rs:3:31:
the settings file should contain a number: ParseIntError { kind: InvalidDigit }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

> 📏 **Convention:** If you must panic, prefer `expect` over `unwrap`, and write what you
> *expected* to be true ("the settings file should contain a number"), not what went wrong. The
> message helps whoever reads the crash report, maybe you in six months.

When is a panic acceptable?

- in **tests** (a panic is how a test fails);
- in **small examples and quick experiments**;
- when you **can prove** the value is there. For example `"42".parse::<i32>().unwrap()` on a literal
  that you wrote yourself, or after you checked `is_some()`.
- when continuing would be worse than stopping (a broken internal promise).

> 🔒 **Security:** Never `unwrap()` a value that comes from outside the program: user input, a file,
> the network. An attacker (or just a typo) turns your `unwrap` into a crash, and a crashing server
> is a denial of service. Handle the `Err` instead.

## 3. `Result<T, E>`: handle an expected failure

`Result` is a normal enum from the standard library (and, like `Option`, it is always in scope):

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

`T` is the type of the success value, `E` the type of the error value. (The letters stand for types you
choose; the general feature is called generics, [Lesson 05.1](../05-rust-intermediate/01-generics.md).) For example `Result<i32, ParseIntError>` is
"either an `i32` or an error that says why parsing failed". You handle it with `match`, as you did
with `Option`:

```rust
fn main() {
    for text in ["42", "-7", "forty-two", "", "99999999999"] {
        match text.parse::<i32>() {
            Ok(n) => println!("{:?} -> number {}", text, n),
            Err(e) => println!("{:?} -> error: {}", text, e),
        }
    }
}
```

**What happens:**

1. `text.parse::<i32>()` tries to read an `i32`. It returns `Result<i32, ParseIntError>`.
2. The `Ok(n)` arm runs on success and gives the number. The `Err(e)` arm runs on failure; `e` is a
   `ParseIntError`, and `{}` prints a human-readable reason (it implements `Display`).
3. The error does not stop the program. Each of the five inputs is handled in its own way.

```text
"42" -> number 42
"-7" -> number -7
"forty-two" -> error: invalid digit found in string
"" -> error: cannot parse integer from empty string
"99999999999" -> error: number too large to fit in target type
```

### Helper methods

`match` is always correct, but for small jobs there are shorter methods:

```rust
fn main() {
    let a: Result<i32, String> = Ok(5);
    let b: Result<i32, String> = Err(String::from("broken"));

    println!("{}", a.is_ok());
    println!("{}", b.is_err());
    println!("{}", a.clone().unwrap_or(0));
    println!("{}", b.clone().unwrap_or(0));
    println!("{:?}", a.clone().ok());
    println!("{:?}", b.clone().ok());

    let parsed: Option<i32> = "12".parse().ok();
    println!("{:?}", parsed);

    let n: i32 = "oops".parse().unwrap_or_default();
    println!("{n}");

    let missing: Option<i32> = None;
    let r: Result<i32, String> = missing.ok_or(String::from("no value"));
    println!("{:?}", r);

    let e: Result<i32, String> = "x".parse::<i32>().map_err(|err| format!("bad number: {err}"));
    println!("{:?}", e);
}
```

**What happens:**

1. `is_ok()` and `is_err()` answer yes or no.
2. `unwrap_or(0)` gives the value inside `Ok`, or the fallback `0` for an `Err`. It never panics.
   (`.clone()` makes a copy so that we can use `a` and `b` again; you learned `clone` in
   [Lesson 03.3](../03-ownership-and-borrowing/03-move-copy-and-clone.md).)
3. `.ok()` turns a `Result<T, E>` into an `Option<T>`: `Ok(v)` becomes `Some(v)`, `Err(_)` becomes `None`
   (the error details are thrown away).
4. `unwrap_or_default()` uses the type's default value (`0` for `i32`) when there is an error.
5. `Option::ok_or(err)` goes the other way: `None` becomes `Err(err)`.
6. `map_err` changes the **error** part and leaves `Ok` alone. Its argument `|err| format!(...)` is a
   *closure*: a small nameless function written inline, where `err` is its parameter. Closures get
   their own lesson ([Lesson 05.4](../05-rust-intermediate/04-closures.md)); for now read it as "take the error and build a nicer
   message from it".

```text
true
true
5
0
Some(5)
None
Some(12)
0
Err("no value")
Err("bad number: invalid digit found in string")
```

> ⚠️ **Common mistake:** Using `.ok()` or `unwrap_or_default()` everywhere "to make the error go away."
> That hides real problems: the program continues with a wrong value and nobody knows why. Throw an
> error away only when you have decided that you truly do not care.

## 4. Errors from the outside world: files

Reading a file can fail in many ways: no such file, no permission, the content is not valid UTF-8 (the text encoding from [Lesson 04.8](08-strings-in-depth.md)). The standard
library function `std::fs::read_to_string` reads a whole file into a `String` and returns
`Result<String, std::io::Error>`. `io::Error` is the standard error type for **I/O** problems.

Create a file `config.txt` first: `printf 'port=8080\nname=demo\n' > config.txt`.

```rust
use std::fs;

fn main() {
    match fs::read_to_string("config.txt") {
        Ok(text) => println!("read {} bytes", text.len()),
        Err(e) => println!("cannot read: {e}"),
    }
    match fs::read_to_string("does-not-exist.txt") {
        Ok(text) => println!("read {} bytes", text.len()),
        Err(e) => println!("cannot read: {e}"),
    }
}
```

**What happens:**

1. The relative path `config.txt` is looked up in the *current directory*: the folder where you ran
   `cargo run` (the project folder).
2. The first call succeeds; the text has 20 bytes (`port=8080` + new line + `name=demo` + new line).
3. The second file does not exist, so the call returns `Err`. `{e}` prints the operating system's
   message: `No such file or directory (os error 2)`.

```text
read 20 bytes
cannot read: No such file or directory (os error 2)
```

## 5. The `?` operator

Now we write a function that reads a number from a file. It can fail in two ways: the file cannot
be read, and the text is not a number. With `match` only:

```rust
use std::fs;
use std::io;
use std::num::ParseIntError;

fn read_number(path: &str) -> Result<i32, String> {
    let text = match fs::read_to_string(path) {
        Ok(t) => t,
        Err(e) => return Err(format!("cannot read {path}: {e}")),
    };
    match text.trim().parse::<i32>() {
        Ok(n) => Ok(n),
        Err(e) => Err(format!("{path} does not contain a number: {e}")),
    }
}

fn read_number_with_question_mark(path: &str) -> Result<i32, io::Error> {
    let text = fs::read_to_string(path)?;
    Ok(text.trim().len() as i32)
}

fn parse_twice(a: &str, b: &str) -> Result<i32, ParseIntError> {
    let x: i32 = a.parse()?;
    let y: i32 = b.parse()?;
    Ok(x + y)
}

fn main() {
    println!("{:?}", read_number("number.txt"));
    println!("{:?}", read_number("bad_number.txt"));
    println!("{:?}", read_number("nothing.txt"));
    println!("{:?}", read_number_with_question_mark("number.txt"));
    println!("{:?}", parse_twice("40", "2"));
    println!("{:?}", parse_twice("40", "two"));
}
```

**What happens:**

1. `read_number` handles each failure with `match` and `return Err(...)`. It works but it is long.
   It also converts both errors to a `String` message.
2. `let text = fs::read_to_string(path)?;` is the same idea in one character. The **`?` operator** looks at
   the `Result` on its left:
   - if it is `Ok(v)`, the whole expression becomes `v` and the function goes on;
   - if it is `Err(e)`, the function **returns `Err(e)` immediately**.
3. In `read_number_with_question_mark`, the function's own error type is `io::Error`, the same as the
   error of `read_to_string`, so the error passes through unchanged. (The `.len()` is only there to
   make a number.)
4. `parse_twice` uses `?` twice. If `a` is bad we return before even looking at `b`. The success path
   reads like straight code, with no nesting.

```text
Ok(42)
Err("bad_number.txt does not contain a number: invalid digit found in string")
Err("cannot read nothing.txt: No such file or directory (os error 2)")
Ok(2)
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
```

What `?` does, written out (simplified):

```text
let x = a.parse::<i32>()?;      is roughly:

let x = match a.parse::<i32>() {
    Ok(v)  => v,
    Err(e) => return Err(From::from(e)),   // converts the error type if needed (section 7)
};
```

`?` also works on `Option` in a function that returns `Option`: `None` makes the function return `None`.

### `?` only works in functions that return `Result` (or `Option`)

```rust
use std::fs;

fn main() {
    let text = fs::read_to_string("config.txt")?;
    println!("{text}");
}
```

```text
error[E0277]: the `?` operator can only be used in a function that returns `Result` or `Option` (or another type that implements `FromResidual`)
 --> src/main.rs:4:48
  |
3 | fn main() {
  | --------- this function should return `Result` or `Option` to accept `?`
4 |     let text = fs::read_to_string("config.txt")?;
  |                                                ^ cannot use the `?` operator in a function that returns `()`
  |
help: consider adding return type
  |
3 ~ fn main() -> Result<(), Box<dyn std::error::Error>> {
```

**What the error means:** `?` needs somewhere to send the error: the function's return value. `main`
returns `()` (nothing), so there is no place. The compiler suggests the fix. In the next section
we do exactly that.

## 6. `main` can return a `Result`

```rust
use std::error::Error;
use std::fs;

fn main() -> Result<(), Box<dyn Error>> {
    let text = fs::read_to_string("nums.txt")?;
    let mut total = 0;
    for line in text.lines() {
        let n: i32 = line.trim().parse()?;
        total += n;
    }
    println!("total = {total}");
    Ok(())
}
```

**What happens:**

1. `-> Result<(), Box<dyn Error>>` means: on success `main` gives `()`, on failure it gives *some
   error*. `Box<dyn Error>` is a catch-all error type (section 8).
2. Both `?` can now fail: reading the file and parsing a line. Different error types are converted
   automatically into `Box<dyn Error>`.
3. `Ok(())` at the end is the success value: "all good, nothing to return". Note: no semicolon
   after the last expression.
4. If `main` returns `Err`, Rust prints `Error: ` followed by the error in debug form and ends the
   program with exit code `1`.

Try it with three different situations. In a terminal in the project folder:

```bash
printf '10\n20\n30\n' > nums.txt
cargo run -q
printf '10\nabc\n' > nums.txt
cargo run -q
rm nums.txt
cargo run -q
```

```text
total = 60
Error: ParseIntError { kind: InvalidDigit }
Error: Os { code: 2, kind: NotFound, message: "No such file or directory" }
```

The second and third runs end with exit code 1 (`echo $?` shows it). The messages are correct but
not very friendly (`Error: ParseIntError { kind: InvalidDigit }`). In the next sections we build
better ones.

## 7. Your own error type

Strings as errors are fine for tiny programs, but they cannot be compared, matched or extended.
A real error type is an **enum** with one variant per kind of failure.

```rust
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug)]
enum ConfigError {
    MissingKey(String),
    BadNumber { key: String, source: ParseIntError },
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ConfigError::MissingKey(key) => write!(f, "missing key '{key}'"),
            ConfigError::BadNumber { key, source } => {
                write!(f, "key '{key}' is not a number: {source}")
            }
        }
    }
}

impl std::error::Error for ConfigError {}

/// Finds `key=value` in the text and returns the value as a number.
fn get_number(text: &str, key: &str) -> Result<u32, ConfigError> {
    for line in text.lines() {
        if let Some((k, v)) = line.split_once('=') {
            if k.trim() == key {
                return match v.trim().parse::<u32>() {
                    Ok(n) => Ok(n),
                    Err(e) => Err(ConfigError::BadNumber { key: key.to_string(), source: e }),
                };
            }
        }
    }
    Err(ConfigError::MissingKey(key.to_string()))
}

fn main() {
    let text = "port=8080\nname=demo\nworkers=many\n";
    for key in ["port", "workers", "timeout"] {
        match get_number(text, key) {
            Ok(n) => println!("{key} = {n}"),
            Err(e) => println!("error: {e}"),
        }
    }
    println!("{:?}", get_number(text, "timeout"));
}
```

**What happens:**

1. `ConfigError` has two variants. `MissingKey` carries the key name. `BadNumber` carries the key and the
   original `ParseIntError` (so no information is lost).
2. `impl fmt::Display for ConfigError` gives the error its human message. `impl Trait for Type` means
   "this type has the abilities of that trait" ([Lesson 05.2](../05-rust-intermediate/02-traits.md) teaches traits properly; for now copy the shape).
   The method `fmt` receives `&self` and a formatter `f`; `write!(f, "...")` works like `println!`, but writes into
   `f`. The `match self` produces a different text for each variant.
3. `impl std::error::Error for ConfigError {}` has an empty body. That is enough: `Error` has default
   behaviour for everything, but it requires the type to be `Debug` + `Display`. It lets the error be used
   with `Box<dyn Error>` and by other libraries.
4. `get_number` returns `Result<u32, ConfigError>`: every failure is one of our two variants.
5. `{e}` in `main` uses `Display` for a user-facing message; `{:?}` (the last line) uses `Debug`, which shows the
   variant for a developer.

```text
port = 8080
error: key 'workers' is not a number: invalid digit found in string
error: missing key 'timeout'
Err(MissingKey("timeout"))
```

### `From` makes `?` convert errors

If one function has several kinds of failure, you would like `?` to wrap each of them into your
error enum. For that, implement the `From` trait: `impl From<ParseIntError> for AppError` tells Rust how
to turn a `ParseIntError` into an `AppError`. The `?` operator calls it for you.

```rust
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    BadNumber(ParseIntError),
    Negative(i32),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            AppError::BadNumber(e) => write!(f, "not a number: {e}"),
            AppError::Negative(n) => write!(f, "{n} is negative"),
        }
    }
}

impl std::error::Error for AppError {}

impl From<ParseIntError> for AppError {
    fn from(e: ParseIntError) -> AppError {
        AppError::BadNumber(e)
    }
}

fn parse_positive(text: &str) -> Result<i32, AppError> {
    let n: i32 = text.trim().parse()?;
    if n < 0 {
        return Err(AppError::Negative(n));
    }
    Ok(n)
}

fn main() {
    for text in ["15", " 8 ", "-3", "abc"] {
        match parse_positive(text) {
            Ok(n) => println!("{:?} -> {n}", text),
            Err(e) => println!("{:?} -> error: {e}", text),
        }
    }
}
```

**What happens:**

1. `text.trim().parse()?` can fail with a `ParseIntError`, but the function returns `AppError`.
   The `?` operator sees the mismatch and calls `AppError::from(e)`, which wraps the error in the
   `BadNumber` variant.
2. A negative number is a *different* failure, created by hand: `return Err(AppError::Negative(n))`.
3. The caller sees one error type, `AppError`, no matter what went wrong inside.

```text
"15" -> 15
" 8 " -> 8
"-3" -> error: -3 is negative
"abc" -> error: not a number: invalid digit found in string
```

If you forget the `From` impl, the compiler tells you exactly what is missing:

```rust
use std::num::ParseIntError;

#[derive(Debug)]
enum AppError {
    BadNumber(ParseIntError),
}

fn parse(text: &str) -> Result<i32, AppError> {
    let n = text.parse::<i32>()?;
    Ok(n)
}

fn main() {
    println!("{:?}", parse("5"));
}
```

```text
error[E0277]: `?` couldn't convert the error to `AppError`
 --> src/main.rs:9:32
  |
8 | fn parse(text: &str) -> Result<i32, AppError> {
  |                         --------------------- expected `AppError` because of this
9 |     let n = text.parse::<i32>()?;
  |                  --------------^ the trait `From<ParseIntError>` is not implemented for `AppError`
  |                  |
  |                  this can't be annotated with `?` because it has type `Result<_, ParseIntError>`
  |
note: `AppError` needs to implement `From<ParseIntError>`
```

The fix is the `impl From<ParseIntError> for AppError` from above.

> 📏 **Convention:** Name error types after what they are for: `ConfigError`, `ParseError`. Write
> messages that start with a lowercase letter and have no final period (`missing key 'port'`), so
> they can be put into longer sentences: `could not load config: missing key 'port'`.

## 8. `Box<dyn Error>`: one type for everything

Writing an error enum and its `From` impls takes effort. In a small program or in `main` you
often want: "any error is fine, just pass it up and print it." That is `Box<dyn Error>`:

```rust
use std::error::Error;
use std::fs;

fn run() -> Result<(), Box<dyn Error>> {
    let text = fs::read_to_string("number.txt")?;
    let n: i32 = text.trim().parse()?;
    println!("the number is {n}");

    let text2 = fs::read_to_string("bad_number.txt")?;
    let n2: i32 = text2.trim().parse()?;
    println!("the second number is {n2}");
    Ok(())
}

fn main() {
    if let Err(e) = run() {
        println!("error: {e}");
    }
}
```

**What happens:**

1. `run` has four possible failures with two different error types (`io::Error` and
   `ParseIntError`). `Box<dyn Error>` can hold any of them, and `?` converts each one automatically.
2. The first number is read and printed. The second file contains `forty-two`, so the second `parse()?` fails
   and `run` returns `Err`.
3. `main` handles the result with `if let Err(e)` and prints the message with `{e}` (the `Display`
   text), which is friendlier than the debug form.

```text
the number is 42
error: invalid digit found in string
```

`Box<dyn Error>` is a good fit for `main`, small tools, and prototypes. Its price is that the caller cannot
easily `match` on the exact kind of error. Libraries and bigger programs should use their own enum.

> 💡 **Tip:** Two popular crates remove most of the boilerplate of this section. **`thiserror`**
> generates `Display`, `Error` and `From` for your enum from a few attributes. **`anyhow`** gives a ready-made
> catch-all error type with context messages, for applications. You will learn both in
> [Lesson 05.11 — Practical crates](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md). Understand the manual way first, as here.

## 9. Which tool, when?

| Question | Answer |
|---|---|
| The caller made a promise and broke it (a bug) | `panic!` (or `assert!`) |
| Could this fail even when my code is correct (input, files, network)? | return `Result` |
| The value may simply be missing, and "missing" is not an error | return `Option` |
| Writing a **library** | your own error enum; never panic on bad input |
| Writing a small **application** or `main` | `Box<dyn Error>` (later: `anyhow`) |
| Prototype or test | `unwrap` / `expect` are fine |
| I want to ignore an error | decide explicitly, with a comment: `let _ = ...;` |

Rust also warns you when you forget a `Result` completely:

```rust
fn main() {
    "5".parse::<i32>();
    std::fs::remove_file("whatever.txt");
}
```

```text
warning: unused `Result` that must be used
 --> src/main.rs:2:5
  |
2 |     "5".parse::<i32>();
  |     ^^^^^^^^^^^^^^^^^^
  |
  = note: this `Result` may be an `Err` variant, which should be handled
  = note: `#[warn(unused_must_use)]` (part of `#[warn(unused)]`) on by default
help: use `let _ = ...` to ignore the resulting value
```

(We shortened the message. There is one such warning for each line.) Never leave this warning
in your code: it means an error may be silently dropped.

## 10. Reporting failure: stderr and exit codes

A command-line program talks to two audiences: a human in the terminal, and other programs or
scripts. Two conventions make the program a good citizen:

- **Messages go to the right stream.** `println!` writes to **stdout**, `eprintln!` writes to **stderr**.
  Results go to stdout, so they can be piped to another program; errors and warnings go to stderr, so
  they do not mix with the results.
- **The exit code tells success from failure.** `0` = success, anything else = failure. Shell scripts,
  build tools and CI (Continuous Integration) servers check it. `std::process::exit(code)` ends the program
  with a given code.

```rust
use std::process;

fn run(arg: &str) -> Result<u32, String> {
    arg.parse::<u32>().map_err(|e| format!("'{arg}' is not a valid count: {e}"))
}

fn main() {
    match run("ten") {
        Ok(n) => println!("count = {n}"),
        Err(message) => {
            eprintln!("error: {message}");
            process::exit(2);
        }
    }
}
```

**What happens:**

1. `run("ten")` fails, so the `Err` arm runs.
2. `eprintln!` writes the message to stderr.
3. `process::exit(2)` stops the program at once with exit code `2` (we checked: `echo $?` prints `2`).
   No code after it runs, and values still alive are not cleaned up, so use it only in `main`, at the very end.

```text
error: 'ten' is not a valid count: invalid digit found in string
```

> 📏 **Convention:** Exit code `0` for success, `1` for a general failure, `2` for wrong usage
> (bad arguments). `fn main() -> Result<...>` gives you `0` and `1` automatically.

> 🔒 **Security:** Error messages are information. Show users a short, useful message
> (`cannot open the settings file`). Do not print secrets (passwords, tokens), internal file paths
> or full debug dumps to people who should not see them; log the details privately instead.

## 11. Testing errors

> 🧪 **Testing:** Error paths are code too, and they break most often because nobody runs them. Test
> the good case **and** each way to fail. Useful tools:
> `assert_eq!(f(x), Err(...))` (the error type needs `PartialEq` and `Debug`), `assert!(f(x).is_err())`,
> `matches!(value, pattern)` (true when the value fits the pattern), `#[should_panic]` for functions
> that must panic, and test functions that return `Result`.

```rust
fn parse_age(text: &str) -> Result<u8, String> {
    match text.trim().parse::<u8>() {
        Ok(age) if age <= 150 => Ok(age),
        Ok(age) => Err(format!("{age} is not a realistic age")),
        Err(e) => Err(format!("invalid age: {e}")),
    }
}

fn main() {
    println!("{:?}", parse_age("30"));
    println!("{:?}", parse_age("200"));
    println!("{:?}", parse_age("-1"));
    println!("{:?}", parse_age("abc"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_a_normal_age() {
        assert_eq!(parse_age("30"), Ok(30));
    }

    #[test]
    fn rejects_unrealistic_age() {
        assert_eq!(parse_age("200"), Err(String::from("200 is not a realistic age")));
    }

    #[test]
    fn rejects_text() {
        assert!(parse_age("abc").is_err());
    }

    #[test]
    fn matches_macro() {
        assert!(matches!(parse_age("-1"), Err(_)));
    }

    #[test]
    #[should_panic(expected = "invalid age")]
    fn unwrap_on_error_panics() {
        parse_age("x").unwrap();
    }

    #[test]
    fn test_can_return_a_result() -> Result<(), String> {
        let age = parse_age(" 41 ")?;
        assert_eq!(age, 41);
        Ok(())
    }
}
```

**What happens:**

1. `Ok(age) if age <= 150 => ...` is a **match guard**: this arm is used only when the extra condition
   after `if` is true. A `u8` holds 0 to 255, so `200` parses fine but fails the guard and reaches the
   second arm. `-1` does not fit in `u8`, so it is a parse error.
2. `assert_eq!(parse_age("30"), Ok(30))` compares whole `Result` values; this works because `Result<u8, String>`
   implements `PartialEq` and `Debug`.
3. `#[should_panic(expected = "invalid age")]` makes the test pass only if the function panics and the panic
   message contains that text. Here `unwrap()` on the `Err` panics, and the message includes our text.
4. A test may return `Result<(), E>`. Then `?` works inside the test, and an `Err` fails the test.

```text
Ok(30)
Err("200 is not a realistic age")
Err("invalid age: invalid digit found in string")
Err("invalid age: invalid digit found in string")
```

```text
running 6 tests
test tests::accepts_a_normal_age ... ok
test tests::rejects_text ... ok
test tests::test_can_return_a_result ... ok
test tests::matches_macro ... ok
test tests::rejects_unrealistic_age ... ok
test tests::unwrap_on_error_panics - should panic ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## Common mistakes

**1. `unwrap()` on user input or a file.** It compiles, it works in your test, and it crashes in the
field. Handle the `Err` with `match`, `?` or a fallback (section 2).

**2. Using `?` in a function that returns `()`.** Error E0277 from section 5. Change the return type to
`Result<(), Box<dyn Error>>` (or another `Result`) and end the function with `Ok(())`.

**3. Forgetting `Ok(())` at the end of a function that returns `Result<(), E>`.**

```rust
fn show(n: i32) -> Result<(), String> {
    println!("{n}");
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:1:20
  |
1 | fn show(n: i32) -> Result<(), String> {
  |    ----            ^^^^^^^^^^^^^^^^^^ expected `Result<(), String>`, found `()`
  |    |
  |    implicitly returns `()` as its body has no tail or `return` expression
```

The body ends without a value, but the function must return a `Result`. Fix: add `Ok(())` as the
last line.

**4. Assigning a `Result` to a plain value.**

```rust
fn main() {
    let n: i32 = "5".parse();
    println!("{n}");
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:2:18
  |
2 |     let n: i32 = "5".parse();
  |            ---   ^^^^^^^^^^^ expected `i32`, found `Result<_, _>`
  |            |
  |            expected due to this
  |
help: consider using `Result::expect` to unwrap the `Result<_, _>` value, panicking if the value is a `Result::Err`
```

`parse` returns a `Result`, never a bare number. Handle it with `match`, `?`, or (when
acceptable) `unwrap` / `expect`.

**5. Different error types without a `From` impl.** Error E0277 from section 7. Implement
`From<TheirError> for YourError`, or use `Box<dyn Error>`.

**6. Throwing information away.** Writing `Err(_) => println!("error")` or `.ok()` loses the
reason. Print or return the error value.

## Try it yourself

Do the experiment from section 6 yourself, then extend it:

1. Replace `src/main.rs` with the `main` program of section 6 (the one with `nums.txt`). Create the
   file and run:

```bash
printf '10\n20\n30\n' > nums.txt
cargo run -q
```

Expected output:

```text
total = 60
```

2. Change `nums.txt` to contain a bad line and run again. Then check the exit code:

```bash
printf '10\nabc\n' > nums.txt
cargo run -q
echo $?
```

Expected output:

```text
Error: ParseIntError { kind: InvalidDigit }
1
```

3. Make the failure friendlier. Change `main` into a function `run()` that returns
   `Result<(), Box<dyn Error>>`, and write a new `main` like the one in section 8 that prints
   `error: {e}` to stderr with `eprintln!` and calls `std::process::exit(1)`.
4. Delete `nums.txt` and run again. What does the new message say?
5. Commit your work: `git add .` and `git commit -m "Practice error handling with Result and ?"`.
   (Do not commit `nums.txt`: if Git shows it, add it to `.gitignore`, as you learned in Phase 01.)

## Exercises

### Exercise 1 — Safe division (easy)

Write `fn divide(a: i32, b: i32) -> Result<i32, String>` that returns `Err("division by zero")` when `b`
is `0`. Call it with `(10, 2)`, `(1, 0)` and print the third call (`7, 0`) with a `match` that says `result: …` or
`error: …`. Expected output:

```text
Ok(5)
Err("division by zero")
error: division by zero
```

<details>
<summary>Hint</summary>

Check `b == 0` first and `return Err(String::from("division by zero"));`. Otherwise return `Ok(a / b)`.
In `main`, print the first two with `{:?}`.

</details>

<details>
<summary>Answer</summary>

```rust
fn divide(a: i32, b: i32) -> Result<i32, String> {
    if b == 0 {
        return Err(String::from("division by zero"));
    }
    Ok(a / b)
}

fn main() {
    println!("{:?}", divide(10, 2));
    println!("{:?}", divide(1, 0));
    match divide(7, 0) {
        Ok(n) => println!("result: {n}"),
        Err(e) => println!("error: {e}"),
    }
}
```

Why it works: the check happens before the division that would panic, and the caller decides how to react to the `Err`.

</details>

### Exercise 2 — Add two texts, with tests (easy)

Write `fn add_texts(a: &str, b: &str) -> Result<i32, ParseIntError>` that parses both texts (ignoring spaces
around them) and returns the sum. Use `?`. Write three tests: success, a bad first argument and a bad second one.
(`ParseIntError` lives in `std::num`.)

<details>
<summary>Hint</summary>

`let x: i32 = a.trim().parse()?;` then the same for `b`, then `Ok(x + y)`. Use `assert_eq!(…, Ok(42))` for success
and `.is_err()` for the failures.

</details>

<details>
<summary>Answer</summary>

```rust
use std::num::ParseIntError;

fn add_texts(a: &str, b: &str) -> Result<i32, ParseIntError> {
    let x: i32 = a.trim().parse()?;
    let y: i32 = b.trim().parse()?;
    Ok(x + y)
}

fn main() {
    println!("{:?}", add_texts("40", " 2 "));
    println!("{:?}", add_texts("40", "two"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_numbers() {
        assert_eq!(add_texts("40", " 2 "), Ok(42));
    }

    #[test]
    fn first_is_bad() {
        assert!(add_texts("x", "2").is_err());
    }

    #[test]
    fn second_is_bad() {
        assert!(add_texts("1", "").is_err());
    }
}
```

```text
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
```

Why it works: each `?` returns early with the `ParseIntError` if its parse fails; otherwise it unwraps the number.

</details>

### Exercise 3 — Errors from a file (medium)

Write `fn double_from_file(path: &str) -> Result<i32, Box<dyn Error>>` that reads a file, parses its
trimmed text as an `i32` and returns twice the number. In `main`, call it for `number.txt`, `bad_number.txt` and
`missing.txt` (the last does not exist) and print `path: result` or `path: error: message`. Expected output:

```text
number.txt: 84
bad_number.txt: error: invalid digit found in string
missing.txt: error: No such file or directory (os error 2)
```

<details>
<summary>Hint</summary>

`fs::read_to_string(path)?` then `text.trim().parse()?` into an `i32`. Import `std::error::Error` and `std::fs`. Run
the program from the project folder, where the two data files are.

</details>

<details>
<summary>Answer</summary>

```rust
use std::error::Error;
use std::fs;

fn double_from_file(path: &str) -> Result<i32, Box<dyn Error>> {
    let text = fs::read_to_string(path)?;
    let n: i32 = text.trim().parse()?;
    Ok(n * 2)
}

fn main() {
    for path in ["number.txt", "bad_number.txt", "missing.txt"] {
        match double_from_file(path) {
            Ok(n) => println!("{path}: {n}"),
            Err(e) => println!("{path}: error: {e}"),
        }
    }
}
```

Why it works: `Box<dyn Error>` accepts both `io::Error` and `ParseIntError`, and `?` converts them automatically.

</details>

### Exercise 4 — Add up lines, with the line number (medium)

Write `fn sum_lines(text: &str) -> Result<i64, LineError>`. Each line holds one integer; empty lines are
skipped. On the first bad line return a `LineError { line, problem }` with the line number (counting from 1)
and the `ParseIntError`. Implement `Display` for `LineError` so it prints `line 3: invalid digit found in string`.
Write three tests: a good text, an empty text and a bad line (check `err.line`).

<details>
<summary>Hint</summary>

Use `for (index, line) in text.lines().enumerate()`. Skip when `line.trim().is_empty()`. On
`Err(e)` from `parse::<i64>()` return `Err(LineError { line: index + 1, problem: e })`. Derive `Debug` and
`PartialEq` for the struct so the tests can compare. For the tests, `unwrap_err()` returns the error
inside an `Err` (and panics on `Ok`).

</details>

<details>
<summary>Answer</summary>

```rust
use std::fmt;
use std::num::ParseIntError;

#[derive(Debug, PartialEq)]
struct LineError {
    line: usize,
    problem: ParseIntError,
}

impl fmt::Display for LineError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "line {}: {}", self.line, self.problem)
    }
}

/// Adds up one integer per line. Empty lines are skipped.
/// On the first bad line, returns its number (counting from 1).
fn sum_lines(text: &str) -> Result<i64, LineError> {
    let mut total = 0;
    for (index, line) in text.lines().enumerate() {
        let line = line.trim();
        if line.is_empty() {
            continue;
        }
        match line.parse::<i64>() {
            Ok(n) => total += n,
            Err(e) => return Err(LineError { line: index + 1, problem: e }),
        }
    }
    Ok(total)
}

fn main() {
    println!("{:?}", sum_lines("10\n20\n\n30\n"));
    match sum_lines("10\n20\nthirty\n40\n") {
        Ok(total) => println!("total {total}"),
        Err(e) => println!("error: {e}"),
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sums_good_lines() {
        assert_eq!(sum_lines("10\n20\n\n30\n"), Ok(60));
    }

    #[test]
    fn empty_text_is_zero() {
        assert_eq!(sum_lines(""), Ok(0));
    }

    #[test]
    fn reports_line_number() {
        let err = sum_lines("1\n2\nx\n4").unwrap_err();
        assert_eq!(err.line, 3);
    }
}
```

```text
Ok(60)
error: line 3: invalid digit found in string
```

Why it works: the error is a struct that carries extra information (the line number) next to the original
cause. The function takes `&str` rather than a file name, so the tests need no files; a small wrapper that
reads the file and calls `sum_lines` can be added later.

</details>

### Exercise 5 — Parse a temperature with your own error type (hard)

Write `fn parse_temperature(text: &str) -> Result<f64, TempError>` for inputs like `"21.5C"` or `"212F"`.
It returns degrees Celsius: `C` keeps the number, `F` converts with `(value - 32) * 5 / 9`. Define the enum
`TempError` with three variants: `Empty`, `UnknownUnit(char)` and `BadNumber(ParseFloatError)`. Implement
`Display`, `Error` and `From<ParseFloatError>` so that `?` works. Write tests for: Celsius, Fahrenheit (`212F`
is `100.0`), empty input, unknown unit (`30K`) and a bad number (`abcC`). Expected output for the
five samples:

```text
"21.5C" -> 21.5 C
"212F" -> 100.0 C
"" -> error: the input is empty
"30K" -> error: unknown unit 'K' (use C or F)
"abcC" -> error: bad number: invalid float literal
```

<details>
<summary>Hint</summary>

Trim the text. `text.chars().last()` gives the unit as an `Option<char>`: use `let Some(unit) = ... else { return Err(TempError::Empty); };`.
The number part is `&text[..text.len() - unit.len_utf8()]`; parse it with `parse::<f64>()?` (that needs the `From` impl).
Then `match unit { 'C' => ..., 'F' => ..., other => Err(TempError::UnknownUnit(other)) }`. Derive `Debug` and `PartialEq` on the enum;
`ParseFloatError` supports both. In the format string use `{:.1}` to print one decimal place.

</details>

<details>
<summary>Answer</summary>

```rust
use std::fmt;
use std::num::ParseFloatError;

#[derive(Debug, PartialEq)]
enum TempError {
    Empty,
    UnknownUnit(char),
    BadNumber(ParseFloatError),
}

impl fmt::Display for TempError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            TempError::Empty => write!(f, "the input is empty"),
            TempError::UnknownUnit(c) => write!(f, "unknown unit '{c}' (use C or F)"),
            TempError::BadNumber(e) => write!(f, "bad number: {e}"),
        }
    }
}

impl std::error::Error for TempError {}

impl From<ParseFloatError> for TempError {
    fn from(e: ParseFloatError) -> TempError {
        TempError::BadNumber(e)
    }
}

/// "21.5C" -> 21.5, "212F" -> 100.0 (always returns Celsius)
fn parse_temperature(text: &str) -> Result<f64, TempError> {
    let text = text.trim();
    let Some(unit) = text.chars().last() else {
        return Err(TempError::Empty);
    };
    let number_part = &text[..text.len() - unit.len_utf8()];
    let value: f64 = number_part.trim().parse()?;
    match unit {
        'C' => Ok(value),
        'F' => Ok((value - 32.0) * 5.0 / 9.0),
        other => Err(TempError::UnknownUnit(other)),
    }
}

fn main() {
    for text in ["21.5C", "212F", "", "30K", "abcC"] {
        match parse_temperature(text) {
            Ok(c) => println!("{:?} -> {:.1} C", text, c),
            Err(e) => println!("{:?} -> error: {}", text, e),
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn celsius() {
        assert_eq!(parse_temperature("21.5C"), Ok(21.5));
    }

    #[test]
    fn fahrenheit_boiling_point() {
        assert_eq!(parse_temperature("212F"), Ok(100.0));
    }

    #[test]
    fn empty_input() {
        assert_eq!(parse_temperature("  "), Err(TempError::Empty));
    }

    #[test]
    fn unknown_unit() {
        assert_eq!(parse_temperature("30K"), Err(TempError::UnknownUnit('K')));
    }

    #[test]
    fn bad_number() {
        assert!(matches!(parse_temperature("abcC"), Err(TempError::BadNumber(_))));
    }
}
```

Why it works: each failure has its own variant, so tests (and callers) can tell them apart. The `From` impl
lets `?` wrap the float parsing error. Slicing at `text.len() - unit.len_utf8()` is safe because we cut off
exactly one whole character from the end ([Lesson 04.8](08-strings-in-depth.md)). Comparing floats with `==` works here
because the results are exact (21.5 and 100.0 are exactly representable in this calculation); in general,
compare floats with a small tolerance.

</details>

## Quiz

1. When should you use `panic!` and when `Result`?

<details>
<summary>Answer</summary>

Use `panic!` for bugs and broken promises, situations the program cannot meaningfully continue from. Use
`Result` for failures that can happen even in correct code (bad input, missing file, network error), so the
caller can decide what to do.

</details>

2. What does `unwrap()` do on `Err(e)`? On `Ok(v)`?

<details>
<summary>Answer</summary>

On `Ok(v)` it returns `v`. On `Err(e)` it panics, printing the error value. `expect("msg")` does the same
but with your own message.

</details>

3. What does the `?` operator do?

<details>
<summary>Answer</summary>

It looks at a `Result`. On `Ok(v)` the expression becomes `v`. On `Err(e)` the current function returns
`Err(e)` immediately (converting the error with `From` if the types differ). It works only in functions
that return `Result` (or `Option`).

</details>

4. Why does this fail: `fn main() { let t = std::fs::read_to_string("a.txt")?; }`?

<details>
<summary>Answer</summary>

`main` returns `()`, and `?` needs the function to return a `Result` or `Option` so it has somewhere to send
the error. Change the signature to `fn main() -> Result<(), Box<dyn Error>>` and end with `Ok(())`.

</details>

5. What do you need to write so that `?` can convert a `ParseIntError` into your `AppError`?

<details>
<summary>Answer</summary>

An implementation of `From<ParseIntError> for AppError`, for example `impl From<ParseIntError> for AppError`
whose `from` function wraps the error in one of the enum's variants.

</details>

6. What is `Box<dyn Error>` good for, and what is its drawback?

<details>
<summary>Answer</summary>

It is a catch-all error type: any error can be put in it, and `?` converts to it automatically. That is
convenient for `main`, small tools and prototypes. The drawback is that the caller cannot easily `match` on the
exact kind of error, so libraries should use their own error enums.

</details>

7. Which stream should an error message use, and what exit code means success?

<details>
<summary>Answer</summary>

Errors go to stderr (`eprintln!`), results to stdout (`println!`). Exit code `0` means success; any other
number means failure.

</details>

8. How do you check in a test that a function returns an error?

<details>
<summary>Answer</summary>

`assert!(f(x).is_err())`, `assert_eq!(f(x), Err(expected))` (needs `PartialEq` and `Debug` on the error type),
or `assert!(matches!(f(x), Err(_)))`. Use `#[should_panic]` only for functions that are supposed to panic.

</details>

## Summary

- A bug stops the program with `panic!`; an expected failure is returned as a `Result<T, E>`.
- `unwrap` and `expect` panic on `Err`. Use them in tests and for provable cases, never on outside data.
- `match` handles every case. Helpers like `unwrap_or`, `ok`, `ok_or` and `map_err` cover small jobs.
- The `?` operator returns the error early (and converts it with `From`); it needs a function that returns
  `Result` or `Option`.
- `main` can return `Result<(), Box<dyn Error>>`. A returned `Err` prints `Error: …` and exits with code `1`.
- A good error type is an enum with `Debug`, `Display` and `Error`, plus `From` impls for the errors it wraps.
- Libraries use their own enums; small applications can use `Box<dyn Error>` (later: `thiserror` and `anyhow`).
- Send errors to stderr, use exit code `0` for success, never print secrets in messages, and test the failure
  cases as carefully as the success cases.

## Sources and further reading

- [The Rust Programming Language — 9. Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html) — the official book chapter (panic, `Result`, `?`, and when to use each).
- [std::result::Result](https://doc.rust-lang.org/std/result/index.html) — all methods of `Result` with examples.
- [std::error::Error](https://doc.rust-lang.org/std/error/trait.Error.html) — the `Error` trait.
- [Rust by Example — Error handling](https://doc.rust-lang.org/rust-by-example/error.html) — many short examples, including custom errors and `Box<dyn Error>`.
- [std::process::exit](https://doc.rust-lang.org/std/process/fn.exit.html) — exit codes and what `exit` does not clean up.
- [The Rust API Guidelines — Error types](https://rust-lang.github.io/api-guidelines/interoperability.html) — what a good error type looks like for libraries.

---

[🏠 Start](../START-HERE.md) · [← 04.9 Hash maps](09-hash-maps.md) · [📋 Phase 04](README.md) · [04.11 Project: phone book →](11-project-phone-book.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
