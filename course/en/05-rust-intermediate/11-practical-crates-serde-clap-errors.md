<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 05.10 Threads, channels and shared state](10-threads-channels-and-shared-state.md) · [📋 Phase 05](README.md) · [05.12 Benchmarking basics →](12-benchmarking-basics.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/05-rust-intermediate/11-practical-crates-serde-clap-errors.md) <!-- nav -->

# Lesson 05.11 — Practical crates: serde, clap, thiserror and anyhow

> **Phase 05 — Rust intermediate** · Lesson 11 of 14 · ⏱ about 4.5 hours (3 sittings)

## What you will learn

- How to turn Rust structs into JSON (and back) with `serde` and `serde_json`, and how to read TOML settings.
- How to build a command-line tool with arguments, options and subcommands using `clap`.
- How to define clean error types for a library with `thiserror`.
- How to report errors with context in an application with `anyhow`.
- How these four crates fit together in one small program, `taskbox`.

## Before you start

- You finished [Lesson 05.10 — Threads, channels and shared state](10-threads-channels-and-shared-state.md).
- You know how to add dependencies and features: [Lesson 05.8 — Cargo: dependencies and versions](08-cargo-dependencies-and-versions.md).
- You know error handling with `Result`, `?` and custom errors: [Lesson 04.10 — Error handling](../04-structs-enums-collections-errors/10-error-handling.md).
- You know structs and enums ([Lesson 04.1](../04-structs-enums-collections-errors/01-structs.md), [Lesson 04.3](../04-structs-enums-collections-errors/03-enums.md)) and traits ([Lesson 05.2](02-traits.md)).
- Your computer must be online: Cargo downloads the crates.

## New words

