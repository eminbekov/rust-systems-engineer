<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 04.7 Vectors](07-vectors.md) · [📋 Phase 04](README.md) · [04.9 Hash maps →](09-hash-maps.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/04-structs-enums-collections-errors/08-strings-in-depth.md) <!-- nav -->

# Lesson 04.8 — Strings in depth

> **Phase 04 — Structs, enums, collections and errors** · Lesson 8 of 12 · ⏱ about 4 hours (2 sittings)

## What you will learn

- How a computer stores text: bytes, Unicode and UTF-8, and why `"привет".len()` is not 6.
- How to create and grow a `String`, and the difference between `+` and `format!`.
- Why you cannot write `s[0]` on a string, and what to do instead.
- How to walk through text by bytes, by characters and by position.
- The everyday methods: `trim`, `split`, `split_once`, `find`, `replace`, `parse` and more.

## Before you start

- You finished [Lesson 04.7 — Vectors](07-vectors.md).
- You know the difference between `String` and `&str` ([Lesson 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md)) and how to read and parse text ([Lesson 02.7](../02-programming-basics/07-reading-input-and-parsing.md)).
- Create a project:

```bash
cd ~/projects/rust
cargo new code/p04-strings
cd code/p04-strings
```

## New words

| Word | Meaning |
|---|---|
| **byte** | A number from 0 to 255. The smallest piece of memory a program usually works with. Rust's type for it is `u8`. |
| **ASCII** (American Standard Code for Information Interchange) | An old table that gives a number to 128 characters: English letters, digits, punctuation and control codes. For example `A` is 65. |
| **Unicode** | A huge table that gives a unique number to every character of every writing system, plus symbols and emoji (over 150 000 of them). |
| **code point** | One number from the Unicode table, for example U+044F for the Cyrillic letter `я`. |
| **encoding** | A rule for turning code points into bytes (and back) so text can be stored or sent. |
| **UTF-8** (Unicode Transformation Format, 8-bit) | The most used encoding. It uses 1 to 4 bytes for each code point. Rust strings are always valid UTF-8. |
| **`char`** | A Rust type that holds one Unicode code point (called a *Unicode scalar value*). It is always 4 bytes wide. A literal is written with single quotes: `'a'`. |
| **`String`** | An owned, growable piece of text on the heap. Inside, it is a `Vec<u8>` that is guaranteed to hold valid UTF-8. |
| **`&str`** (string slice) | A borrowed view of text: a pointer and a length. String literals like `"hello"` have this type. |
| **char boundary** | A byte position where one character ends and the next begins. Slicing text is allowed only at boundaries. |
| **concatenation** | Joining two pieces of text into one longer piece. |
| **delimiter** | A character or text that separates parts in a longer text, for example the comma in `"a,b,c"`. |
| **FFI** (Foreign Function Interface) | A way for Rust code to call code written in another language, such as C. See [Lesson 06.10 — FFI](../06-rust-advanced/10-ffi-calling-c.md). |
| **grapheme cluster** | What a human reads as one "letter" on the screen. It can be several code points, for example `e` plus a separate accent mark. The standard library does not split text this way. |

## 1. How a computer stores text

A computer stores only numbers. To store text, people agreed on a table: "the number 65 means the
letter `A`". The first big table was **ASCII**. It had 128 entries, enough for English, and every
character took one byte.

The world has many more characters than 128: `é`, `я`, `世`, `😀`. **Unicode** is the modern
table that covers all of them. Each entry is a **code point**, written like `U+044F`.

A code point is a number, but we still need to decide how to put that number into bytes. That rule
is an **encoding**. The encoding used almost everywhere today (and the only one Rust strings use)
is **UTF-8**:

| Code points | Bytes used | Example |
|---|---|---|
| U+0000 to U+007F (ASCII) | 1 | `a` |
| U+0080 to U+07FF | 2 | `é`, `я` |
| U+0800 to U+FFFF | 3 | `世` |
| U+10000 to U+10FFFF | 4 | `😀` |

The nice property: plain English text takes one byte per letter, exactly like ASCII, and other
languages still work. Let us look at real bytes.

```rust
fn main() {
    let samples = ["a", "é", "я", "世", "😀"];
    for s in samples {
        println!("{:<3} chars = {}, bytes = {}, utf-8 = {:02x?}", s, s.chars().count(), s.len(), s.as_bytes());
    }
}
```

**What happens:**

