<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 02.5 Control flow: conditions and loops](05-control-flow.md) · [📋 Phase 02](README.md) · [02.7 Reading input and parsing text →](07-reading-input-and-parsing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/02-programming-basics/06-printing-formatting-and-comments.md) <!-- nav -->

# Lesson 02.6 — Printing, formatting and comments

> **Phase 02 — Programming basics in Rust** · Lesson 6 of 11 · ⏱ about 3 hours (2 sittings)

## What you will learn

- The four printing macros and the difference between standard output and standard error.
- How `{}` (Display) and `{:?}` (Debug) differ, and when each is used.
- How to control width, alignment, padding, precision and number base in a format.
- How to build text with `format!`.
- How to write good comments, and what doc comments (`///`) are for.

## Before you start

- You finished [Lesson 02.5 — Control flow: conditions and loops](05-control-flow.md).
- You know the basic terminal tools from [Lesson 00.4 — The terminal: your first commands](../00-getting-started/04-terminal-basics.md).

## New words

| Word | Meaning |
|---|---|
| **stdout** (standard output) | The normal output stream of a program. `println!` writes here. |
| **stderr** (standard error) | A second output stream for errors and diagnostics. `eprintln!` writes here. |
| **stream** | A flow of bytes that a program writes to or reads from. |
| **format string** | The text in quotes that comes first in `println!`; it contains `{}` placeholders. |
| **placeholder** | The `{ }` in a format string; it is replaced by a value. |
| **format specifier** | The part after the colon in a placeholder, such as `:>8.2`. It says how to show the value. |
| **Display** | The "for people" way to show a value: what `{}` uses. |
| **Debug** | The "for programmers" way to show a value: what `{:?}` uses. |
| **width** | The minimum number of characters a value takes when printed. |
| **precision** | The number of digits after the decimal point for floats (`.2`). |
| **escape sequence** | A backslash followed by a letter that means a special character: `\n` (new line), `\t` (tab). |
| **raw string** | Text written as `r"…"`, where backslashes are not special. |
| **`String`** | A type for text that your program owns and can grow. `format!` returns it. Phase 04, Lesson 8 explains it in depth. |
| **comment** | Text for humans that the compiler ignores. |
| **doc comment** | A comment that starts with `///` (or `//!`). Tools turn it into documentation. |
| **trait** | A set of abilities that a type can have, for example "can be shown with `{}`". Phase 05 teaches traits. |

## 1. The printing macros

You know `println!`. There are four such macros:

| Macro | Writes to | New line at the end? |
|---|---|---|
| `println!` | stdout | yes |
| `print!` | stdout | no |
| `eprintln!` | stderr | yes |
| `eprint!` | stderr | no |

```rust
fn main() {
    println!("Hello");
    print!("one ");
    print!("two ");
    println!("three");
    println!();
    eprintln!("this goes to the error stream");
    eprint!("so does this ");
    eprintln!("one");
    println!("done");
}
```

```text
Hello
one two three

this goes to the error stream
so does this one
done
```

What happens:

1. `println!("Hello")` prints the text and moves to the next line.
2. The three `print!` and `println!` lines build one line, `one two three`.
3. `println!()` with nothing inside prints an empty line.
4. `eprintln!` and `eprint!` write to **stderr**, the "standard error" stream. In the terminal you see both streams mixed, so it looks the same.

Why two streams? Programs are often connected to each other or to files. Normal results go to **stdout**; warnings and errors go to **stderr**, so they do not get mixed into the results. You can see the difference by redirecting a stream in the terminal:

Save the program above as `src/bin/06_streams.rs` and run:

```bash
cargo run -q --bin 06_streams 2>/dev/null      # hide stderr
cargo run -q --bin 06_streams >/dev/null       # hide stdout
```

```text
Hello
one two three

done
```

```text
this goes to the error stream
so does this one
```

`2>/dev/null` sends stream number 2 (stderr) to the "black hole" device, and `>/dev/null` does the same for stream 1 (stdout). The first command shows only the results; the second shows only the errors. (`-q` makes Cargo itself quiet.)