| Word | Meaning |
|---|---|
| **serialization** | Turning a value in memory (a struct) into text or bytes that you can store or send. |
| **deserialization** | The opposite: turning text or bytes back into a value. |
| **serde** (SERialization / DEserialization) | The standard Rust framework for serialization. It defines the traits `Serialize` and `Deserialize`. |
| **JSON** (JavaScript Object Notation) | A popular text format for data: `{"name": "Ana", "age": 30}`. |
| **TOML** (Tom's Obvious Minimal Language) | A text format for settings files. `Cargo.toml` uses it. |
| **YAML** (YAML Ain't Markup Language) | Another text format for settings, used by many tools. |
| **derive macro** | A `#[derive(...)]` line. The compiler runs code that writes the trait implementation for you. (You write your own macros in [Phase 06](../06-rust-advanced/08-procedural-macros.md).) |
| **attribute** | A `#[...]` line that gives the compiler or a macro extra information about the next item. |
| **CLI** (Command-Line Interface) | A program that you use by typing a command and arguments in the terminal. |
| **argument** | A word you type after the program name. A **positional argument** is identified by its place. |
| **option** | An argument with a name, like `--count 3` or `-c 3`. |
| **flag** | An option that has no value; it is either present or not, like `--shout`. |
| **subcommand** | A command inside a command, like `git commit` or `cargo build`. |
| **exit code** | A number that a program returns to the shell when it ends. `0` means success; anything else means failure. |
| **library error** | An error type of a crate that others use. It should be precise, so the caller can react to each case. |
| **application error** | An error at the top of a program. It is usually just shown to the user, with enough context to understand it. |
| **context** | A short message added to an error that says what the program was doing ("failed to load tasks from tasks.json"). |
| **error chain** | An error together with the errors that caused it ("failed to load…" because "the file is not valid JSON" because "expected ident at line 1"). |
| **EOF** (End Of File) | The point where there is no more input. |

## 1. Why these four crates?

Rust's standard library is small on purpose. For everyday jobs the community made crates that almost every project uses. You will meet these four in nearly every real Rust program:

| Crate | Job | Used mostly in |
|---|---|---|
| `serde` (+ `serde_json`, `toml`) | Convert structs to and from text formats. | libraries and applications |
| `clap` | Read command-line arguments, write `--help`. | applications |
| `thiserror` | Define error types with little code. | **libraries** (and the library part of an app) |
| `anyhow` | One catch-all error type with context. | **applications** (`main` and the code near it) |

A simple rule for errors: **a library tells the caller exactly what went wrong (`thiserror`); an application only reports it to a human (`anyhow`)**. The program `taskbox` in section 6 uses both.

Versions used in this lesson (checked 2026-11-17): `serde` 1.0, `serde_json` 1.0, `toml` 1.2, `clap` 4.6, `thiserror` 2.0, `anyhow` 1.0. You write only the short form: `"1"`, `"4"`, `"2"`.

## 2. serde: turning data into text and back

### Setup

Create a project and add the crates (`derive` is a **feature** of `serde`, see [Lesson 05.8](08-cargo-dependencies-and-versions.md)):

```bash
cd ~/projects/rust/code
cargo new p05-serde-lab
cd p05-serde-lab
cargo add serde@1 --features derive
cargo add serde_json@1
```

### Serialize and deserialize a struct

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Book {
    title: String,
    pages: u32,
    tags: Vec<String>,
}

fn main() {
    let book = Book {
        title: String::from("Rust in Action"),
        pages: 456,
        tags: vec![String::from("rust"), String::from("systems")],
    };

    let json = serde_json::to_string(&book).unwrap();
    println!("{json}");

    let pretty = serde_json::to_string_pretty(&book).unwrap();
    println!("{pretty}");

    let text = r#"{"title":"Zero To Production","pages":528,"tags":["web"]}"#;
    let parsed: Book = serde_json::from_str(text).unwrap();
    println!("{parsed:?}");
}
```

```text
{"title":"Rust in Action","pages":456,"tags":["rust","systems"]}
{
  "title": "Rust in Action",
  "pages": 456,
  "tags": [
    "rust",
    "systems"
  ]
}
Book { title: "Zero To Production", pages: 528, tags: ["web"] }
```

**What happens:**

1. `#[derive(Serialize, Deserialize)]` is a **derive macro**. At compile time it writes the code that knows how to turn a `Book` into data and back. You get the implementations of the traits `Serialize` and `Deserialize` for free.
2. `serde_json::to_string(&book)` produces compact JSON text. `to_string_pretty` adds line breaks and indentation for humans.
3. `r#"..."#` is a **raw string**: backslashes and quotes inside need no escaping.
4. `serde_json::from_str(text)` parses text into a value. The type comes from the annotation `let parsed: Book`. `from_str` returns a `Result`, because the text may be wrong (we use `unwrap` only for the short examples).

serde itself is only the framework (the traits and the derive). Each format lives in its own crate: `serde_json` for JSON, `toml` for TOML, and others for formats like MessagePack. Your struct, with one derive, works with all of them.

### Errors you will see

Data from outside is never trustworthy ([Lesson 02.7](../02-programming-basics/07-reading-input-and-parsing.md): never trust input). `from_str` tells you exactly what is wrong:

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Book {
    title: String,
    pages: u32,
}

fn main() {
    let missing = r#"{"title":"No pages"}"#;
    let wrong_type = r#"{"title":"Bad","pages":"many"}"#;
    let broken = r#"{"title":"Broken","pages":12"#;
    let negative = r#"{"title":"Neg","pages":-5}"#;

    for text in [missing, wrong_type, broken, negative] {
        match serde_json::from_str::<Book>(text) {
            Ok(book) => println!("ok: {} has {} pages", book.title, book.pages),
            Err(e) => println!("error: {e}"),
        }
    }
}
```

```text
error: missing field `pages` at line 1 column 20
error: invalid type: string "many", expected u32 at line 1 column 29
error: EOF while parsing an object at line 1 column 28
error: invalid value: integer `-5`, expected u32 at line 1 column 25
```

**What happens:** all four inputs are rejected with a clear message and a position (line and column). The Rust types do the validation: `pages: u32` means the value must be a whole number that fits in `u32`, so `"many"` and `-5` are errors. A missing field is an error unless you say otherwise (next section).

> 🔒 **Security:** Deserializing into precise types is a strong defence. Your program never sees a string where it expects a number. But a type does not check *meaning*: a `u32` of `4000000000` is valid for `pages`. Validate ranges and business rules yourself, as you will see in Phase 10.

### Attributes: adjusting the format

Real data rarely matches your Rust names. Use `#[serde(...)]` attributes:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Settings {
    #[serde(rename = "userName")]
    user_name: String,
    #[serde(default = "default_port")]
    port: u16,
    #[serde(default)]
    verbose: bool,
    #[serde(skip_serializing_if = "Option::is_none")]
    nickname: Option<String>,
}

fn default_port() -> u16 {
    8080
}

#[derive(Debug, Serialize, Deserialize, PartialEq)]
#[serde(rename_all = "lowercase")]
enum Level {
    Low,
    High,
}

fn main() {
    let text = r#"{"userName":"ana"}"#;
    let s: Settings = serde_json::from_str(text).unwrap();
    println!("{s:?}");
    println!("{}", serde_json::to_string(&s).unwrap());

    let with_nick = Settings { nickname: Some(String::from("A")), ..s };
    println!("{}", serde_json::to_string(&with_nick).unwrap());

    println!("{}", serde_json::to_string(&Level::High).unwrap());
    let l: Level = serde_json::from_str("\"low\"").unwrap();
    println!("{l:?}");
}
```

```text
Settings { user_name: "ana", port: 8080, verbose: false, nickname: None }
{"userName":"ana","port":8080,"verbose":false}
{"userName":"ana","port":8080,"verbose":false,"nickname":"A"}
"high"
Low
```

**What happens:**

1. `rename = "userName"` maps the JSON key `userName` to the Rust field `user_name` (Rust uses `snake_case`, many JSON files use `camelCase`).
2. `default = "default_port"` calls the function when the key is missing, so `port` becomes `8080`. A bare `#[serde(default)]` uses the type's default value (`false` for `bool`).
3. `skip_serializing_if = "Option::is_none"` leaves the key out when the value is `None`. That is why the first output has no `nickname`, and the second has.
4. `rename_all = "lowercase"` on an enum writes the variants `Low` and `High` as `"low"` and `"high"`.
5. `..s` is struct update syntax: take the remaining fields from `s`.

### JSON without a struct

When you do not know the shape in advance, use `serde_json::Value` and the `json!` macro:

```rust
use serde_json::{json, Value};

fn main() {
    let v: Value = serde_json::from_str(r#"{"name":"ana","scores":[3,5,8]}"#).unwrap();
    println!("name = {}", v["name"]);
    println!("second score = {}", v["scores"][1]);
    println!("missing = {}", v["nope"]);

    let built = json!({ "ok": true, "count": 3 });
    println!("{built}");
}
```

```text
name = "ana"
second score = 5
missing = null
{"count":3,"ok":true}
```

`Value` is an enum of all JSON kinds (null, bool, number, string, array, object). Indexing a missing key gives `null`, not a panic. Keys of an object come out sorted (`count` before `ok`). Prefer typed structs when you can: they document the data and the compiler checks your code.

### TOML for settings

Add `cargo add toml@1`. The same derive works:

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Config {
    name: String,
    port: u16,
    features: Vec<String>,
}

fn main() {
    let text = r#"
name = "demo"
port = 8080
features = ["fast", "safe"]
"#;
    let config: Config = toml::from_str(text).unwrap();
    println!("{config:?}");
    println!("{}", toml::to_string(&config).unwrap());
}
```

```text
Config { name: "demo", port: 8080, features: ["fast", "safe"] }
name = "demo"
port = 8080
features = ["fast", "safe"]
```

> 📏 **Convention:** In the Rust world, settings files are usually TOML (it is what Cargo uses, and it allows comments), data exchange is JSON. Avoid the old crate `serde_yaml`: it is marked deprecated, and its author no longer maintains it.

> ⚠️ **Common mistake:** `#[derive(Serialize)]` without the `derive` feature:
>
> ```text
> error: cannot find derive macro `Serialize` in this scope
>   --> src/main.rs:3:10
>   |
> 3 | #[derive(Serialize)]
>   |          ^^^^^^^^^
>   |
> note: `Serialize` is a trait, not a derive macro
> ```
>
> The trait `Serialize` exists, but the derive macro is behind the feature `derive`. Run `cargo add serde --features derive`.

## 3. clap: command-line arguments

A program started from the terminal receives **arguments**. You could read them with `std::env::args()` and parse them yourself, but you would have to write the help text, the error messages and the checks. `clap` (Command Line Argument Parser) does all of it from a struct.

```bash
cd ~/projects/rust/code
cargo new greet
cd greet
cargo add clap@4 --features derive
```

```rust
use clap::Parser;

/// Greets people from the command line.
#[derive(Parser, Debug)]
#[command(version, about)]
struct Cli {
    /// Name of the person to greet
    name: String,

    /// How many times to greet
    #[arg(short, long, default_value_t = 1)]
    count: u32,

    /// Print in capital letters
    #[arg(short, long)]
    shout: bool,
}

fn main() {
    let cli = Cli::parse();
    for _ in 0..cli.count {
        let line = format!("Hello, {}!", cli.name);
        if cli.shout {
            println!("{}", line.to_uppercase());
        } else {
            println!("{line}");
        }
    }
}
```

**What happens:**

1. `#[derive(Parser)]` makes `Cli::parse()`, which reads the real command line, checks it, and returns a filled `Cli`. If something is wrong, it prints an error and ends the program with exit code 2.
2. The field `name` has no attribute, so it is a **required positional argument**.
3. `#[arg(short, long, default_value_t = 1)]` makes an **option** with a short name `-c` and a long name `--count`. The type `u32` is checked automatically. If the user gives nothing, the default `1` is used.
4. `shout: bool` with `#[arg(short, long)]` becomes a **flag**: `-s` or `--shout`.
5. The doc comments (`///`) become the help texts. `#[command(version, about)]` takes the version from `Cargo.toml` and the description from the doc comment of the struct.

Run it (`cargo build`, then use `./target/debug/greet`, or `cargo run -- <arguments>`; the `--` separates Cargo's own arguments from your program's):

```bash
./target/debug/greet --help
```

```text
Greets people from the command line

Usage: greet [OPTIONS] <NAME>

Arguments:
  <NAME>  Name of the person to greet

Options:
  -c, --count <COUNT>  How many times to greet [default: 1]
  -s, --shout          Print in capital letters
  -h, --help           Print help
  -V, --version        Print version
```

```bash
./target/debug/greet --version
./target/debug/greet Ana -c 2 -s
```

```text
greet 0.1.0
HELLO, ANA!
HELLO, ANA!
```

The wrong usage gives helpful errors for free:

```bash
./target/debug/greet
./target/debug/greet Ana --count many
./target/debug/greet Ana --color
```

```text
error: the following required arguments were not provided:
  <NAME>

Usage: greet <NAME>

For more information, try '--help'.
error: invalid value 'many' for '--count <COUNT>': invalid digit found in string

For more information, try '--help'.
error: unexpected argument '--color' found

  tip: to pass '--color' as a value, use '-- --color'

Usage: greet <NAME>

For more information, try '--help'.
```

Each of these runs ended with exit code 2. (Check it with `echo $?` right after a command.)

### Subcommands

Bigger tools group actions: `git commit`, `cargo build`. In `clap`, a subcommand is an `enum` marked with `#[derive(Subcommand)]`. You will see it in full in section 6:

```rust
#[derive(Subcommand)]
enum Command {
    /// Add a new task
    Add {
        /// The text of the task
        title: String,
    },
    /// Show the tasks
    List {
        /// Show finished tasks too
        #[arg(short, long)]
        all: bool,
    },
}
```

Every variant is a subcommand; its fields are its arguments.

### Testing the argument parser

You do not need to start the program to test it. `try_parse_from` takes the arguments from a list, and returns a `Result` instead of ending the program:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_name_and_defaults() {
        let cli = Cli::try_parse_from(["greet", "Ana"]).unwrap();
        assert_eq!(cli.name, "Ana");
        assert_eq!(cli.count, 1);
        assert!(!cli.shout);
    }

    #[test]
    fn parses_options() {
        let cli = Cli::try_parse_from(["greet", "Ana", "-c", "3", "--shout"]).unwrap();
        assert_eq!(cli.count, 3);
        assert!(cli.shout);
    }

    #[test]
    fn missing_name_is_an_error() {
        assert!(Cli::try_parse_from(["greet"]).is_err());
    }
}
```

```text
test tests::parses_name_and_defaults ... ok
test tests::parses_options ... ok
test tests::missing_name_is_an_error ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