1. `s.len()` returns the number of **bytes** in the text, not the number of letters.
2. `s.chars().count()` walks through the text character by character and counts them
   (`.chars()` makes an iterator over `char` values, like `.iter()` in [Lesson 04.7](07-vectors.md)).
3. `s.as_bytes()` shows the raw bytes as a slice of `u8`. The format `{:02x?}` prints each byte as
   two hexadecimal digits (base 16: digits `0-9` and letters `a-f`; `c3` means 195).
4. `{:<3}` pads the text to width 3, aligned to the left.

```text
a   chars = 1, bytes = 1, utf-8 = [61]
é   chars = 1, bytes = 2, utf-8 = [c3, a9]
я   chars = 1, bytes = 2, utf-8 = [d1, 8f]
世   chars = 1, bytes = 3, utf-8 = [e4, b8, 96]
😀   chars = 1, bytes = 4, utf-8 = [f0, 9f, 98, 80]
```

This is the key fact of the whole lesson: **one character can be 1, 2, 3 or 4 bytes.**
That is why Rust is careful about strings.

```text
String "aя😀" in memory (heap):

 byte offset:   0     1    2     3    4    5    6
              ┌────┬─────────┬──────────────────┐
              │ 61 │ d1   8f │ f0   9f   98  80 │
              └────┴─────────┴──────────────────┘
 character:      a      я            😀
 boundaries:  ^0    ^1        ^3                ^7
```

## 2. Creating a `String`

```rust
fn main() {
    let a = String::new();
    let b = String::from("hello");
    let c = "hello".to_string();
    let d = "hello".to_owned();
    let e: String = "hello".into();
    let f = format!("{}-{}", "hel", "lo");
    println!("[{a}] [{b}] [{c}] [{d}] [{e}] [{f}]");
    println!("{}", b == c);
    println!("{}", b == "hello");
}
```

**What happens:**

1. `String::new()` is an empty string, like `Vec::new()`.
2. `String::from("hello")`, `"hello".to_string()`, `"hello".to_owned()` and `"hello".into()`
   all do the same job: they copy the literal's text to the heap and give you an owned `String`.
   (`.into()` needs the type written on the left, `: String`. It uses a conversion rule you will
   study in [Lesson 05.2 — Traits](../05-rust-intermediate/02-traits.md).)
3. `format!` works like `println!` but returns a `String` instead of printing.
4. A `String` can be compared with another `String` or with a `&str` using `==`.

```text
[] [hello] [hello] [hello] [hello] [hel-lo]
true
true
```

> 📏 **Convention:** `String::from("text")` and `"text".to_string()` are both common. Pick one
> and use it everywhere in a project. This course uses `String::from`.

## 3. Growing a `String`

```rust
fn main() {
    let mut s = String::from("Rust");
    s.push(' ');
    s.push_str("is fun");
    s += "!";
    println!("{s}");

    let first = String::from("Hello, ");
    let second = String::from("world");
    let joined = first + &second;
    println!("{joined}");
    println!("{second}");

    let x = String::from("tic");
    let y = String::from("tac");
    let z = String::from("toe");
    let game = format!("{x}-{y}-{z}");
    println!("{game}");
    println!("{x} {y} {z}");
}
```

**What happens:**

1. `push(' ')` adds **one character** (a `char`, single quotes). `push_str("is fun")` adds a
   **piece of text** (a `&str`, double quotes). `s += "!"` is another way to write `push_str`.
2. `first + &second` is **concatenation**. It *moves* `first` into the operation, appends the
   text of `second`, and gives back the same buffer as `joined`. So `first` cannot be used any more.
   The right side must be a `&str`; `&second` is a `&String`, and Rust converts it for you.
   `second` is only borrowed, so it is still usable (the second `println!` proves it).
3. `format!("{x}-{y}-{z}")` builds a new `String` from several parts and **borrows** all of them.
   Nothing is moved, so the last line can still print `x`, `y` and `z`.

```text
Rust is fun!
Hello, world
world
tic-tac-toe
tic tac toe
```

> 💡 **Tip:** For two parts, `+` is fine. For three or more parts, use `format!`. It is easier to
> read and it does not move anything.

> ⚡ **Performance:** Every `push_str` may need more memory, just like `push` on a vector.
> `String` has a length and a capacity too, and `String::with_capacity(n)` reserves room in advance
> (the same idea as in [Lesson 04.7](07-vectors.md)). Do not use `+` in a loop to build a very long text; `push_str` on one
> `String` is better.

## 4. You cannot index a string