> 📏 **Convention:** Results go to stdout (`println!`). Error messages, warnings and progress information go to stderr (`eprintln!`). Command-line tools that follow this rule can be used in pipelines and scripts.

> ⚠️ **Common mistake:** `print!` may not show its text at once. Rust sends stdout line by line, so text without a new line can wait in a buffer. If you print a prompt like `Enter a number: ` with `print!`, you must "flush" the buffer; Lesson 02.7 shows how.

## 2. Placeholders: `{}`

The first argument of `println!` is the **format string**. Every `{}` in it is a **placeholder** that is replaced by a value:

```rust
fn main() {
    let name = "Aziza";
    let age = 29;
    println!("{} is {} years old", name, age);
    println!("{0} is {1}; {1} is {1}; {0} again", name, age);
    println!("{who} is {years} years old", who = name, years = age);
    println!("{name} is {age} years old");
    println!("braces: {{ and }}");
    println!("2 + 3 = {}", 2 + 3);
    println!("tab:\there, newline:\nnext line");
    println!("quote: \"hi\", backslash: \\");
    println!(r"raw: C:\Users\new\table");
}
```

What happens:

1. Empty `{}` take the arguments in order: the first `{}` gets `name`, the second gets `age`.
2. `{0}` and `{1}` choose arguments **by position**, so one value can be used several times.
3. `{who}` takes a **named argument** given as `who = name`.
4. `{name}` takes a **variable with that name** from the surrounding code. This is the shortest form and the one you should prefer. It works only for plain variable names. For expressions like `2 + 3`, use `{}` and pass the expression as an argument.
5. To print a brace character, double it: `{{` and `}}`.
6. Inside the quotes, a backslash starts an **escape sequence**: `\t` is a tab, `\n` a new line, `\"` a quote, `\\` a backslash.
7. A **raw string** `r"…"` ignores escapes, which is handy for Windows paths and regular expressions.

```text
Aziza is 29 years old
Aziza is 29; 29 is 29; Aziza again
Aziza is 29 years old
Aziza is 29 years old
braces: { and }
2 + 3 = 5
tab:	here, newline:
next line
quote: "hi", backslash: \
raw: C:\Users\new\table
```

The compiler checks the format string at compile time, so mistakes never reach your users:

```rust
fn main() {
    println!("{} and {}", 1);
}
```

```text
error: 2 positional arguments in format string, but there is 1 argument
 --> src/main.rs:2:15
  |
2 |     println!("{} and {}", 1);
  |               ^^     ^^   -
```

An extra argument gives `error: argument never used`, and a name that does not exist gives `E0425 cannot find value`.

Text can run over several lines, and a backslash at the end of a line removes the new line and the spaces after it. A character can be written by its Unicode number:

```rust
fn main() {
    let text = "line one
line two";
    println!("{text}");
    let joined = "first part, \
                  second part";
    println!("{joined}");
    println!("crab: \u{1F980}");
}
```

```text
line one
line two
first part, second part
crab: 🦀
```

## 3. Display and Debug

`{}` uses the **Display** format: a nice form for people. `{:?}` uses the **Debug** format: a form for programmers, which shows the structure. Not every type can be shown with `{}`, but almost every type can be shown with `{:?}`.

```rust
fn main() {
    let pair = (1, "two");
    let numbers = [1, 2, 3];
    println!("{:?}", pair);
    println!("{:?}", numbers);
    println!("{:?} and {}", "text", "text");
    println!("{:?} and {}", 'x', 'x');
    println!("{:?} and {}", 7.0, 7.0);
    println!("{:#?}", pair);
}
```

```text
(1, "two")
[1, 2, 3]
"text" and text
'x' and x
7.0 and 7
(
    1,
    "two",
)
```

What happens:

1. Tuples and arrays have no Display format, because there is no single obvious way to show them. `{:?}` shows them in a form that looks like the code.
2. With `{:?}`, text keeps its quotes and a character keeps its single quotes. That is very helpful when you debug: you can see invisible things, such as an extra space at the end of a string (`"abc "`).
3. A float that is a whole number shows as `7.0` in Debug and `7` in Display.
4. `{:#?}` is the **pretty** Debug format: every item on its own line with indentation. It is the tool of choice for big structures.

What happens when you use `{}` on something that has no Display format?

```text
error[E0277]: `({integer}, {integer})` doesn't implement `std::fmt::Display`
 --> src/main.rs:3:20
  |
3 |     println!("{}", pair);
  |               --   ^^^^ `({integer}, {integer})` cannot be formatted with the default formatter
  |
  = help: the trait `std::fmt::Display` is not implemented for `({integer}, {integer})`
  = note: in format strings you may be able to use `{:?}` (or `{:#?}` for pretty-print) instead
```

The message says that the **trait** `Display` is missing for this type. A trait is a set of abilities; Phase 05 teaches them, and then you will give your own types the abilities `Display` and `Debug`. The compiler's note already shows the fix: use `{:?}`.

## 4. Width, alignment and padding

After the colon, a **format specifier** tells how to show the value. The general form is:

```text
{ : [fill][alignment][sign][#][0][width][.precision][type] }
```

You do not need to know all of it. These pieces are enough for most work:

| Piece | Meaning | Example |
|---|---|---|
| `width` | minimum number of characters | `{:5}` |
| `<` `>` `^` | align left, right, centre | `{:<5}` `{:>5}` `{:^5}` |
| a fill character before the alignment | what to fill with | `{:*^9}` |
| `0` | fill numbers with zeros | `{:05}` |
| `.N` | digits after the point | `{:.2}` |
| `+` | always show the sign | `{:+}` |

```rust
fn main() {
    println!("[{:5}]", 42);
    println!("[{:5}]", "ab");
    println!("[{:<5}]", 42);
    println!("[{:>5}]", "ab");
    println!("[{:^5}]", "ab");
    println!("[{:*^9}]", "mid");
    println!("[{:->8}]", 123);
    println!("[{:05}]", 42);
    let width = 8;
    println!("[{:>width$}]", "dyn");
    println!("[{:>1$}]", "dyn", 6);
}
```

```text
[   42]
[ab   ]
[42   ]
[   ab]
[ ab  ]
[***mid***]
[-----123]
[00042]
[     dyn]
[   dyn]
```

What happens:

1. The width `5` makes the field 5 characters wide. By default, **numbers are right-aligned** and **text is left-aligned**.
2. `<`, `>` and `^` set the alignment explicitly. With `^` (centre) and an odd number of spare spaces, the extra space goes to the right.
3. A character before the alignment is the **fill**: `*^9` means "centre in 9 characters, filled with `*`".
4. `05` pads a number with zeros.
5. `{:>width$}` takes the width from a variable named `width`. `{:>1$}` takes the width from the argument with index 1.

Alignment is what makes tables look like tables:

```rust
fn main() {
    println!("{:>3} | {:>4} | {:>5}", "n", "n^2", "n^3");
    println!("{:-<3}-+-{:-<4}-+-{:-<5}", "", "", "");
    for n in 1..=5 {
        println!("{:>3} | {:>4} | {:>5}", n, n * n, n * n * n);
    }
}
```

```text
  n |  n^2 |   n^3
----+------+------
  1 |    1 |     1
  2 |    4 |     8
  3 |    9 |    27
  4 |   16 |    64
  5 |   25 |   125
```

## 5. Precision and number bases

For floats, `.N` sets how many digits are shown after the decimal point. It **rounds** the printed value; the number itself does not change.

```rust
fn main() {
    let pi = 3.14159265;
    println!("{}", pi);
    println!("{:.2}", pi);
    println!("{:.0}", pi);
    println!("[{:8.3}]", pi);
    println!("[{:08.3}]", pi);
    println!("[{:<8.1}]", pi);
    println!("{:+}", 5);
    println!("{:+.1}", 2.0);
    println!("{:e}", 1234.5);
    let digits = 4;
    println!("{:.digits$}", pi);
    println!("{:.1}", 0.25);
    println!("{:.1}", 0.35);
    println!("{:.1}", 2.5);
}
```

```text
3.14159265
3.14
3
[   3.142]
[0003.142]
[3.1     ]
+5
+2.0
1.2345e3
3.1416
0.2
0.3
2.5
```

What happens:

1. `{:.2}` shows two decimals; `{:.0}` shows none.
2. `{:8.3}` combines width 8 and precision 3. `{:08.3}` adds zero padding.
3. `{:+}` shows a plus sign for positive numbers. `{:e}` shows scientific notation: `1.2345e3` is 1.2345 × 10³.
4. `{:.digits$}` takes the precision from a variable.
5. The last three lines show that rounding follows the *stored* binary value (Lesson 02.2): `0.35` is stored as a number a tiny bit below 0.35, so it shows `0.3`; and the exact tie `0.25` rounds to the even digit, `0.2`. Do not be surprised by this; it is another reason to keep money in whole cents.

For integers you can choose the **base**:

```rust
fn main() {
    let n = 255;
    println!("decimal:  {}", n);
    println!("binary:   {:b}", n);
    println!("octal:    {:o}", n);
    println!("hex:      {:x}", n);
    println!("HEX:      {:X}", n);
    println!("0x hex:   {:#x}", n);
    println!("0b bin:   {:#b}", n);
    println!("8 bits:   {:08b}", 5u8);
    println!("padded:   {:#010x}", 255);
}
```

```text
decimal:  255
binary:   11111111
octal:    377
hex:      ff
HEX:      FF
0x hex:   0xff
0b bin:   0b11111111
8 bits:   00000101
padded:   0x000000ff
```

The letters after the colon are: `b` binary, `o` octal, `x` lowercase hexadecimal, `X` uppercase hexadecimal. The `#` adds the prefix `0x` or `0b`. `{:08b}` — the format you saw in Lesson 02.3 — means "binary, at least 8 characters, filled with zeros".

## 6. Building text with `format!`

`format!` works exactly like `println!`, but instead of printing it **returns** the text as a `String`. A `String` is text that your program owns (Phase 04, Lesson 8 explains it in depth; for now, you can print it, compare it with a literal, and ask for its `.len()`).

```rust
fn main() {
    let label = format!("{:>5}", 42);
    println!("[{label}]");
    assert_eq!(label, "   42");
    assert_eq!(format!("{:.2}", 3.14159), "3.14");
    assert_eq!(format!("{}-{}", 1, 2), "1-2");
    assert_eq!(format!("{:03}", 7), "007");
    assert_eq!(format!("{:?}", (1, "a")), "(1, \"a\")");
    println!("format checks passed");
}
```

```text
[   42]
format checks passed
```

Because the result is a value and not printed output, `format!` is the way to **test** formatting code: compare the result with the expected text. A function that returns a `String` is easier to test than a function that prints.

> 🧪 **Testing:** Separate *making* the text from *printing* it. Write `fn format_money(cents: u32) -> String` and test it with `assert_eq!`; then print its result in `main`. Exercise 3 asks you to do that.

## 7. Comments

A **comment** is a note for humans. The compiler ignores it.

```rust
//! This file shows the kinds of comments.

/// Adds two numbers.
///
/// # Examples
/// ```
/// assert_eq!(add(2, 3), 5);
/// ```
fn add(a: i32, b: i32) -> i32 {
    a + b // inline comment: the sum
}

fn main() {
    // A normal line comment.
    /* A block comment
       over two lines. */
    let total = add(2, 3); // 5
    println!("{total}");
}
```

```text
5
```

The kinds:

| Kind | Looks like | Use |
|---|---|---|
| Line comment | `// text` | The normal comment. Runs to the end of the line. |
| Block comment | `/* text */` | Can span many lines. Rarely used; many teams prefer `//` on each line. |
| Doc comment for the next item | `/// text` | Documents the function, type or constant that follows. |
| Doc comment for the enclosing item | `//! text` | Documents the file or module itself; written at its start. |

**Doc comments** are special: tools read them. The command `cargo doc --open` builds a documentation website from your `///` comments, and the code examples inside them are run as tests by `cargo test`. Phase 05, Lesson 7 covers documentation in depth.

### How to comment well

- Comment the **why**, not the **what**. The code already says what it does. A comment like `// add 1 to counter` above `counter += 1;` is noise. A comment such as `// the API counts pages from 1, not from 0` is gold.
- Prefer clear names to comments. If you feel a need to explain a variable, rename it.
- Keep comments true. A comment that disagrees with the code is worse than none; update it when you change the code.
- Use `TODO:` for work you know is unfinished. Many editors list all `TODO`s.
- Do not leave old code commented out. Git remembers it ([Lesson 01.4](../01-git-and-version-control/04-viewing-history-and-changes.md)).

> 🔒 **Security:** Never put passwords, API (Application Programming Interface) keys or tokens in comments or in `println!` messages. Comments live forever in Git history, and logs are often shared. Section 1 gives one more reason to use `eprintln!` for diagnostics: they stay out of the normal output.

## Common mistakes

**1. `{}` on a tuple, array or your own type.** `E0277 … doesn't implement std::fmt::Display`. Use `{:?}`.

**2. The wrong number of arguments.** `println!("{} {}", 1)` is a compile error; the compiler counts for you.

**3. Using `{name}` for an expression.** `println!("{a + b}")` does not work. Write `println!("{}", a + b)` or first `let sum = a + b;`.

**4. Forgetting that `print!` has no new line.** Your prompt and the next output can end up on one line.

**5. Printing floats for money.** Use whole cents and print `{}.{:02}`.

**6. A comment that repeats the code.** Say why, not what.

> 📏 **Convention:** Prefer the inline form `{name}` when the variable is a plain name, and put the format specifier after the colon: `{price:>8.2}`.

> ⚡ **Performance:** Printing is slow compared with calculation, because it talks to the terminal. A loop that prints a million lines can take longer than all the other work. When you need speed, print less, or collect the text and print it once (Phase 05, Lesson 12 shows how to measure).

## Try it yourself

1. Create `src/bin/06_printing.rs`. Print your name, your age and your favourite number in one sentence, using the inline form `{name}`.
2. Print the number `2024` in decimal, binary, octal and hexadecimal (with prefixes).
3. Print a receipt with three items (names left-aligned in 10 characters, prices right-aligned in 8 characters with 2 decimals) and a total. Compare with Exercise 1.
4. Send one line to stderr with `eprintln!`, and run `cargo run -q --bin 06_printing 2>/dev/null` to check it disappears.
5. Add a doc comment `///` before one of your functions from Lesson 02.4. Run `cargo doc --bin 06_printing --open`: find your function in the browser page that opens.
6. Run `cargo fmt`, `cargo clippy`. Commit:

```bash
cd ~/projects/rust
git add code/p02-basics
git commit -m "docs(p02): add printing and formatting examples"
```

## Exercises

### Exercise 1 — A receipt (easy)

Print this receipt exactly (the item names are left-aligned in 10 characters, the prices are right-aligned in 8 characters with 2 decimals, and the total is calculated by the program, not typed):

```text
Item         Price
Tea           2.50
Cake          4.75
Water         1.00
------------------
Total         8.25
```

<details>
<summary>Hint</summary>

Use `{:<10}` for the name and `{:>8.2}` for the price. The separator is `{:-<18}` applied to the empty text `""`. The total is `2.5 + 4.75 + 1.0`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    println!("{:<10}{:>8}", "Item", "Price");
    println!("{:<10}{:>8.2}", "Tea", 2.5);
    println!("{:<10}{:>8.2}", "Cake", 4.75);
    println!("{:<10}{:>8.2}", "Water", 1.0);
    println!("{:-<18}", "");
    println!("{:<10}{:>8.2}", "Total", 2.5 + 4.75 + 1.0);
}
```

Why it works: `{:-<18}` means "fill with `-`, align left, width 18"; the text to align is empty, so the result is 18 dashes. The widths 10 + 8 add up to 18.

</details>

### Exercise 2 — Predict the brackets (easy)

Write down the output of each line, then run to check.

```rust
fn main() {
    println!("[{:>6}]", 42);
    println!("[{:<6}]", "ab");
    println!("[{:^7}]", "abc");
    println!("[{:06.2}]", 3.14159);
    println!("[{:x}]", 255);
    println!("[{:#b}]", 5);
    println!("[{:?}]", "hi");
    println!("[{:+}]", 7);
}
```

<details>
<summary>Hint</summary>

Count the characters between the brackets. `06.2` means: zero fill, width 6, two decimals.

</details>

<details>
<summary>Answer</summary>

```text
[    42]
[ab    ]
[  abc  ]
[003.14]
[ff]
[0b101]
["hi"]
[+7]
```

Why it works: `42` is right-aligned in 6; `ab` is left-aligned in 6; `abc` is centred in 7 (two spaces on each side); `3.14` has 4 characters, so two zeros are added to reach 6; `255` in hex is `ff`; `5` in binary is `101`, with the prefix `0b`; Debug shows the quotes; `+` shows the sign.

</details>

### Exercise 3 — Money without floats (medium)

Write `fn format_money(cents: u32) -> String` that turns a price in cents into text with two decimals: `1999` becomes `"19.99"`, `5` becomes `"0.05"`, `100` becomes `"1.00"`, `0` becomes `"0.00"`. Test it with `assert_eq!`, then print `format_money(123456)`.

<details>
<summary>Hint</summary>

The whole part is `cents / 100`; the remainder `cents % 100` must be printed with at least two digits and zero fill: `{:02}`. Use `format!`.

</details>

<details>
<summary>Answer</summary>

```rust
fn format_money(cents: u32) -> String {
    format!("{}.{:02}", cents / 100, cents % 100)
}

fn main() {
    assert_eq!(format_money(1999), "19.99");
    assert_eq!(format_money(5), "0.05");
    assert_eq!(format_money(100), "1.00");
    assert_eq!(format_money(0), "0.00");
    println!("{}", format_money(123456));
    println!("format_money: all checks passed");
}
```

```text
1234.56
format_money: all checks passed
```

Why it works: `5 / 100` is `0` and `5 % 100` is `5`, printed as `05`. No floats are used, so there are no rounding surprises. The text is compared with `assert_eq!` because the function returns it instead of printing it.

</details>

### Exercise 4 — A number base table (medium)

Print a table of the numbers 0 to 16 with three columns: the decimal number (width 3), the binary form with the `0b` prefix (width 10) and the hexadecimal form with the `0x` prefix (width 6). All columns are right-aligned. Print the headers `n`, `binary`, `hex`.

<details>
<summary>Hint</summary>

The width counts the prefix too. Put the `#` flag after the alignment: `{:>#10b}` means "right-aligned, with the `0b` prefix, binary, 10 characters wide". The hexadecimal version is `{:>#6x}`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    println!("{:>3} {:>10} {:>6}", "n", "binary", "hex");
    for n in 0..=16 {
        println!("{:>3} {:>#10b} {:>#6x}", n, n, n);
    }
}
```

The first and last lines of the output:

```text
  n     binary    hex
  0        0b0    0x0
  1        0b1    0x1
  2       0b10    0x2
  3       0b11    0x3
 ...
 15     0b1111    0xf
 16    0b10000   0x10
```

Why it works: in `{:>#10b}` the parts come in the order alignment, `#`, width, type. (You could also build the text with `format!("{:#b}", n)` first and align that text with `{:>10}`; the result is the same.) The real output has all 17 rows.

</details>

### Exercise 5 — A table that fits its numbers (hard)

Print a 5 × 5 multiplication table (rows 1..=5, columns 1..=5). All numbers must be right-aligned in columns whose width is **calculated** from the biggest number (25), not typed as a constant. Hint: the width is the length of the text of the biggest number.

<details>
<summary>Hint</summary>

`format!("{}", max).len()` gives the number of characters of `max`. Then use `{:>width$}`. Print each cell with `print!` and end each row with `println!()`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let max = 5 * 5;
    let width = format!("{}", max).len();
    for row in 1..=5 {
        for col in 1..=5 {
            print!("{:>width$} ", row * col);
        }
        println!();
    }
}
```

```text
 1  2  3  4  5 
 2  4  6  8 10 
 3  6  9 12 15 
 4  8 12 16 20 
 5 10 15 20 25 
```

Why it works: `width` is 2, because `25` has two characters. If you change the table to 10 × 10, the width becomes 3 on its own, and the table stays aligned. Each row ends with a space because `print!("{:>width$} ", …)` adds one after every cell.

</details>

## Quiz

1. Which macro writes to stderr and adds a new line?

<details>
<summary>Answer</summary>

`eprintln!`.

</details>

2. What is the difference between `{}` and `{:?}`?

<details>
<summary>Answer</summary>

`{}` uses Display (the readable form for people); `{:?}` uses Debug (the form for programmers, which shows quotes and structure). Tuples and arrays can only be printed with `{:?}`.

</details>

3. How do you print a literal `{` character?

<details>
<summary>Answer</summary>

Double it: `println!("{{")`.

</details>

4. What does `{:>8.2}` mean?

<details>
<summary>Answer</summary>

Right-aligned in a field 8 characters wide, with 2 digits after the decimal point.

</details>

5. What does `format!` return, and why is it useful for tests?

<details>
<summary>Answer</summary>

A `String` with the formatted text (nothing is printed). You can compare it with the expected text in `assert_eq!`.

</details>

6. What is the difference between `//` and `///`?

<details>
<summary>Answer</summary>

`//` is a normal comment. `///` is a doc comment: tools such as `cargo doc` turn it into documentation for the item that follows.

</details>

7. Why should a comment explain "why" and not "what"?

<details>
<summary>Answer</summary>

The code already shows what it does; only the comment can explain the reasons, the limits and the surprises that the code does not show. "What" comments become out of date and add noise.

</details>

## Summary

- `println!` and `print!` write to stdout; `eprintln!` and `eprint!` write to stderr. Results go to stdout, diagnostics to stderr.
- `{}` is Display (for people); `{:?}` is Debug (for programmers); `{:#?}` is pretty Debug.
- Prefer `{name}` for plain variables. Use `{{` and `}}` for literal braces.
- Specifiers: width, `<` `>` `^`, fill characters, `0`, `.N` precision, `+`, and `b` `o` `x` `X` `e`, with `#` for prefixes.
- `format!` returns a `String`; separate "making text" from "printing it" so that you can test it.
- Comments: `//` for notes, `///` and `//!` for documentation. Explain why, not what.

## Sources and further reading

- [Standard library — std::fmt](https://doc.rust-lang.org/std/fmt/index.html) — the full reference of every format specifier.
- [Rust by Example — Formatted print](https://doc.rust-lang.org/rust-by-example/hello/print.html) — runnable examples of `print!`, `{:?}` and formatting.
- [The Rust Programming Language — 3.4 Comments](https://doc.rust-lang.org/book/ch03-04-comments.html) — the official book chapter on comments.
- [The Rust Programming Language — 14.2 Publishing a Crate (documentation comments)](https://doc.rust-lang.org/book/ch14-02-publishing-to-crates-io.html) — how `///` comments become documentation.
- [Rust API Guidelines — Documentation](https://rust-lang.github.io/api-guidelines/documentation.html) — what a well-documented item looks like.

---

[🏠 Start](../START-HERE.md) · [← 02.5 Control flow: conditions and loops](05-control-flow.md) · [📋 Phase 02](README.md) · [02.7 Reading input and parsing text →](07-reading-input-and-parsing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