The first item of the list is the program name (`argv[0]`), like in a real command line. (In [Lesson 05.13](13-project-minigrep.md) you will also test a whole program from the outside.)

> 📏 **Convention:** Write the doc comment for every argument: it is both documentation in the code and the `--help` text. Follow the usual command-line habits: `-h`/`--help`, `-V`/`--version`, long options in `kebab-case`, errors to the error stream and a non-zero exit code on failure.

## 4. thiserror: error types for libraries

In [Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md) you wrote an error by hand: an `enum`, an `impl Display`, an `impl Error`, and `impl From<...>` for each source error. That is a lot of repeated code. `thiserror` writes it for you with a derive macro.

```bash
cargo add thiserror@2
```

```rust
use std::fmt;
use thiserror::Error;

// By hand (from Lesson 04.10):
#[derive(Debug)]
enum ByHand {
    TooShort(usize),
    BadChar(char),
}

impl fmt::Display for ByHand {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match self {
            ByHand::TooShort(n) => write!(f, "password too short: {n} characters"),
            ByHand::BadChar(c) => write!(f, "bad character {c:?}"),
        }
    }
}

impl std::error::Error for ByHand {}

// With thiserror: the same, in half the lines.
#[derive(Debug, Error)]
enum PasswordError {
    #[error("password too short: {0} characters")]
    TooShort(usize),
    #[error("bad character {0:?}")]
    BadChar(char),
    #[error("could not read the number")]
    Number(#[from] std::num::ParseIntError),
}

fn check(pw: &str) -> Result<(), PasswordError> {
    if pw.len() < 8 {
        return Err(PasswordError::TooShort(pw.len()));
    }
    if let Some(c) = pw.chars().find(|c| c.is_whitespace()) {
        return Err(PasswordError::BadChar(c));
    }
    Ok(())
}

fn parse_len(text: &str) -> Result<usize, PasswordError> {
    let n: usize = text.parse()?;
    Ok(n)
}

fn main() {
    println!("{}", ByHand::TooShort(3));
    println!("{}", ByHand::BadChar(' '));
    println!("{:?}", check("abc"));
    println!("{}", check("abc").unwrap_err());
    println!("{}", check("abcd efgh").unwrap_err());
    println!("{:?}", check("abcdefgh"));
    let e = parse_len("x1").unwrap_err();
    println!("{e}");
    println!("source: {}", std::error::Error::source(&e).unwrap());
}
```