You might expect `text[0]` to give the first letter, as `numbers[0]` gives the first item of an array. For a string, this does not compile.

```rust
fn main() {
    let s = String::from("hello");
    let c = s[0];
    println!("{c}");
}
```

```text
error[E0277]: the type `str` cannot be indexed by `{integer}`
 --> src/main.rs:3:15
  |
3 |     let c = s[0];
  |               ^ string indices are ranges of `usize`
  |
  = note: you can use `.chars().nth()` or `.bytes().nth()`
```

**Why?** `s[0]` would have to return "the first character", but with UTF-8 a character may start at
any byte offset and be 1 to 4 bytes long. Rust cannot find the 5th character in constant time
(it must walk through the bytes from the start). Returning "the first byte" would give you half a
letter for `я`. So the designers chose to forbid the operation and make you say what you mean.

Say what you mean:

- the **n-th character**: `s.chars().nth(n)` (returns `Option<char>`);
- the **n-th byte**: `s.as_bytes()[n]` or `s.bytes().nth(n)`;
- a **part of the text** by byte positions: `&s[a..b]`.

The last one is allowed, but **both ends must be on character boundaries**, or the program panics.

```rust
fn main() {
    let s = String::from("Здравствуйте");
    let part = &s[0..4];
    println!("{part}");
    let bad = &s[0..3];
    println!("{bad}");
}
```

