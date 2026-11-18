<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 02.6 Printing, formatting and comments](06-printing-formatting-and-comments.md) · [📋 Phase 02](README.md) · [02.8 Your first tests →](08-your-first-tests.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/02-programming-basics/07-reading-input-and-parsing.md) <!-- nav -->

# Lesson 02.7 — Reading input and parsing text

> **Phase 02 — Programming basics in Rust** · Lesson 7 of 11 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How a program reads a line that the user types (standard input).
- Why the text you read ends with a new line, and how `trim` removes it.
- How to turn text into a number with `parse`, and what a `Result` is.
- How to handle wrong input without crashing, and how to ask again until the input is valid.
- How to make a prompt appear before the user types, and how to test input code from the terminal.

## Before you start

- You finished [Lesson 02.6 — Printing, formatting and comments](06-printing-formatting-and-comments.md).
- You remember `loop`, `break` and `match` from [Lesson 02.5](05-control-flow.md).
- You remember the terminal and the idea of redirecting streams from [Lesson 02.6](06-printing-formatting-and-comments.md).

## New words

| Word | Meaning |
|---|---|
| **stdin** (standard input) | The normal input stream of a program. In the terminal, it is what you type. |
| **I/O** (Input/Output) | Reading data into a program and writing data out of it. |
| **EOF** (End Of File) | The signal that there is no more input. In the terminal, press `Ctrl+D` on an empty line to send it. |
| **buffer** | A piece of memory used to hold data on its way in or out. |
| **parse** | Read text and turn it into a value of a type, for example `"42"` into the number `42`. |
| **`Result`** | A type that is either `Ok(value)` (it worked) or `Err(error)` (it failed). Phase 04, Lesson 10 teaches it fully. |
| **`expect`** | A method of `Result`: if it is `Ok`, give the value; if it is `Err`, panic with your message. |
| **trim** | Remove white space (spaces, tabs, new lines) from the start and the end of text. |
| **pipe** (`\|`) | A terminal symbol that sends the output of one command to the input of the next. |
| **reference** (`&`) | A way to let another piece of code use a value without giving the value away. Phase 03 teaches this. |
| **UTF-8** (Unicode Transformation Format, 8-bit) | The standard way to store Unicode text as bytes. Rust text is always valid UTF-8. |
| **flush** | Force a buffer to be sent out now, instead of waiting. |

## 1. How input arrives

A program talks to the world through **streams**. You know the output ones from Lesson 02.6: stdout and stderr. The input one is **stdin**: the keyboard when you run the program in a terminal, or the output of another command when you connect them.

```text
 keyboard --> terminal --> [ stdin ] --> your program --> [ stdout ] --> terminal screen
                                                      \-> [ stderr ] --> terminal screen
```

The program does not see single keys. The terminal collects what you type, and when you press **Enter**, it sends the whole line, including the new-line character `\n` at the end.

## 2. Reading a line

```rust
use std::io;

fn main() {
    let mut input = String::new();
    println!("What is your name?");
    io::stdin()
        .read_line(&mut input)
        .expect("failed to read a line");
    println!("raw:     {:?}", input);
    let name = input.trim();
    println!("trimmed: {:?}", name);
    println!("Hello, {name}!");
}
```

Run this with `echo "Aziza" | cargo run -q --bin 07_read_line` (the pipe `|` sends the text `Aziza` to stdin; section 8 explains it) or run `cargo run -q --bin 07_read_line` and type your name.

```text
What is your name?
raw:     "Aziza\n"
trimmed: "Aziza"
Hello, Aziza!
```

What happens:

1. `use std::io;` makes the module `io` of the standard library (`std`) available under the short name `io`. A *module* is a named group of items; Phase 04, Lesson 6 teaches them.
2. `String::new()` creates an empty `String`, the growable text type from Lesson 02.6. It is `mut` because `read_line` will write into it. The variable is the **buffer** that receives the text.
3. `io::stdin()` gives access to standard input. `.read_line(&mut input)` waits until the user presses Enter, then **appends** the line to `input`.
4. `&mut input` means: "lend `input` to `read_line` and allow it to change it". The `&` makes a **reference**: a way of giving access without giving away the value. Phase 03 explains references fully. For now, remember that `read_line` always needs `&mut` before the buffer.
5. `read_line` returns a `Result`, because reading can fail (for example, if the input is not valid UTF-8). The method `.expect("…")` says: "if it worked, continue; if it failed, panic with this message". We will handle errors more politely in section 5.
6. `{:?}` shows `"Aziza\n"`: the text has a hidden `\n` at the end. This is why we use Debug when debugging.
7. `input.trim()` returns the text without white space at both ends. Now `name` is `"Aziza"`.