```text
password too short: 3 characters
bad character ' '
Err(TooShort(3))
password too short: 3 characters
bad character ' '
Ok(())
could not read the number
source: invalid digit found in string
```

**What happens:**

1. `#[derive(Error)]` implements `std::error::Error` for the enum. It needs `Debug` too (`#[derive(Debug, Error)]`).
2. `#[error("...")]` on each variant writes the `Display` message. `{0}` is the first field of a tuple variant; named fields are written `{name}`. `{0:?}` uses the `Debug` form.
3. `#[from]` on a field does two things: it implements `From<ParseIntError> for PasswordError` (so `?` converts the error automatically, as in `parse_len`) and it marks the field as the **source** of the error. That is why `Error::source(&e)` returns the original message.
4. The output of the hand-written `ByHand` and of the `thiserror` version is the same. The derive only removes the boilerplate.

If you forget `#[from]`, `?` cannot convert and the compiler says so:

```rust
#[derive(Debug, Error)]
enum MyError {
    #[error("bad number")]
    Number(std::num::ParseIntError),
}

fn parse(text: &str) -> Result<i32, MyError> {
    let n: i32 = text.parse()?;
    Ok(n)
}
```

```text
error[E0271]: type mismatch resolving `<i32 as FromStr>::Err == MyError`
  --> src/main.rs:10:23
   |
10 |     let n: i32 = text.parse()?;
   |                       ^^^^^ expected `MyError`, found `ParseIntError`
```

The fix: write `Number(#[from] std::num::ParseIntError)`.

> 📏 **Convention:** One error enum per module or crate, with variants named after *what went wrong* (`NotFound`, `Io`, `Json`), not after the crate that produced it. Start messages with a lowercase letter and do not end with a full stop: errors are often chained with `: ` between them.

## 5. anyhow: error handling in applications

In `main` you rarely need to match on error cases. You want to: stop, show a clear message with the reason, and return a failure exit code. `anyhow` gives one error type, `anyhow::Error`, which can hold any error, and the alias `anyhow::Result<T>`.

```bash
cargo add anyhow@1
```

```rust
use anyhow::{anyhow, bail, ensure, Context, Result};

fn parse_port(text: &str) -> Result<u16> {
    let port: u16 = text
        .trim()
        .parse()
        .with_context(|| format!("`{text}` is not a valid port number"))?;
    ensure!(port >= 1024, "port {port} is reserved for the system");
    Ok(port)
}

fn pick(name: &str) -> Result<u32> {
    match name {
        "small" => Ok(1),
        "big" => Ok(100),
        "" => bail!("the name is empty"),
        other => Err(anyhow!("unknown size `{other}`")),
    }
}

fn main() {
    for text in ["8080", "80", "http"] {
        match parse_port(text) {
            Ok(p) => println!("port = {p}"),
            Err(e) => {
                println!("short : {e}");
                println!("chain : {e:#}");
            }
        }
    }
    println!("{:?}", pick("small").unwrap());
    for name in ["", "huge"] {
        println!("{}", pick(name).unwrap_err());
    }

    let e = parse_port("http").unwrap_err();
    println!("debug view:\n{e:?}");
    println!(
        "is ParseIntError inside? {}",
        e.downcast_ref::<std::num::ParseIntError>().is_some()
    );
}
```

```text
port = 8080
short : port 80 is reserved for the system
chain : port 80 is reserved for the system
short : `http` is not a valid port number
chain : `http` is not a valid port number: invalid digit found in string
1
the name is empty
unknown size `huge`
debug view:
`http` is not a valid port number

Caused by:
    invalid digit found in string
is ParseIntError inside? true
```

**What happens:**

1. `Result<u16>` here is `anyhow::Result<u16>`, a short form of `Result<u16, anyhow::Error>`.
2. `.with_context(|| ...)` adds a message to the error if there is one. The closure runs only on failure, so no text is built on the success path. (`.context("fixed text")` does the same for a fixed message.) It works on any `Result` whose error implements `std::error::Error`, and on `Option`.
3. `?` converts the `ParseIntError` into an `anyhow::Error` automatically, keeping the original error inside.
4. `ensure!(condition, "message")` returns an error if the condition is false. `bail!("message")` returns an error at once. `anyhow!("message")` makes an error value. All three accept `format!`-style arguments.
5. `{e}` prints only the outermost message. `{e:#}` prints the whole **chain** on one line. `{e:?}` prints the message and a "Caused by:" list. When `main` returns `Err(e)`, Rust prints the `{e:?}` form.
6. `downcast_ref` lets you ask "was the original error a `ParseIntError`?". It is the escape hatch when you do need to react to one case.

### Library or application?

| | Library (`thiserror`) | Application (`anyhow`) |
|---|---|---|
| Who reads the error? | Other code. It matches on variants. | A human, in the terminal or a log. |
| Error type | A precise `enum` | `anyhow::Error` |
| Context | In the variants | `.context(...)` messages |
| Typical place | `src/lib.rs`, modules | `fn main()` and the code near it |