```text
Зд

thread 'main' (3885762) panicked at src/main.rs:5:17:
end byte index 3 is not a char boundary; it is inside 'д' (bytes 2..4 of string)
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**What happens:**

1. Each Cyrillic letter takes 2 bytes. Bytes 0..4 are exactly two letters, `Зд`, so the first
   slice works.
2. Bytes 0..3 would cut the second letter in half. The program panics and the message tells you
   which letter was cut.

To avoid a panic, use `get`, which returns an `Option` (like `Vec::get`), and
`is_char_boundary` to test a position:

```rust
fn main() {
    let text = "naïve café";
    println!("{}", text.is_char_boundary(3));
    println!("{}", text.is_char_boundary(2));
    println!("{:?}", text.get(0..3));
    println!("{:?}", text.get(0..4));
}
```

```text
false
true
None
Some("naï")
```

Here `ï` occupies bytes 2 and 3, so position 3 is in the middle of it.

> 🔒 **Security:** Text from a user, a file or the network can contain any characters. Code like
> `&input[..10]` to "shorten" it will panic when byte 10 is inside a letter, and an attacker can
> choose such input on purpose. Cut text by characters (Exercise 5) or use `get`.

## 5. Walking through text

There are three common views of the same text.

```rust
fn main() {
    let word = "Привет";
    println!("len()           = {}", word.len());
    println!("chars().count() = {}", word.chars().count());

    for c in word.chars() {
        print!("[{c}]");
    }
    println!();

    for b in "Hi!".bytes() {
        print!("{b} ");
    }
    println!();

    for (i, c) in "aя😀".char_indices() {
        println!("byte offset {i}: {c}");
    }
}
```

**What happens:**

1. `len()` counts bytes: 6 letters × 2 bytes = 12. `chars().count()` counts characters: 6.
2. `for c in word.chars()` gives one `char` per round. `print!` is like `println!` without the
   new line at the end; `println!()` alone then ends the line.
3. `.bytes()` gives the raw numbers: `H` is 72, `i` is 105, `!` is 33.
4. `.char_indices()` gives pairs of (byte offset, character). The offsets 0, 1, 3 match the
   picture in section 1: `a` starts at 0, `я` at 1, the emoji at 3.

```text
len()           = 12
chars().count() = 6
[П][р][и][в][е][т]
72 105 33 
byte offset 0: a
byte offset 1: я
byte offset 3: 😀
```

> A note about the word "character". A `char` is a *code point*, not always what a person sees
> as one letter. Some letters are written with two code points (a base letter plus a combining
> accent), and many emoji are several code points glued together. Splitting text into what humans
> see (grapheme clusters) needs the `unicode-segmentation` crate; you will learn how to add
> crates in [Lesson 05.8 — Cargo: dependencies and versions](../05-rust-intermediate/08-cargo-dependencies-and-versions.md). For most programs, `chars()` is enough.

## 6. Everyday methods

Most of these methods work on `&str`, so they also work on a `String` (Rust lets you call `&str`
methods on a `String` directly, and pass `&my_string` where a `&str` is expected).

### Cleaning and splitting

```rust
fn main() {
    let text = "  Alice, Bob,Carol ,  Dave  ";
    println!("[{}]", text.trim());

    for part in text.split(',') {
        println!("part: [{}]", part.trim());
    }

    let words: Vec<&str> = "one two   three".split_whitespace().collect();
    println!("{:?}", words);

    let line = "add alice +998-90-123-45-67";
    match line.split_once(' ') {
        Some((command, rest)) => println!("command = {command}, rest = {rest}"),
        None => println!("no space"),
    }

    let poem = "line one\nline two\nline three";
    for (n, l) in poem.lines().enumerate() {
        println!("{}: {}", n + 1, l);
    }
}
```

**What happens:**

1. `trim()` removes spaces, tabs and new lines from both ends. (`trim_start` and `trim_end` do one
   side.) It does not copy: it returns a smaller `&str` that points into the same text.
2. `split(',')` cuts the text at every comma. It returns an iterator, so you can use it in a
   `for` loop. Each part still has its spaces, so we call `trim()` on every part.
3. `split_whitespace()` cuts at any run of spaces, so the three spaces between `two` and `three`
   do not make an empty word.
4. `.collect()` collects the iterator's items into a collection. We say which collection with the
   type on the left: `Vec<&str>`. (How iterators and `collect` work is the subject of
   [Lesson 05.5](../05-rust-intermediate/05-iterators.md).)
5. `split_once(' ')` cuts at the **first** space only and returns `Option<(&str, &str)>`: the part
   before and the part after. It is the best tool for "command + the rest" parsing.
6. `lines()` splits at line ends.

```text
[Alice, Bob,Carol ,  Dave]
part: [Alice]
part: [Bob]
part: [Carol]
part: [Dave]
["one", "two", "three"]
command = add, rest = alice +998-90-123-45-67
1: line one
2: line two
3: line three
```

### Searching and changing

```rust
fn main() {
    let s = "Hello, World";
    println!("{}", s.contains("World"));
    println!("{}", s.starts_with("Hello"));
    println!("{}", s.ends_with('d'));
    println!("{:?}", s.find("World"));
    println!("{:?}", s.find('z'));
    println!("{}", s.replace("l", "L"));
    println!("{}", s.to_uppercase());
    println!("{}", s.to_lowercase());
    println!("{}", "ab".repeat(3));
    println!("{:?}", "add alice".strip_prefix("add "));
    println!("{:?}", "list".strip_prefix("add "));
    println!("{}", "Alice".eq_ignore_ascii_case("ALICE"));
    println!("{:?}", "42".parse::<u32>());
    println!("{:?}", "4x2".parse::<u32>());
}
```

**What happens:**

1. `contains`, `starts_with` and `ends_with` answer yes/no questions. They accept a `&str` or a
   single `char`.
2. `find` returns the **byte** offset of the first match as an `Option<usize>`: `Some(7)` for
   `World`, `None` when nothing matches.
3. `replace`, `to_uppercase`, `to_lowercase` and `repeat` do **not** change the original. They
   build and return a new `String`.
4. `strip_prefix("add ")` removes the beginning if it is there and returns `Some(rest)`; if the
   text does not start with the prefix it returns `None`.
5. `eq_ignore_ascii_case` compares two texts ignoring the case of English letters.
6. `parse::<u32>()` turns text into a number. It returns a `Result`, which is either `Ok(value)` or
   `Err(reason)`. You will study `Result` in [Lesson 04.10](10-error-handling.md).

```text
true
true
true
Some(7)
None
HeLLo, WorLd
HELLO, WORLD
hello, world
ababab
Some("alice")
None
true
Ok(42)
Err(ParseIntError { kind: InvalidDigit })
```

### Methods of `char`

```rust
fn main() {
    let c = 'q';
    println!("{} {} {}", c.is_alphabetic(), c.is_numeric(), c.is_ascii_digit());
    println!("{}", c.to_ascii_uppercase());
    println!("{:?}", '7'.to_digit(10));
    println!("{:?}", 'x'.to_digit(10));
    println!("{}", 'a' as u32);
    println!("{}", 'я' as u32);
    println!("{:?}", char::from_u32(0x1F600));
    println!("{}", c.len_utf8());
    println!("{}", 'я'.len_utf8());
}
```

**What happens:**

1. `is_alphabetic`, `is_numeric` and `is_ascii_digit` classify a character. `is_alphabetic`
   knows letters of all alphabets.
2. `'7'.to_digit(10)` converts a digit character to the number it represents (`10` is the base),
   or `None` if it is not a digit.
3. `'a' as u32` shows the code point: 97. `'я'` is 1103.
4. `char::from_u32(0x1F600)` builds a character from a code point, or `None` if the number is not
   a valid code point. `0x1F600` is a number written in hexadecimal (base 16).
5. `len_utf8()` tells how many bytes the character takes in UTF-8.

```text
true false false
Q
Some(7)
None
97
1103
Some('😀')
1
2
```

## 7. Testing text code

> 🧪 **Testing:** Always include a non-ASCII test for functions that handle text. A function that
> passes with `"hello"` may panic with `"привет"`. Here is a palindrome checker (a text that reads
> the same in both directions) with five tests, including Cyrillic.

```rust
/// Keeps only letters and digits, in lowercase.
fn normalize(text: &str) -> String {
    let lower = text.to_lowercase();
    let mut result = String::new();
    for c in lower.chars() {
        if c.is_alphanumeric() {
            result.push(c);
        }
    }
    result
}