Two details to remember:

- `read_line` **appends**. If you reuse the same `String` for the next line without emptying it, the new line is added after the old one:

```rust
use std::io;

fn main() {
    let mut input = String::new();
    io::stdin().read_line(&mut input).expect("read failed");
    io::stdin().read_line(&mut input).expect("read failed");
    println!("{:?}", input);
}
```

With the input `one` and `two` on separate lines, it prints `"one\ntwo\n"`. Create a fresh `String` for each line (as in the loop below), or call `input.clear()`.

- At the end of the input (**EOF**), `read_line` returns `Ok(0)` — zero bytes — and leaves the buffer empty. A loop that reads lines must check for this, or it will spin forever:

```rust
use std::io;

fn main() {
    let mut input = String::new();
    let bytes = io::stdin()
        .read_line(&mut input)
        .expect("failed to read a line");
    println!("bytes read: {bytes}");
    println!("text: {:?}", input);
}
```

```text
bytes read: 3
text: "hi\n"
```

That is the output for the input `hi`. With empty input (`cargo run -q --bin 07_eof < /dev/null`) the result is `bytes read: 0` and `text: ""`.

## 3. From text to a number: `parse`

Everything you read is text. To calculate, you must **parse** it into a number:

```rust
use std::io;

fn main() {
    let mut input = String::new();
    io::stdin()
        .read_line(&mut input)
        .expect("failed to read a line");
    let number: i32 = input.trim().parse().expect("please type a whole number");
    println!("{number} doubled is {}", number * 2);
}
```

```text
21 doubled is 42
```

(for the input `21`.) What happens:

1. `input.trim()` removes the new line. This is essential: `"21\n"` is not a valid number.
2. `.parse()` tries to convert the text into a number. **Which type?** The compiler learns it from `let number: i32`. This is why the type annotation is needed (remember the `E0284` error in Lesson 02.2). You can also write the type next to the method: `.parse::<i32>()`.
3. `parse` returns a `Result`, just like `read_line`. Its `.expect(…)` gives the number if the text was valid.
4. With the input `abc`, the program panics:

```text
thread 'main' (4056609) panicked at src/main.rs:8:44:
please type a whole number: ParseIntError { kind: InvalidDigit }
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

A panic is acceptable in a quick experiment, but a real program should not crash because of a typing mistake. To handle errors, you need to understand `Result`.

### What `parse` accepts and rejects

`parse::<T>()` works for numbers, `bool`, `char` and more. Look at what it accepts (`Ok`) and rejects (`Err`):

```rust
fn main() {
    println!("{:?}", "42".parse::<i32>());
    println!("{:?}", " 42 ".parse::<i32>());
    println!("{:?}", "42\n".parse::<i32>());
    println!("{:?}", "abc".parse::<i32>());
    println!("{:?}", "".parse::<i32>());
    println!("{:?}", "3.5".parse::<i32>());
    println!("{:?}", "3.5".parse::<f64>());
    println!("{:?}", "99999999999".parse::<i32>());
    println!("{:?}", "-5".parse::<u32>());
    println!("{:?}", "+7".parse::<i32>());
    println!("{:?}", "true".parse::<bool>());
    println!("{:?}", "x".parse::<char>());
}
```

```text
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
Err(ParseIntError { kind: InvalidDigit })
Err(ParseIntError { kind: InvalidDigit })
Err(ParseIntError { kind: Empty })
Err(ParseIntError { kind: InvalidDigit })
Ok(3.5)
Err(ParseIntError { kind: PosOverflow })
Err(ParseIntError { kind: InvalidDigit })
Ok(7)
Ok(true)
Ok('x')
```

Notes:

- Spaces and new lines around the number make it **invalid** (lines 2 and 3). Always `trim()` first.
- An empty text, letters, and a number with a fraction are not valid `i32` values. The text `"3.5"` is valid for `f64`.
- A number too big for the type gives `PosOverflow` ("positive overflow"). A negative number for an unsigned type is `InvalidDigit`, because `-` is not a digit for `u32`.
- A leading `+` is accepted.

## 4. Understanding `Result`

A `Result` is a value that is **either** a success **or** a failure:

```text
Result<i32, ParseIntError>
   |
   +-- Ok(42)                         the number was parsed
   +-- Err(ParseIntError { ... })     it failed, and here is the reason