> ⚠️ **Common mistake:** Returning `anyhow::Error` from a library function. The caller can no longer see the cases without `downcast`. Libraries should return their own error type. Applications can use `anyhow` everywhere, because nobody downstream needs to match.

## 6. Putting it together: taskbox

`taskbox` is a small task list in the terminal. It stores tasks in a JSON file. It uses all four crates, and it has the usual shape of a Rust tool: a **library** with the logic and a thin **binary** (`main.rs`) with the command-line part.

```text
taskbox/
├── Cargo.toml
└── src/
    ├── lib.rs     ← the logic: Task, TaskError (thiserror), serde, tests
    └── main.rs    ← the command line: clap + anyhow
```

Create it:

```bash
cd ~/projects/rust/code
cargo new --lib taskbox
cd taskbox
cargo add serde@1 --features derive
cargo add serde_json@1 thiserror@2 anyhow@1
cargo add clap@4 --features derive
```

`cargo new --lib` makes `src/lib.rs`. A package can have both `src/lib.rs` and `src/main.rs`; `main.rs` can use the library by the package name (`taskbox`). `Cargo.toml` now has:

```toml
[dependencies]
anyhow = "1"
clap = { version = "4", features = ["derive"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
thiserror = "2"
```

`src/lib.rs`:

```rust
use std::fs;
use std::path::Path;

use serde::{Deserialize, Serialize};
use thiserror::Error;

#[derive(Debug, PartialEq, Serialize, Deserialize)]
pub struct Task {
    pub id: u32,
    pub title: String,
    pub done: bool,
}

#[derive(Debug, Error)]
pub enum TaskError {
    #[error("cannot read or write the task file")]
    Io(#[from] std::io::Error),
    #[error("the task file is not valid JSON")]
    Json(#[from] serde_json::Error),
    #[error("there is no task with id {0}")]
    NotFound(u32),
}

/// Reads all tasks from `path`. A missing file means "no tasks yet".
pub fn load(path: &Path) -> Result<Vec<Task>, TaskError> {
    if !path.exists() {
        return Ok(Vec::new());
    }
    let text = fs::read_to_string(path)?;
    Ok(serde_json::from_str(&text)?)
}

/// Writes all tasks to `path`.
pub fn save(path: &Path, tasks: &[Task]) -> Result<(), TaskError> {
    let text = serde_json::to_string_pretty(tasks)?;
    fs::write(path, text)?;
    Ok(())
}

/// Adds a task and returns its new id.
pub fn add(tasks: &mut Vec<Task>, title: &str) -> u32 {
    let id = tasks.iter().map(|t| t.id).max().unwrap_or(0) + 1;
    tasks.push(Task {
        id,
        title: title.to_string(),
        done: false,
    });
    id
}

/// Marks a task as done.
pub fn complete(tasks: &mut [Task], id: u32) -> Result<(), TaskError> {
    let task = tasks
        .iter_mut()
        .find(|t| t.id == id)
        .ok_or(TaskError::NotFound(id))?;
    task.done = true;
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn add_gives_increasing_ids() {
        let mut tasks = Vec::new();
        assert_eq!(add(&mut tasks, "first"), 1);
        assert_eq!(add(&mut tasks, "second"), 2);
        assert_eq!(tasks.len(), 2);
    }

    #[test]
    fn complete_marks_done() {
        let mut tasks = Vec::new();
        let id = add(&mut tasks, "write lesson");
        complete(&mut tasks, id).unwrap();
        assert!(tasks[0].done);
    }

    #[test]
    fn complete_unknown_id_is_an_error() {
        let mut tasks = Vec::new();
        let err = complete(&mut tasks, 7).unwrap_err();
        assert_eq!(err.to_string(), "there is no task with id 7");
    }

    #[test]
    fn json_round_trip() {
        let tasks = vec![Task { id: 1, title: String::from("a"), done: true }];
        let text = serde_json::to_string(&tasks).unwrap();
        let back: Vec<Task> = serde_json::from_str(&text).unwrap();
        assert_eq!(tasks, back);
    }
}
```

`src/main.rs`:

```rust
use std::path::PathBuf;

use anyhow::{Context, Result};
use clap::{Parser, Subcommand};
use taskbox::{add, complete, load, save};

/// A tiny task list that is stored in a JSON file.
#[derive(Parser)]
#[command(version, about)]
struct Cli {
    /// The file that stores the tasks
    #[arg(short, long, default_value = "tasks.json")]
    file: PathBuf,

    #[command(subcommand)]
    command: Command,
}

#[derive(Subcommand)]
enum Command {
    /// Add a new task
    Add {
        /// The text of the task
        title: String,
    },
    /// Show the tasks
    List {
        /// Show finished tasks too
        #[arg(short, long)]
        all: bool,
    },
    /// Mark a task as done
    Done {
        /// The id of the task
        id: u32,
    },
}

fn main() -> Result<()> {
    let cli = Cli::parse();
    let mut tasks = load(&cli.file)
        .with_context(|| format!("failed to load tasks from {}", cli.file.display()))?;

    match cli.command {
        Command::Add { title } => {
            let id = add(&mut tasks, &title);
            save(&cli.file, &tasks)?;
            println!("added task {id}");
        }
        Command::List { all } => {
            for task in tasks.iter().filter(|t| all || !t.done) {
                let mark = if task.done { "x" } else { " " };
                println!("[{mark}] {}: {}", task.id, task.title);
            }
        }
        Command::Done { id } => {
            complete(&mut tasks, id)?;
            save(&cli.file, &tasks)?;
            println!("task {id} is done");
        }
    }
    Ok(())
}
```

**What happens:**

1. `lib.rs` knows nothing about the terminal. `Task` uses `serde` to be saved; `TaskError` uses `thiserror`, with `#[from]` so that `?` turns an `io::Error` or a `serde_json::Error` into a `TaskError`.
2. `load` treats a missing file as an empty list and everything else as an error. `save` writes pretty JSON.
3. `add` and `complete` are plain functions on `Vec<Task>` or `[Task]`, so they are easy to test without files (the four tests in the file).
4. `main.rs` defines the command line with `clap`: a global option `--file` and three subcommands.
5. `fn main() -> Result<()>` is `anyhow::Result`. Any error that reaches the end of `main` is printed and the program exits with code 1. `with_context` adds the file name to errors from `load`.
6. `use taskbox::{...}`: the binary uses the library like any other crate.