fn is_palindrome(text: &str) -> bool {
    let chars: Vec<char> = normalize(text).chars().collect();
    let mut reversed = chars.clone();
    reversed.reverse();
    chars == reversed
}

fn main() {
    println!("{}", is_palindrome("A man, a plan, a canal: Panama"));
    println!("{}", is_palindrome("Rust"));
    println!("{}", is_palindrome("А роза упала на лапу Азора"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn simple_palindrome() {
        assert!(is_palindrome("level"));
    }

    #[test]
    fn ignores_case_and_punctuation() {
        assert!(is_palindrome("A man, a plan, a canal: Panama"));
    }

    #[test]
    fn works_with_non_ascii_letters() {
        assert!(is_palindrome("А роза упала на лапу Азора"));
    }

    #[test]
    fn not_a_palindrome() {
        assert!(!is_palindrome("Rust"));
    }

    #[test]
    fn empty_is_a_palindrome() {
        assert!(is_palindrome(""));
    }
}
```

**What happens:**

1. `normalize` lowercases the whole text, then copies only letters and digits into a new `String`.
2. `is_palindrome` collects the characters into a `Vec<char>`, makes a reversed copy with
   `clone()` and `reverse()` ([Lesson 04.7](07-vectors.md)), and compares the two vectors with `==`.
3. `assert!(condition)` fails the test when the condition is `false`; `assert!(!x)` expects `false`.

```text
true
false
true
```

```text
running 5 tests
test tests::not_a_palindrome ... ok
test tests::empty_is_a_palindrome ... ok
test tests::simple_palindrome ... ok
test tests::works_with_non_ascii_letters ... ok
test tests::ignores_case_and_punctuation ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## 8. Other string types you will meet

`String` and `&str` always hold valid UTF-8. The outside world is messier, so the standard
library has other types. You do not need them yet; this table only tells you they exist.

| Type | Used for | Where you will meet it |
|---|---|---|
| `OsString` / `&OsStr` | Names the operating system gives you (file names, command-line arguments), which may not be valid UTF-8 | file and path work in later phases |
| `PathBuf` / `&Path` | File-system paths | file work in later phases |
| `CString` / `&CStr` | Text for C code, which ends with a zero byte | [Lesson 06.10 — FFI](../06-rust-advanced/10-ffi-calling-c.md) |
| `Vec<u8>` / `&[u8]` | Raw bytes that are not necessarily text | network and file formats |

(FFI means Foreign Function Interface: calling code written in another language. It is explained in
[Lesson 06.10](../06-rust-advanced/10-ffi-calling-c.md).)

## Common mistakes

**1. Adding two `String`s with `+`.**

```rust
fn main() {
    let a = String::from("Hello, ");
    let b = String::from("world");
    let c = a + b;
    println!("{c}");
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:4:17
  |
4 |     let c = a + b;
  |                 ^ expected `&str`, found `String`
  |
help: consider borrowing here
  |
4 |     let c = a + &b;
  |                 +
```

The right side of `+` must be a `&str`. Fix: write `a + &b`.

**2. Using the left side after `+`.**

```rust
fn main() {
    let a = String::from("Hello, ");
    let b = String::from("world");
    let c = a + &b;
    println!("{c} {a}");
}
```

```text
error[E0382]: borrow of moved value: `a`
 --> src/main.rs:5:20
  |
4 |     let c = a + &b;
  |             - value moved here
5 |     println!("{c} {a}");
  |                    ^ value borrowed here after move
  |
help: consider cloning the value if the performance cost is acceptable
  |
4 |     let c = a.clone() + &b;
  |              ++++++++
```

`+` takes ownership of `a`. Fix: use `format!("{a}{b}")`, which borrows both, or `a.clone() + &b`.

**3. Indexing with a number** (`s[0]`): see section 4. Use `chars().nth(0)`.

**4. Slicing in the middle of a character:** see section 4. Use `get`, or work with `chars()`.

**5. Thinking `len()` is the number of letters.** It is the number of bytes. Use `chars().count()`
for characters.

**6. Forgetting that `trim()` does not change the original.** `s.trim();` alone does nothing useful.
Write `let s = s.trim();`.

## Try it yourself

In `code/p04-strings` replace `src/main.rs` with a tiny command parser, the same idea you will
use in the phone book project:

```rust
fn main() {
    let lines = ["add alice +998-90-123-45-67", "  get   alice  ", "count", ""];

    for line in lines {
        let line = line.trim();
        if line.is_empty() {
            println!("(empty line)");
            continue;
        }
        match line.split_once(' ') {
            Some((command, rest)) => println!("command {:?} with argument {:?}", command, rest.trim()),
            None => println!("command {:?} without arguments", line),
        }
    }
}
```

1. Run it with `cargo run`. Expected output:

```text
command "add" with argument "alice +998-90-123-45-67"
command "get" with argument "alice"
command "count" without arguments
(empty line)
```

2. Change the input so a line starts with an upper-case command, `"ADD bob 123"`. Make the program
   print the command in lower case with `to_lowercase()`.
3. Add a line with a tab character between the command and the argument (`"get\talice"`). Does
   `split_once(' ')` find the tab? (No. Try `split_whitespace()` or `split_once(char::is_whitespace)`.)
4. Commit your work to Git: `git add .` and `git commit -m "Practice string parsing"`.

## Exercises

### Exercise 1 — Bytes versus characters (easy)

For each of the words `"hello"`, `"привет"` and `"日本語"`, print the number of bytes and the number
of characters. Expected output:

```text
hello: 5 bytes, 5 chars
привет: 12 bytes, 6 chars
日本語: 9 bytes, 3 chars
```

<details>
<summary>Hint</summary>

Put the words into an array and use a `for` loop. `len()` counts bytes; `chars().count()`
counts characters.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    for word in ["hello", "привет", "日本語"] {
        println!("{word}: {} bytes, {} chars", word.len(), word.chars().count());
    }
}
```

Why it works: Latin letters use 1 byte each, Cyrillic letters 2 bytes, and these Japanese
characters 3 bytes each.

</details>

### Exercise 2 — Build a greeting (easy)

Given `let name = String::from("Sara");` and `let city = "Samarkand";`, build the text
`Hello, Sara. Welcome to Samarkand!` in two ways: with `push_str`/`push`/`+=` and with `format!`.
Print the first one, and print whether the two results are equal.

<details>
<summary>Hint</summary>

Start with `String::from("Hello, ")`. Use `push_str(&name)` (pass a reference), `+= ". Welcome to "`,
`push_str(city)` and `push('!')`. Compare two `String`s with `==`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let name = String::from("Sara");
    let city = "Samarkand";

    let mut greeting = String::from("Hello, ");
    greeting.push_str(&name);
    greeting += ". Welcome to ";
    greeting.push_str(city);
    greeting.push('!');
    println!("{greeting}");

    let same = format!("Hello, {name}. Welcome to {city}!");
    println!("{}", greeting == same);
}
```

```text
Hello, Sara. Welcome to Samarkand!
true
```

Why it works: `push_str` borrows the text it appends, so `name` is still usable. `format!` gives the
same result with less typing.

</details>

### Exercise 3 — Initials, with tests (medium)

Write `fn initials(name: &str) -> String` that turns `"Ali Valiyev"` into `"A.V."`. Extra spaces
must be ignored, letters must become upper case, and an empty input gives an empty `String`. Write
tests for: two words, extra spaces with lower case, empty input and a non-ASCII name (`"олег иванов"`
gives `"О.И."`).

<details>
<summary>Hint</summary>

Loop over `name.split_whitespace()`. For each word, `word.chars().next()` returns an
`Option<char>` (the first character, if any). Use `if let Some(first) = ...`. Converting to
upper case: `first.to_uppercase()` returns something you can loop over, because one character can
become several in upper case. Push each result and then push `'.'`.

</details>

<details>
<summary>Answer</summary>

```rust
/// "Ali Valiyev" -> "A.V."
fn initials(name: &str) -> String {
    let mut result = String::new();
    for word in name.split_whitespace() {
        if let Some(first) = word.chars().next() {
            for upper in first.to_uppercase() {
                result.push(upper);
            }
            result.push('.');
        }
    }
    result
}

fn main() {
    println!("{}", initials("Ali Valiyev"));
    println!("{}", initials("  mary   jane watson "));
    println!("[{}]", initials(""));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn two_words() {
        assert_eq!(initials("Ali Valiyev"), "A.V.");
    }

    #[test]
    fn extra_spaces_and_lowercase() {
        assert_eq!(initials("  mary   jane watson "), "M.J.W.");
    }

    #[test]
    fn empty_input() {
        assert_eq!(initials(""), "");
    }

    #[test]
    fn non_ascii() {
        assert_eq!(initials("олег иванов"), "О.И.");
    }
}
```

```text
A.V.
M.J.W.
[]
```

Why it works: `split_whitespace` removes the empty pieces, so extra spaces are harmless. A word
always has a first character, but `chars().next()` returns an `Option`, so the code handles the
"no character" case without any chance of a panic. `assert_eq!` can compare a `String` with a `&str`.

</details>

### Exercise 4 — Parse `key = value` (medium)

Write `fn parse_kv(line: &str) -> Option<(&str, &str)>` for configuration lines. `"name = Ali"`
gives `Some(("name", "Ali"))` (spaces around key and value are removed). A line without `=` gives
`None`. An empty key gives `None`. Only the **first** `=` splits the line: `"a=b=c"` gives
`Some(("a", "b=c"))`. Write a test for each rule.

<details>
<summary>Hint</summary>

`line.split_once('=')` returns an `Option<(&str, &str)>`. Use `let Some((key, value)) = ... else {
return None; };` ([Lesson 04.5](05-if-let-let-else-and-patterns.md)). Then `trim()` both parts and check `key.is_empty()`.

</details>

<details>
<summary>Answer</summary>

```rust
fn parse_kv(line: &str) -> Option<(&str, &str)> {
    let Some((key, value)) = line.split_once('=') else {
        return None;
    };
    let key = key.trim();
    if key.is_empty() {
        return None;
    }
    Some((key, value.trim()))
}

fn main() {
    println!("{:?}", parse_kv("name = Ali"));
    println!("{:?}", parse_kv("port=8080"));
    println!("{:?}", parse_kv("no equals sign"));
    println!("{:?}", parse_kv(" = value"));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn trims_both_sides() {
        assert_eq!(parse_kv("  name =  Ali "), Some(("name", "Ali")));
    }

    #[test]
    fn missing_equals() {
        assert_eq!(parse_kv("hello"), None);
    }

    #[test]
    fn empty_key() {
        assert_eq!(parse_kv("=x"), None);
    }

    #[test]
    fn value_may_contain_equals() {
        assert_eq!(parse_kv("a=b=c"), Some(("a", "b=c")));
    }
}
```

```text
Some(("name", "Ali"))
Some(("port", "8080"))
None
None
```

Why it works: the result holds slices of the input, not copies, so no memory is allocated. (Why
the returned references are valid is explained in [Lesson 05.3 — Lifetimes](../05-rust-intermediate/03-lifetimes.md).)

</details>

### Exercise 5 — Safe truncation (hard)

Write `fn truncate_chars(s: &str, n: usize) -> &str` that keeps at most `n` **characters** and
never panics, even for Cyrillic text or emoji. Tests: `("hello world", 5)` gives `"hello"`;
`("привет мир", 6)` gives `"привет"`; `("hi", 10)` gives `"hi"`; `("abc", 0)` gives `""`;
`("a😀b", 2)` gives `"a😀"`.

<details>
<summary>Hint</summary>

`s.char_indices().nth(n)` returns the `n`-th pair `(byte_offset, char)` as an `Option`. If it is
`Some((i, _))`, the text before byte `i` is exactly `n` characters, so return `&s[..i]`. If it is
`None`, the text has `n` characters or fewer, so return all of `s`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Keeps at most `n` characters. Never panics, even for non-ASCII text.
fn truncate_chars(s: &str, n: usize) -> &str {
    match s.char_indices().nth(n) {
        Some((byte_index, _)) => &s[..byte_index],
        None => s,
    }
}

fn main() {
    println!("{}", truncate_chars("hello world", 5));
    println!("{}", truncate_chars("привет мир", 6));
    println!("{}", truncate_chars("hi", 10));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn ascii() {
        assert_eq!(truncate_chars("hello world", 5), "hello");
    }

    #[test]
    fn cyrillic() {
        assert_eq!(truncate_chars("привет мир", 6), "привет");
    }

    #[test]
    fn shorter_than_limit() {
        assert_eq!(truncate_chars("hi", 10), "hi");
    }

    #[test]
    fn zero() {
        assert_eq!(truncate_chars("abc", 0), "");
    }

    #[test]
    fn emoji() {
        assert_eq!(truncate_chars("a😀b", 2), "a😀");
    }
}
```

```text
hello
привет
hi
```

Why it works: `char_indices` only produces offsets that are real character boundaries, so
`&s[..byte_index]` can never cut a character in half.

</details>

## Quiz

1. Why is `"привет".len()` equal to 12 and not 6?

<details>
<summary>Answer</summary>

`len()` returns the number of bytes. In UTF-8 each Cyrillic letter takes 2 bytes, and there are
6 letters. `chars().count()` returns 6.

</details>

2. What is the difference between a `char` and a `u8`?

<details>
<summary>Answer</summary>

A `u8` is one byte (0 to 255). A `char` is one Unicode code point and is 4 bytes wide; in a UTF-8
string, one `char` takes 1 to 4 bytes.

</details>

3. Why does Rust not allow `s[0]` on a `String`?

<details>
<summary>Answer</summary>

Characters have different byte lengths in UTF-8, so a byte index does not correspond to a
character, and finding the n-th character needs a walk through the text. Rust forces you to choose
what you mean: `chars().nth(n)`, `bytes()`, or a byte range slice.

</details>

4. What is the difference between `a + &b` and `format!("{a}{b}")`?

<details>
<summary>Answer</summary>

`a + &b` moves `a` (it cannot be used afterwards) and reuses its buffer. `format!` only borrows
both and builds a new `String`, so `a` and `b` stay usable.

</details>

5. What happens with `&s[0..3]` if the second character of `s` is `д` (2 bytes)?

<details>
<summary>Answer</summary>

The range ends inside a character, so the program panics with "is not a char boundary". Use
`s.get(0..3)` to get an `Option` instead.

</details>

6. Which method splits `"add alice 555"` into `"add"` and `"alice 555"`?

<details>
<summary>Answer</summary>

`split_once(' ')`. It splits at the first space only and returns `Some(("add", "alice 555"))`.

</details>

7. Does `s.to_uppercase()` change `s`?

<details>
<summary>Answer</summary>

No. It returns a new `String`, and the original stays as it was.

</details>

## Summary

- Text is stored as bytes. Rust strings are always valid UTF-8, where a character takes 1 to 4 bytes.
- `len()` is bytes; `chars().count()` is characters.
- `String` is owned and growable; `&str` is a borrowed view. Create strings with `String::from`,
  `to_string` or `format!`.
- Use `push` for a character, `push_str` or `+=` for text, `format!` to join many parts without
  moving them, and avoid `+` for more than two parts.
- You cannot write `s[0]`. Use `chars()`, `bytes()`, `char_indices()` or a slice with valid
  boundaries; use `get` for safety.
- `trim`, `split`, `split_whitespace`, `split_once`, `lines`, `find`, `replace` and `parse` do
  most daily work; none of them changes the original text.
- Always test text functions with non-ASCII input, and never slice untrusted text by raw byte
  positions.

## Sources and further reading

- [The Rust Programming Language — 8.2 Storing UTF-8 Encoded Text with Strings](https://doc.rust-lang.org/book/ch08-02-strings.html) — the official book chapter on strings.
- [std::string::String](https://doc.rust-lang.org/std/string/struct.String.html) — all methods of `String`.
- [std::primitive::str](https://doc.rust-lang.org/std/primitive.str.html) — all methods of `&str`, including `split`, `trim` and `find`.
- [std::primitive::char](https://doc.rust-lang.org/std/primitive.char.html) — all methods of `char`.
- [Unicode FAQ — UTF-8, UTF-16, UTF-32 and BOM](https://www.unicode.org/faq/utf_bom.html) — answers about UTF-8, UTF-16 and byte order.
- [Rust by Example — Strings](https://doc.rust-lang.org/rust-by-example/std/str.html) — short runnable examples.

---

[🏠 Start](../START-HERE.md) · [← 04.7 Vectors](07-vectors.md) · [📋 Phase 04](README.md) · [04.9 Hash maps →](09-hash-maps.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