```

The two parts in angle brackets are the types of the two cases: here, the success value is an `i32`, and the error value is a `ParseIntError`. (Types with `< >` hold other types; Phase 05, Lesson 1, explains generics.)

Rust forces you to deal with both cases. You cannot use the number until you have checked that it exists. Compare:

```rust
fn main() {
    let n: i32 = "42".parse();
    println!("{n}");
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:2:18
  |
2 |     let n: i32 = "42".parse();
  |            ---   ^^^^^^^^^^^^ expected `i32`, found `Result<_, _>`
  |            |
  |            expected due to this
  |
help: consider using `Result::expect` to unwrap the `Result<_, _>` value, panicking if the value is a `Result::Err`
```

The text is not a number yet; it is a `Result` that may *contain* a number. The way to get the value is to handle both cases. `expect` handles the failure by panicking. A polite way is `match`, which you know from Lesson 02.5:

```rust
fn main() {
    match "abc".parse::<i32>() {
        Ok(n) => println!("number {n}"),
        Err(e) => println!("error message: {e}"),
    }
    match "".parse::<i32>() {
        Ok(n) => println!("number {n}"),
        Err(e) => println!("error message: {e}"),
    }
    match "99999999999".parse::<i32>() {
        Ok(n) => println!("number {n}"),
        Err(e) => println!("error message: {e}"),
    }
}
```

```text
error message: invalid digit found in string
error message: cannot parse integer from empty string
error message: number too large to fit in target type
```

What happens: the pattern `Ok(n)` matches a success and names the value inside `n`. The pattern `Err(e)` matches a failure and names the error `e`. With `{e}` (Display) the error prints as a message that a human can read — much nicer than the Debug form `ParseIntError { kind: InvalidDigit }`.

## 5. Asking again until the input is valid

Now combine `loop`, `read_line`, `parse` and `match`:

```rust
use std::io;

fn main() {
    loop {
        println!("Type a whole number:");
        let mut input = String::new();
        io::stdin()
            .read_line(&mut input)
            .expect("failed to read a line");
        match input.trim().parse::<i32>() {
            Ok(n) => {
                println!("Thank you! {n} squared is {}", n * n);
                break;
            }
            Err(e) => {
                println!("'{}' is not a number ({e}). Try again.", input.trim());
            }
        }
    }
}
```

What happens:

1. The `loop` repeats until a valid number is read.
2. A **new** `String` is created in every iteration, so lines do not pile up.
3. If the text is a number, the `Ok` arm prints the square and `break` leaves the loop.
4. Otherwise the `Err` arm prints a message, and the loop starts again.

With the input lines `hello`, an empty line, `3.5`, `12`:

```text
Type a whole number:
'hello' is not a number (invalid digit found in string). Try again.
Type a whole number:
'' is not a number (cannot parse integer from empty string). Try again.
Type a whole number:
'3.5' is not a number (invalid digit found in string). Try again.
Type a whole number:
Thank you! 12 squared is 144
```

> ⚠️ **Common mistake:** If the input stream ends (EOF), `read_line` gives `Ok(0)` and an empty text; the loop above would print the "not a number" message forever. A robust loop checks the byte count: `if bytes == 0 { break; }`. Exercise 5 does this.

## 6. A prompt on the same line

You want the cursor to wait right after the question: `Your age: ▮`. For this you use `print!` (no new line). But Rust sends output to the terminal line by line, so the prompt might not appear before the program waits for input. You must **flush** the output:

```rust
use std::io::{self, Write};

fn main() {
    print!("Your age: ");
    io::stdout().flush().expect("failed to flush");
    let mut input = String::new();
    io::stdin()
        .read_line(&mut input)
        .expect("failed to read a line");
    println!("You typed: {}", input.trim());
}
```

```text
Your age: You typed: 30
```

(The output is for the input `30` from a pipe, so the typed text is not visible; in a real terminal, you see your typing after `Your age: `.)

What happens:

1. `use std::io::{self, Write};` imports two things: the module `io` itself (`self`) and `Write`, a **trait**. A trait is a set of abilities; the method `flush` belongs to `Write`, and a trait must be imported to use its methods. (Phase 05, Lesson 2, teaches traits.)
2. `io::stdout().flush()` sends everything waiting in the stdout buffer to the screen immediately. It returns a `Result`, so we `expect` it.

## 7. Several values on one line

If the user types `3 4 5`, split the line into words with `split_whitespace()` and parse each one:

```rust
use std::io;

fn main() {
    let mut line = String::new();
    io::stdin().read_line(&mut line).expect("read failed");
    let mut sum = 0;
    let mut count = 0;
    for word in line.split_whitespace() {
        match word.parse::<i32>() {
            Ok(n) => {
                sum += n;
                count += 1;
            }
            Err(_) => println!("skipping {word:?}"),
        }
    }
    println!("count = {count}, sum = {sum}");
}
```

With the input `3 4 x 5`:

```text
skipping "x"
count = 3, sum = 12
```

`split_whitespace()` gives the words one by one, ignoring any number of spaces and tabs, and `for` goes through them. The pattern `Err(_)` means "an error; I do not need its details" (`_` ignores a value).

## 8. Putting the logic in a function, and testing it

Input code is hard to test, because it waits for a human. The solution is to separate the **work** from the **reading**: read in `main`, and put the decision in a function that takes text and returns a `Result`. Then test the function with `assert_eq!`:

```rust
use std::num::ParseIntError;

fn parse_number(text: &str) -> Result<i32, ParseIntError> {
    text.trim().parse()
}

fn main() {
    assert_eq!(parse_number("42"), Ok(42));
    assert_eq!(parse_number("  -7\n"), Ok(-7));
    assert!(parse_number("abc").is_err());
    assert!(parse_number("").is_err());
    assert!(parse_number("2147483648").is_err());
    println!("parse_number: all checks passed");
}
```

```text
parse_number: all checks passed
```

What happens:

1. The parameter `text: &str` is a piece of text that the function only looks at (you learn why it is `&str` in Phase 03).
2. The return type `Result<i32, ParseIntError>` says in the signature that the function can fail. The error type lives in `std::num`.
3. The body is just `text.trim().parse()`; its type is decided by the return type.
4. `Ok(42)` builds a success value to compare with. `.is_err()` answers `true` for an `Err`.
5. `2147483648` is `i32::MAX + 1`: too big, so an error. An edge case!

> 🧪 **Testing:** Always test the **bad** inputs, not only the good ones: empty text, letters, spaces, a number that is too big, a negative number for an unsigned type. Attackers and tired users find them first.

### Feeding input to your program from the terminal

You do not have to type the input every time. The terminal can connect streams:

| Command | What it does |
|---|---|
| `echo "42" \| cargo run -q --bin x` | `echo` prints `42` and a new line; the pipe `\|` sends it to the program's stdin. |
| `printf '10\nabc\n5\n' \| cargo run -q --bin x` | `printf` lets you write several lines with `\n`. |
| `cargo run -q --bin x < input.txt` | `<` reads stdin from a file. |
| `cargo run -q --bin x < /dev/null` | An empty input, so you test EOF. |

These commands are how you will check the acceptance tests of the project in Lesson 02.10.

> 🔒 **Security:** **Never trust input.** Everything that comes from the outside — keyboard, files, network — may be wrong, huge or hostile. Check it before you use it: parse it, check the range (`age <= 150`), and decide what to do on failure. Do not use `expect` or `unwrap` on user input in a real program: a user can crash your program with one wrong key press. Also note that `read_line` reads a line of any length; a program that must resist attackers limits the amount it reads. You will see this in Phase 10.

## Common mistakes

**1. Forgetting `mut` on the buffer.**

```text
error[E0596]: cannot borrow `input` as mutable, as it is not declared as mutable
 --> src/main.rs:5:27
  |
5 |     io::stdin().read_line(&mut input).expect("read failed");
  |                           ^^^^^^^^^^ cannot borrow as mutable
  |
help: consider changing this to be mutable
  |
4 |     let mut input = String::new();
  |         +++
```

Write `let mut input = String::new();`.

**2. Forgetting `trim()`.** `"42\n".parse::<i32>()` is an error (`InvalidDigit`). The message "invalid digit found in string" for an input that *looks* fine is the usual sign.

**3. Ignoring the `Result`.** If you call `read_line` and drop its result, the compiler warns:

```text
warning: unused `Result` that must be used
 --> src/main.rs:5:5
  |
5 |     io::stdin().read_line(&mut input);
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = note: this `Result` may be an `Err` variant, which should be handled
```

Add `.expect("…")` or handle it with `match`.

**4. Reusing the buffer.** `read_line` appends. Create a new `String` or call `clear()`.

**5. No type for `parse`.** `E0284 type annotations needed`: add `let n: i32` or `parse::<i32>()`.

**6. Using `unwrap` or `expect` on user input.** Fine for a quick test; wrong for a real program.

> ⚡ **Performance:** `read_line` into a new `String` for every line allocates memory each time. For small programs it does not matter. For programs that read millions of lines, you reuse one buffer with `clear()`, or you use buffered readers (Phase 05).

## Try it yourself

1. Create `src/bin/07_read_line.rs` with the first program of section 2. Run it by typing your name, and by piping: `echo "Rust" | cargo run -q --bin 07_read_line`.
2. Create `src/bin/07_double.rs` that asks `Enter a number:` (with `print!` and `flush`), parses it as `f64`, and prints twice the value. Handle bad input with `match`: print `That is not a number.` and stop.
3. Test it from the terminal: `echo 2.5 | cargo run -q --bin 07_double`, `echo abc | cargo run -q --bin 07_double`, and `cargo run -q --bin 07_double < /dev/null`.
4. Change it into a loop that asks again until the input is valid (section 5). Stop if the input ends.
5. Add the function `parse_number` from section 8 to a file and write five `assert!` checks.
6. Run `cargo fmt`, `cargo clippy`. Commit:

```bash
cd ~/projects/rust
git add code/p02-basics
git commit -m "feat(p02): add input and parsing examples"
```

## Exercises

### Exercise 1 — Greeting (easy)

Ask `What is your name?`, read one line, and print `Hello, <name>!` without the new line inside the greeting. Run it with a pipe to check.

<details>
<summary>Hint</summary>

You need `let mut input = String::new();`, `io::stdin().read_line(&mut input)`, and `input.trim()`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::io;

fn main() {
    println!("What is your name?");
    let mut input = String::new();
    io::stdin()
        .read_line(&mut input)
        .expect("failed to read a line");
    println!("Hello, {}!", input.trim());
}
```

Test: `echo Aziza | cargo run -q --bin x` prints `What is your name?` and `Hello, Aziza!`. Without `trim()`, the greeting would be split over two lines, because `input` ends with `\n`.

</details>

### Exercise 2 — Add two numbers (easy)

Read two numbers from two lines and print `a + b = sum`. A helper function `read_number() -> i32` should do the reading and parsing (you may use `expect` for now).

<details>
<summary>Hint</summary>

The body of the helper is the three lines from section 3. Its last expression is the parsed number, without a semicolon. Call it twice in `main`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::io;

fn read_number() -> i32 {
    let mut input = String::new();
    io::stdin().read_line(&mut input).expect("failed to read a line");
    input.trim().parse().expect("please type a whole number")
}

fn main() {
    let a = read_number();
    let b = read_number();
    println!("{a} + {b} = {}", a + b);
}
```

Test: `printf '20\n22\n' | cargo run -q --bin x` prints `20 + 22 = 42`. The return type `i32` tells `parse` which type to produce.

</details>

### Exercise 3 — A validating function with tests (medium)

Write `fn parse_age(text: &str) -> Result<u8, String>`. It must trim the text, parse a `u8`, and accept only ages from 0 to 150. On a failure it returns `Err` with a clear message (a `String` made with `format!`). Test: `"29"` → `Ok(29)`, `" 0\n"` → `Ok(0)`, `"150"` → `Ok(150)`, `"151"` → an `Err`, `"abc"`, `"-3"` and `"300"` → an `Err`.

<details>
<summary>Hint</summary>

Use `match text.trim().parse::<u8>()`. In the `Ok(age)` arm use `if age <= 150 { Ok(age) } else { Err(format!(…)) }`. Note that the type `u8` already rejects `-3` and `300` for you.

</details>

<details>
<summary>Answer</summary>

```rust
fn parse_age(text: &str) -> Result<u8, String> {
    match text.trim().parse::<u8>() {
        Ok(age) => {
            if age <= 150 {
                Ok(age)
            } else {
                Err(format!("{age} is too big for an age"))
            }
        }
        Err(_) => Err(format!("'{}' is not a whole number from 0 to 255", text.trim())),
    }
}

fn main() {
    assert_eq!(parse_age("29"), Ok(29));
    assert_eq!(parse_age(" 0\n"), Ok(0));
    assert_eq!(parse_age("150"), Ok(150));
    assert_eq!(parse_age("151"), Err(String::from("151 is too big for an age")));
    assert!(parse_age("abc").is_err());
    assert!(parse_age("-3").is_err());
    assert!(parse_age("300").is_err());
    println!("{:?}", parse_age("151"));
    println!("{:?}", parse_age("abc"));
    println!("parse_age: all checks passed");
}
```

```text
Err("151 is too big for an age")
Err("'abc' is not a whole number from 0 to 255")
parse_age: all checks passed
```

Why it works: the type `u8` limits the parse to 0–255, and the `if` adds the real rule (at most 150). `Ok(age)` and `Err(…)` build the two kinds of `Result`.

</details>

### Exercise 4 — Average of a line (medium)

Read one line with several numbers (possibly with decimals), for example `10 20 x 30`. Skip the words that are not numbers (print a warning for each to **stderr**), and print the average of the rest. If there are no numbers at all, print `no numbers to average` instead of dividing by zero.

<details>
<summary>Hint</summary>

Use `split_whitespace()`, `parse::<f64>()` and two counters: `sum` and `count`. To divide, write `sum / count as f64`. Check `count == 0` first.

</details>

<details>
<summary>Answer</summary>

```rust
use std::io;

fn main() {
    let mut line = String::new();
    io::stdin().read_line(&mut line).expect("failed to read a line");
    let mut sum = 0.0;
    let mut count = 0;
    for word in line.split_whitespace() {
        match word.parse::<f64>() {
            Ok(x) => {
                sum += x;
                count += 1;
            }
            Err(_) => eprintln!("skipping {word:?}"),
        }
    }
    if count == 0 {
        println!("no numbers to average");
    } else {
        println!("average of {count} numbers = {}", sum / count as f64);
    }
}
```

With `echo "10 20 x 30"` the output is `skipping "x"` (on stderr) and `average of 3 numbers = 20`. With an empty line it prints `no numbers to average`.

Why it works: warnings go to stderr, so they do not mix with the result; and the check `count == 0` avoids a division by zero (for floats it would give `NaN`, which is a wrong answer without a crash).

</details>

### Exercise 5 — Sum until `done` (hard)

Write a program that reads lines until the line `done` or the end of the input. Every line that is a valid `i64` is added to the sum. Lines that are not valid are reported on stderr with the reason (`skipped "abc": invalid digit found in string`) and counted. At the end, print `accepted N, skipped M, sum = S`. The program must never panic because of the input, and must stop at EOF.

<details>
<summary>Hint</summary>

Use `loop`. Read into a **new** `String` each time. If `read_line` returns `0`, `break`. Trim the text, `break` if it equals `"done"`, then `match text.parse::<i64>()`.

</details>

<details>
<summary>Answer</summary>

```rust
use std::io;

fn main() {
    let mut sum = 0i64;
    let mut good = 0;
    let mut bad = 0;
    loop {
        let mut line = String::new();
        let bytes = io::stdin().read_line(&mut line).expect("failed to read a line");
        if bytes == 0 {
            break;
        }
        let text = line.trim();
        if text == "done" {
            break;
        }
        match text.parse::<i64>() {
            Ok(n) => {
                sum += n;
                good += 1;
            }
            Err(e) => {
                eprintln!("skipped {text:?}: {e}");
                bad += 1;
            }
        }
    }
    println!("accepted {good}, skipped {bad}, sum = {sum}");
}
```

Test with `printf '10\nabc\n5\n\n-3\ndone\n99\n' | cargo run -q --bin x`:

```text
skipped "abc": invalid digit found in string
skipped "": cannot parse integer from empty string
accepted 3, skipped 2, sum = 12
```

Why it works: `10 + 5 - 3 = 12`; the line `99` after `done` is never read. With the input `1`, `2` and then EOF (no `done`), the loop stops at `bytes == 0` and prints `accepted 2, skipped 0, sum = 3`.

</details>

## Quiz

1. Why does the text from `read_line` end with `\n`?

<details>
<summary>Answer</summary>

The terminal sends the whole line including the new-line character that comes from pressing Enter. Use `trim()` to remove it.

</details>

2. Why must the buffer be `mut` and passed as `&mut input`?

<details>
<summary>Answer</summary>

`read_line` writes into the buffer, so the buffer must be changeable (`mut`), and `&mut` lends it to the method with permission to change it.

</details>

3. What does `"abc".parse::<i32>()` return?

<details>
<summary>Answer</summary>

A `Result`: `Err(ParseIntError { kind: InvalidDigit })`. A valid text would give `Ok(number)`.

</details>

4. What is the difference between `expect` and `match` for handling a `Result`?

<details>
<summary>Answer</summary>

`expect` gives the value on `Ok` and panics on `Err`. `match` lets you decide what to do in each case, for example print a message and ask again.

</details>

5. What does `read_line` return at the end of the input?

<details>
<summary>Answer</summary>

`Ok(0)` — zero bytes read — and the buffer stays empty.

</details>

6. Why do we call `flush()` after `print!`?

<details>
<summary>Answer</summary>

The output is buffered by line, so text without a new line may stay in the buffer. `flush()` sends it to the screen at once, so the prompt appears before the program waits for input.

</details>

7. How can you test a program that reads from stdin without typing?

<details>
<summary>Answer</summary>

Pipe the input with `echo "text" | cargo run`, or `printf` for many lines, or redirect a file with `< file.txt`. Better still, put the logic in a function that takes text, and test it with `assert_eq!`.

</details>

## Summary

- `std::io::stdin().read_line(&mut buffer)` reads one line **including** `\n` and appends it; the buffer must be `mut`.
- Use `trim()` before `parse()`. `parse::<T>()` (or a typed `let`) turns text into a value.
- `read_line` and `parse` return a `Result`: `Ok(value)` or `Err(error)`. `expect` panics on `Err`; `match` lets you handle it.
- A loop with `match` can ask again until the input is valid. Check for EOF (`Ok(0)`).
- Use `print!` and `io::stdout().flush()` for a prompt on the same line.
- Put decisions into functions that take text and return a `Result`; test them with good and bad inputs.
- Never trust input. Test programs from the terminal with `|` and `<`.

## Sources and further reading

- [The Rust Programming Language — 2. Programming a Guessing Game](https://doc.rust-lang.org/book/ch02-00-guessing-game-tutorial.html) — reading input, `parse` and `match` on a `Result`, in the form of a game (the project of this phase).
- [Standard library — std::io::Stdin](https://doc.rust-lang.org/std/io/struct.Stdin.html) — the documentation of `read_line` and the other ways to read.
- [Standard library — str::parse](https://doc.rust-lang.org/std/primitive.str.html#method.parse) — what `parse` does and which types it supports.
- [Standard library — std::result::Result](https://doc.rust-lang.org/std/result/enum.Result.html) — all methods of `Result` (`is_ok`, `is_err`, `expect`…).
- [Standard library — std::io](https://doc.rust-lang.org/std/io/index.html) — the whole module for input and output: readers, writers and the `Write` trait used for `flush`.

---

[🏠 Start](../START-HERE.md) · [← 02.6 Printing, formatting and comments](06-printing-formatting-and-comments.md) · [📋 Phase 02](README.md) · [02.8 Your first tests →](08-your-first-tests.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