Build it and try (the build is in `target/debug/taskbox`; the data file `tasks.json` appears in the current folder):

```bash
cargo build
mkdir -p run && cd run
../target/debug/taskbox --help
```

```text
A tiny task list that is stored in a JSON file

Usage: taskbox [OPTIONS] <COMMAND>

Commands:
  add   Add a new task
  list  Show the tasks
  done  Mark a task as done
  help  Print this message or the help of the given subcommand(s)

Options:
  -f, --file <FILE>  The file that stores the tasks [default: tasks.json]
  -h, --help         Print help
  -V, --version      Print version
```

```bash
../target/debug/taskbox add "write lesson"
../target/debug/taskbox add "run the code"
../target/debug/taskbox list
../target/debug/taskbox done 1
../target/debug/taskbox list
../target/debug/taskbox list --all
```

```text
added task 1
added task 2
[ ] 1: write lesson
[ ] 2: run the code
task 1 is done
[ ] 2: run the code
[x] 1: write lesson
[ ] 2: run the code
```

And the error paths:

```bash
../target/debug/taskbox done 99
echo "not json" > bad.json
../target/debug/taskbox -f bad.json list
```

```text
Error: there is no task with id 99
Error: failed to load tasks from bad.json

Caused by:
    0: the task file is not valid JSON
    1: expected ident at line 1 column 2
```

The second message is a full **error chain**: the context added in `main` ("failed to load…"), then the `TaskError` message from `thiserror` ("the task file is not valid JSON"), then the original message from `serde_json`. Each layer added what it knew. Each failing command has exit code 1.

The file `tasks.json` looks like this (this is `serde_json::to_string_pretty` at work):

```text
[
  {
    "id": 1,
    "title": "write lesson",
    "done": true
  },
  {
    "id": 2,
    "title": "run the code",
    "done": false
  }
]
```

Run the tests with `cargo test`:

```text
test tests::add_gives_increasing_ids ... ok
test tests::complete_marks_done ... ok
test tests::complete_unknown_id_is_an_error ... ok
test tests::json_round_trip ... ok
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Testing:** Keep the logic in the library and the thin layer (terminal, files, `main`) outside. You can test the logic with ordinary unit tests. In [Lesson 05.13](13-project-minigrep.md) you will see how to test the whole program with files.

### Git: save your work

This is a good moment to practice the habits from [Phase 01](../01-git-and-version-control/03-your-first-repository-and-commits.md) and [Lesson 05.8](08-cargo-dependencies-and-versions.md). `taskbox` lives inside your learning repository (`~/projects/rust`), so work on a branch and commit only this folder:

```bash
git switch -c project/05-taskbox
printf 'run/\ntasks.json\n' >> .gitignore
git add .
git status --short
git commit -m "feat(p05): add taskbox with serde, clap, thiserror and anyhow"
```

Why the extra lines in `.gitignore`? `tasks.json` and the `run` folder are *data* made while you try the program, not source code. Keep them out of Git (this `.gitignore` is inside the `taskbox` folder, and it works for that folder only). `Cargo.lock` stays in, because this package has a binary. `git add .` stages only the current folder, not the whole repository, and `git status --short` shows you what was staged before you commit.

## Common mistakes

1. **No `derive` feature for `serde` or `clap`** – "cannot find derive macro" (section 2). `cargo add serde --features derive`, `cargo add clap --features derive`.
2. **Forgetting `#[from]` in a `thiserror` enum** – `E0271` or `E0277` on `?` (section 4).
3. **Returning `anyhow::Error` from a library** – callers cannot match on cases. Use `thiserror` in libraries.
4. **Using `unwrap()` on user data** in a real tool. Return errors with `?` and show them; `unwrap` is for tests and quick examples.
5. **Using `cargo run -- args`** without the `--`. Cargo would try to read `--shout` as its own option. Put `--` before your program's arguments.
6. **Committing your data files** (`tasks.json`) to Git. Add them to `.gitignore`.
7. **Trusting parsed values** just because the types fit. A `u32` accepts `4000000000`. Check ranges.
8. **Using a deprecated crate** like `serde_yaml` in new code. Check `cargo info` and the crate page for "deprecated".

## Try it yourself

1. Create `taskbox` exactly as in section 6 and run the commands. Compare your output with the lesson.
2. Run `../target/debug/taskbox add` (without a title). Read the error. Run `../target/debug/taskbox done abc`. What does `clap` say?
3. Run `../target/debug/taskbox --version`. Where does the version come from? (Look at `Cargo.toml`.) Change it to `0.2.0`, rebuild, and run it again.
4. Open `tasks.json`, break it by deleting a bracket, and run `list`. Read the error chain.
5. Run `cargo tree --depth 1`. You have 5 direct dependencies. Run `cargo tree` without options to see how many crates you really depend on. This is your supply chain ([Lesson 05.8](08-cargo-dependencies-and-versions.md)).

## Exercises

### Exercise 1 — A round trip with a test (easy)

Define `struct Point { x: i32, y: i32 }` with `Debug`, `PartialEq`, `Serialize` and `Deserialize`. Write a **test** that serializes `Point { x: 3, y: -4 }`, checks that the JSON is exactly `{"x":3,"y":-4}`, parses it back and compares it with the original.

<details>
<summary>Hint</summary>

`serde_json::to_string(&p).unwrap()` and `serde_json::from_str::<Point>(&json).unwrap()`. Compare strings and values with `assert_eq!`.

</details>

<details>
<summary>Answer</summary>

```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, PartialEq, Serialize, Deserialize)]
pub struct Point {
    pub x: i32,
    pub y: i32,
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn point_round_trip() {
        let p = Point { x: 3, y: -4 };
        let json = serde_json::to_string(&p).unwrap();
        assert_eq!(json, r#"{"x":3,"y":-4}"#);
        let back: Point = serde_json::from_str(&json).unwrap();
        assert_eq!(back, p);
    }
}
```

Why it works: the derive writes fields in the order of the struct. `PartialEq` is needed for `assert_eq!` to compare two points. A round trip test is the first test to write for any serialized type.

</details>

### Exercise 2 — Add context with anyhow (easy)

Write `pub fn read_point(text: &str) -> anyhow::Result<Point>` that parses JSON into the `Point` of Exercise 1 and adds the context `could not read the point`. Write a **test** that checks: for the text `{` the error message is `could not read the point`, and the `{:#}` form starts with `could not read the point: `.

<details>
<summary>Hint</summary>

`serde_json::from_str(text).context("could not read the point")?` — `Context` must be imported from `anyhow`.

</details>

<details>
<summary>Answer</summary>

```rust
use anyhow::{Context, Result};

pub fn read_point(text: &str) -> Result<Point> {
    let point: Point = serde_json::from_str(text).context("could not read the point")?;
    Ok(point)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn read_point_adds_context() {
        let err = read_point("{").unwrap_err();
        assert_eq!(err.to_string(), "could not read the point");
        assert!(format!("{err:#}").starts_with("could not read the point: "));
        assert_eq!(read_point(r#"{"x":1,"y":2}"#).unwrap(), Point { x: 1, y: 2 });
    }
}
```

Why it works: `{err}` shows the newest (outermost) message only. `{err:#}` appends the causes after `: `. The `?` converts the context-wrapped error into `anyhow::Error`.

</details>

### Exercise 3 — Test a clap parser (medium)

Write `#[derive(Parser)] pub struct Args { word: String, times: usize }` where `times` is an option `--times`/`-t` with default `2`. Write `fn repeat(args: &Args) -> String` that joins `word` `times` times with spaces. Write a **test** with `try_parse_from`: `["repeat", "hi"]` gives `"hi hi"` and `["repeat", "hi", "--times", "3"]` gives `"hi hi hi"`.

<details>
<summary>Hint</summary>

`#[arg(short, long, default_value_t = 2)]` on `times`. Build the text with `vec![args.word.as_str(); args.times].join(" ")`.

</details>

<details>
<summary>Answer</summary>

```rust
use clap::Parser;

#[derive(Parser, Debug)]
pub struct Args {
    /// Word to repeat
    pub word: String,
    /// How many times
    #[arg(short, long, default_value_t = 2)]
    pub times: usize,
}

pub fn repeat(args: &Args) -> String {
    vec![args.word.as_str(); args.times].join(" ")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn repeat_uses_default_and_option() {
        let a = Args::try_parse_from(["repeat", "hi"]).unwrap();
        assert_eq!(repeat(&a), "hi hi");
        let b = Args::try_parse_from(["repeat", "hi", "--times", "3"]).unwrap();
        assert_eq!(repeat(&b), "hi hi hi");
    }
}
```

Why it works: `try_parse_from` runs the same parser as the real command line, so the test checks defaults and options without starting a process. `vec![x; n]` makes a vector of `n` copies, and `join(" ")` glues them.

</details>

### Exercise 4 — An error enum with thiserror (medium)

Write `pub fn parse_age(text: &str) -> Result<u8, AgeError>`. `AgeError` has two variants: `NotANumber` (message `age must be a number`) and `OutOfRange(u32)` (message `age {0} is not realistic`, for numbers over 150). Spaces around the text are allowed. Write a **test** for `" 42 "`, `"abc"` and `"200"`, including the message of the last error.

<details>
<summary>Hint</summary>

Derive `Debug`, `Error` and `PartialEq` on the enum. Use `text.trim().parse::<u32>()` with `.map_err(|_| AgeError::NotANumber)?`.

</details>

<details>
<summary>Answer</summary>

```rust
use thiserror::Error;

#[derive(Debug, Error, PartialEq)]
pub enum AgeError {
    #[error("age must be a number")]
    NotANumber,
    #[error("age {0} is not realistic")]
    OutOfRange(u32),
}

pub fn parse_age(text: &str) -> Result<u8, AgeError> {
    let n: u32 = text.trim().parse().map_err(|_| AgeError::NotANumber)?;
    if n > 150 {
        return Err(AgeError::OutOfRange(n));
    }
    Ok(n as u8)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn age_errors() {
        assert_eq!(parse_age(" 42 "), Ok(42));
        assert_eq!(parse_age("abc"), Err(AgeError::NotANumber));
        assert_eq!(parse_age("200"), Err(AgeError::OutOfRange(200)));
        assert_eq!(parse_age("200").unwrap_err().to_string(), "age 200 is not realistic");
    }
}
```

Why it works: parsing into `u32` first lets us give the *useful* error `OutOfRange` for `200` (and for `300`, which would not even fit in a `u8`) instead of a vague "not a number". `n as u8` is safe, because at that point `n <= 150`. The enum derives `PartialEq` only to make `assert_eq!` on a `Result` possible.

</details>

### Exercise 5 — A new subcommand for taskbox (hard)

Add a command `remove <id>` to `taskbox`. Write `pub fn remove(tasks: &mut Vec<Task>, id: u32) -> Result<Task, TaskError>` in `lib.rs` (it returns the removed task, or `TaskError::NotFound`). Add the subcommand in `main.rs`, which prints `removed task <id>: <title>`. Write a **test** for `remove`. Commit the change in Git with a good message (the style of this phase: `feat(p05): …`).

<details>
<summary>Hint</summary>

Find the position with `tasks.iter().position(|t| t.id == id)`, turn `None` into the error with `.ok_or(TaskError::NotFound(id))?`, then `tasks.remove(position)`. In `main.rs`, add a `Remove { id: u32 }` variant and a `match` arm that loads, removes, saves and prints.

</details>

<details>
<summary>Answer</summary>

In `src/lib.rs`:

```rust
/// Removes a task and returns it.
pub fn remove(tasks: &mut Vec<Task>, id: u32) -> Result<Task, TaskError> {
    let position = tasks
        .iter()
        .position(|t| t.id == id)
        .ok_or(TaskError::NotFound(id))?;
    Ok(tasks.remove(position))
}
```

In the `tests` module:

```rust
    #[test]
    fn remove_takes_the_task_out() {
        let mut tasks = Vec::new();
        let first = add(&mut tasks, "a");
        add(&mut tasks, "b");
        let removed = remove(&mut tasks, first).unwrap();
        assert_eq!(removed.title, "a");
        assert_eq!(tasks.len(), 1);
        assert!(matches!(remove(&mut tasks, 99), Err(TaskError::NotFound(99))));
    }
```

In `src/main.rs`: change the `use` line to `use taskbox::{add, complete, load, remove, save};`, add the variant to `enum Command`:

```rust
    /// Remove a task
    Remove {
        /// The id of the task
        id: u32,
    },
```

and the arm to the `match`:

```rust
        Command::Remove { id } => {
            let task = remove(&mut tasks, id)?;
            save(&cli.file, &tasks)?;
            println!("removed task {id}: {}", task.title);
        }
```

Running `taskbox add x`, `taskbox add y`, `taskbox remove 1`, `taskbox list` prints `added task 1`, `added task 2`, `removed task 1: x`, `[ ] 2: y`. `taskbox remove 5` prints `Error: there is no task with id 5` and exits with code 1. `cargo test` reports 5 passed.

Git: `git add .` then `git commit -m "feat(p05): add the remove command to taskbox"`.

Why it works: `position` returns `Option<usize>`, `ok_or` changes the `None` to the right error, and `?` returns it. The error type of the library flows through `anyhow` in `main` without any extra code. `matches!` checks a value against a pattern in a test without needing `PartialEq` on the error.

</details>

## Quiz

1. What is the difference between `serde` and `serde_json`?

<details>
<summary>Answer</summary>

`serde` is the framework: the traits `Serialize` and `Deserialize` and the derive macros. `serde_json` is one data format that uses them (JSON). Other formats (TOML, and so on) have their own crates.

</details>

2. What does `#[derive(Serialize, Deserialize)]` do?

<details>
<summary>Answer</summary>

It generates, at compile time, the code that converts the struct or enum to and from a data format. Without it, you would write those implementations by hand.

</details>

3. How does `clap` know which fields are required, which are options and which are flags?

<details>
<summary>Answer</summary>

By their form: a plain field is a required positional argument; `Option<T>` or a field with a default is optional; `#[arg(short, long)]` makes it a named option; a `bool` with `#[arg(...)]` is a flag. The doc comments become the help text.

</details>

4. When do you use `thiserror` and when `anyhow`?

<details>
<summary>Answer</summary>

`thiserror` in libraries, to define precise error types that callers can match on. `anyhow` in applications (especially `main`), to collect any error, add context and print it.

</details>

5. What does `#[from]` do in a `thiserror` enum?

<details>
<summary>Answer</summary>

It implements `From<SourceError>` for the enum, so `?` converts the source error automatically, and it makes the source error available through `Error::source`.

</details>

6. What is the difference between `{e}`, `{e:#}` and `{e:?}` for an `anyhow::Error`?

<details>
<summary>Answer</summary>

`{e}` is the outermost message only. `{e:#}` is the whole chain on one line, separated by `: `. `{e:?}` is the message plus a "Caused by:" list (the form that `main` prints when it returns `Err`).

</details>

7. How do you test a `clap` parser without running the program?

<details>
<summary>Answer</summary>

With `Cli::try_parse_from([...])`, passing the program name and arguments as a list. It returns a `Result` instead of exiting.

</details>

8. Why does `taskbox` keep its logic in `lib.rs` and not in `main.rs`?

<details>
<summary>Answer</summary>

The library has no terminal code, so its functions can be tested with ordinary unit tests and reused. `main.rs` is thin: it reads the command line, calls the library and prints.

</details>

## Summary

- `serde` converts data to and from text formats; `#[derive(Serialize, Deserialize)]` plus `serde_json` or `toml` handles most needs. Attributes like `rename`, `default` and `skip_serializing_if` adjust the format.
- Deserializing into typed structs validates the shape of external data; you still check the meaning yourself.
- `clap` builds a command-line interface from a struct: arguments, options, flags, subcommands, `--help` and errors for free. Test it with `try_parse_from`.
- `thiserror` removes the boilerplate of error enums: `#[error("...")]` for messages and `#[from]` for conversions.
- `anyhow` is for applications: `anyhow::Result`, `.context(...)`, `bail!`, `ensure!`, and a printed error chain.
- A good structure: logic in `lib.rs` (testable, precise errors), a thin `main.rs` (clap + anyhow).
- Keep data files out of Git; keep `Cargo.lock` in for binaries.

## Sources and further reading

- [Serde documentation](https://serde.rs/) — the guide: derive, attributes, data formats, custom implementations.
- [serde_json on docs.rs](https://docs.rs/serde_json/) — `from_str`, `to_string`, `Value`, `json!`.
- [clap on docs.rs](https://docs.rs/clap/) — the derive tutorial and reference.
- [thiserror on docs.rs](https://docs.rs/thiserror/) — all `#[error]`, `#[from]` and `#[source]` forms.
- [anyhow on docs.rs](https://docs.rs/anyhow/) — `Context`, `bail!`, `ensure!`, downcasting.
- [The Rust Programming Language — Chapter 9: Error Handling](https://doc.rust-lang.org/book/ch09-00-error-handling.html) — the foundation under `thiserror` and `anyhow`.
- [Command Line Applications in Rust](https://rust-cli.github.io/book/) — the official CLI working group's book: arguments, output, exit codes, testing.

---

[🏠 Start](../START-HERE.md) · [← 05.10 Threads, channels and shared state](10-threads-channels-and-shared-state.md) · [📋 Phase 05](README.md) · [05.12 Benchmarking basics →](12-benchmarking-basics.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
