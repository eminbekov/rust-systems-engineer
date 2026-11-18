<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 03.5 Mutable references and the borrowing rules](05-mutable-references-and-the-borrow-rules.md) · [📋 Phase 03](README.md) · [03.7 Fixing common ownership errors →](07-fixing-common-ownership-errors.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/03-ownership-and-borrowing/06-slices-and-string-types.md) <!-- nav -->

# Lesson 03.6 — Slices and string types

> **Phase 03 — Ownership and borrowing** · Lesson 6 of 8 · ⏱ about 3 hours (2 sittings)

## What you will learn

- What a **slice** is: a borrowed view of part of a string or an array.
- Why a slice is safer than storing an index.
- How text is stored (UTF-8), why `s[0]` is not allowed, and why `len()` counts bytes.
- The difference between `String` and `&str`, and which one to use in function parameters.
- How to convert between them, and the useful `&str` methods you need for the project.

## Before you start

- You finished [Lesson 03.5 — Mutable references and the borrowing rules](05-mutable-references-and-the-borrow-rules.md).
- You know `String::from`, `push_str` and the idea of a string literal from [Lesson 02.2 — Data types](../02-programming-basics/02-data-types.md).
- Create a project: `cargo new code/p03-slices` (from `~/projects/rust`).

## New words

| Word | Meaning |
|---|---|
| **slice** | A reference to a continuous part of a sequence (of text or of items). It is a pointer and a length. Written `&str` for text and `&[T]` for items. |
| **`&str`** (string slice) | A borrowed view of UTF-8 text. String literals have this type. |
| **`String`** | An owned, growable, heap-allocated piece of UTF-8 text. |
| **string literal** | Text written in the code between double quotes, like `"hello"`. It is stored inside the program file and has the type `&str`. |
| **UTF-8** (Unicode Transformation Format, 8-bit) | The standard way to store text as bytes. Each character takes 1 to 4 bytes. |
| **Unicode** | The worldwide list of characters (letters, digits, symbols, emoji) with a number for each. |
| **ASCII** (American Standard Code for Information Interchange) | The old, small character set of English letters, digits and punctuation. Each ASCII character takes exactly 1 byte in UTF-8. |
| **char boundary** | A byte position in a string where a character starts. Slicing is only allowed on boundaries. |
| **range** | Two numbers with `..` between them, like `0..5`. It means "from 0 up to, but not including, 5". |
| **deref coercion** | An automatic conversion that lets a `&String` be used where a `&str` is expected. |
| **Option** | A value that is either `Some(something)` or `None` ("nothing"). Methods like `find` use it when there may be no answer. Lesson 04.4 teaches it fully. |

## 1. The problem: an index that goes stale

Suppose we want the first word of a text. The simplest design returns the *position* where the first word ends:

```rust
fn space_position(s: &String) -> usize {
    for (i, byte) in s.bytes().enumerate() {
        if byte == b' ' {
            return i;
        }
    }
    s.len()
}

fn main() {
    let mut text = String::from("hello world");
    let end = space_position(&text);
    text.clear(); // the text is now empty, but `end` is still 5
    println!("end = {end}");
    println!("first word: {}", &text[0..end]);
}
```

**What happens:**

1. `s.bytes()` gives the bytes of the text one by one. `.enumerate()` pairs each byte with its position, so `i` is the position. (These are *iterators*; Lesson 05.5.) `b' '` is the byte value of a space character.
2. The function returns the position of the first space, here `5`.
3. `text.clear()` empties the string. The number `5` in `end` knows nothing about it.
4. `&text[0..end]` asks for bytes 0 to 5 of an empty string. It crashes.

```text
end = 5
thread 'main' panicked at src/main.rs:15:37:
end byte index 5 is out of bounds for string of length 0
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

The compiler could not help: the number `5` is just a number, with no connection to the string. This kind of bug is very easy to write. We would like the compiler to **keep the connection** between "the first word" and the string it comes from. That is what slices do.

## 2. String slices: `&str`

A **string slice** is a reference to a part of a `String`. You make it with a range in square brackets:

```rust
fn main() {
    let s = String::from("hello world");
    println!("[0..5]  -> '{}'", &s[0..5]);
    println!("[6..11] -> '{}'", &s[6..11]);
    println!("[..5]   -> '{}'", &s[..5]);
    println!("[6..]   -> '{}'", &s[6..]);
    println!("[..]    -> '{}'", &s[..]);
    println!("[0..=4] -> '{}'", &s[0..=4]);
}
```

```text
[0..5]  -> 'hello'
[6..11] -> 'world'
[..5]   -> 'hello'
[6..]   -> 'world'
[..]    -> 'hello world'
[0..=4] -> 'hello'
```

The range rules, all counted in **bytes**, starting at 0:

| Range | Meaning |
|---|---|
| `a..b` | from `a` up to, but **not** including, `b` |
| `a..=b` | from `a` up to and **including** `b` |
| `..b` | from the start up to `b` (not including) |
| `a..` | from `a` to the end |
| `..` | everything |

A slice is a pointer to the first byte of the part, plus its length. It owns nothing:

```text
 s: String                     HEAP
┌──────────────┐              ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│ ptr ─────────┼────────────► │ h │ e │ l │ l │ o │   │ w │ o │ r │ l │ d │
│ len = 11     │              └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
│ cap = 11     │                ▲                       ▲
└──────────────┘                │                       │
 &s[0..5]                       │                       │
┌──────────────┐                │                 &s[6..11]
│ ptr ─────────┼────────────────┘                ┌──────────────┐
│ len = 5      │                                 │ ptr ─────────┼──┘ (points at 'w')
└──────────────┘                                 │ len = 5      │
                                                 └──────────────┘
```

The type of a string slice is written **`&str`** (read it "string slice" or "ref str"). A `&str` is 16 bytes: a pointer (8) and a length (8). There is no capacity, because a slice cannot grow.

Now the better design: return a slice instead of a number.

```rust
fn first_word(s: &str) -> &str {
    for (i, byte) in s.bytes().enumerate() {
        if byte == b' ' {
            return &s[0..i];
        }
    }
    s
}

fn main() {
    let mut text = String::from("hello world");
    let word = first_word(&text);
    println!("first word: {word}");

    text.clear(); // fine: `word` is not used any more
    println!("text is now: '{text}'");
}
```

```text
first word: hello
text is now: ''
```

(The signature `fn first_word(s: &str) -> &str` says that the result is a view into the input. The compiler understands it without more words, because there is only one input reference. Lesson 05.3 explains the rule.)

The slice `word` is a **borrow** of `text`. That means the rules of Lesson 03.5 apply. If we try to use `word` *after* clearing the string:

```rust
fn main() {
    let mut text = String::from("hello world");
    let word = first_word(&text);
    text.clear();
    println!("first word: {word}");
}
```

```text
error[E0502]: cannot borrow `text` as mutable because it is also borrowed as immutable
  --> src/main.rs:13:5
   |
12 |     let word = first_word(&text);
   |                           ----- immutable borrow occurs here
13 |     text.clear();
   |     ^^^^^^^^^^^^ mutable borrow occurs here
14 |     println!("first word: {word}");
   |                            ---- immutable borrow later used here
```

The stale-index bug from section 1 is now a **compile-time error**. This is the whole point of slices: the compiler knows that `word` depends on `text`.

## 3. Text is UTF-8: bytes are not characters

Computers store text as numbers. The old ASCII table had only 128 characters (English letters, digits, punctuation), each fitting in one byte. The world needs far more: Cyrillic, Chinese, Arabic, emoji. **Unicode** gives every character a number, and **UTF-8** is the way to write those numbers into bytes: English letters take 1 byte, Cyrillic letters take 2, most Chinese characters take 3, and emoji take 4.

A Rust `String` and `&str` always hold valid UTF-8. That has consequences:

```rust
fn main() {
    let words = ["hello", "привет", "日本", "🦀"];
    for w in words {
        println!("{w}: {} bytes, {} chars", w.len(), w.chars().count());
    }
}
```

```text
hello: 5 bytes, 5 chars
привет: 12 bytes, 6 chars
日本: 6 bytes, 2 chars
🦀: 4 bytes, 1 chars
```

**What happens:** `w.len()` counts **bytes**. `w.chars()` goes through the **characters** (Rust's `char`, a Unicode scalar value), and `.count()` counts them. For English text the two numbers are equal; for other text they differ.

> ⚠️ **Common mistake:** assuming `s.len()` is the number of letters. It is the number of bytes. Use `s.chars().count()` for the number of characters. (Even that is not always "what a person sees as one letter" — an emoji with a skin tone is two `char`s. The size of text has three meanings: bytes, `char`s, and what the eye sees. For this phase, bytes and `char`s are enough.)

Slice ranges are in bytes, and they must fall on **char boundaries**:

```rust
fn main() {
    let s = String::from("привет");
    println!("{}", &s[0..2]);
    println!("{}", &s[0..1]);
}
```

```text
п
thread 'main' panicked at src/main.rs:4:22:
end byte index 1 is not a char boundary; it is inside 'п' (bytes 0..2 of string)
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

The first slice, bytes 0 to 2, is exactly the letter `п` (2 bytes). The second cuts the letter in half, and the program **panics** (stops with an error message) at run time. This is why Rust does not let you index a string with a single number:

```rust
fn main() {
    let s = String::from("hello");
    let first = s[0];
    println!("{first}");
}
```

```text
error[E0277]: the type `str` cannot be indexed by `{integer}`
 --> src/main.rs:3:19
  |
3 |     let first = s[0];
  |                   ^ string indices are ranges of `usize`
  |
  = help: the trait `SliceIndex<str>` is not implemented for `{integer}`
  = note: you can use `.chars().nth()` or `.bytes().nth()`
          for more information, see chapter 8 in The Book: <https://doc.rust-lang.org/book/ch08-02-strings.html#indexing-into-strings>
```

(The real message continues with a few more technical notes; the key lines are above.) If `s[0]` were allowed, would it be the first byte or the first character? Rust refuses to guess. Use `s.chars().nth(0)` for the first character, or a slice range when you know the boundaries. Lesson 04.8 goes deeper into strings.

> 🔒 **Security:** text that comes from outside (files, network, users) may be broken or hostile. Slicing it with computed positions can panic and stop your program — a possible *denial of service*. When the positions come from input, prefer methods like `find`, `split_whitespace` or `chars` that handle boundaries for you.

## 4. `String` and `&str`: two types, one idea

| | `String` | `&str` |
|---|---|---|
| Owns its text? | **yes** | no, it borrows |
| Where is the text? | heap (growable) | anywhere: the heap, the stack, or the program file |
| Can grow or change? | yes (if `mut`) | no, read-only view |
| Size on the stack | 24 bytes (ptr, len, cap) | 16 bytes (ptr, len) |
| Created from | `String::from("x")`, `"x".to_string()`, `format!(...)` | `"literal"`, `&some_string`, `&s[a..b]` |

A **string literal** like `"hello"` is a `&str`. The text sits inside your compiled program (read-only), and the variable holds a pointer to it and its length. That explains why a literal cannot grow: you do not own it.

Common ways to make and combine strings:

```rust
fn main() {
    let a = String::from("hello");
    let b = "world".to_string();
    let c = "!".to_owned();

    let mut s = String::new();
    s.push_str(&a);
    s.push(' ');
    s.push_str(&b);
    s.push_str(&c);
    println!("{s}");

    let t = format!("{a}-{b}-{c}");
    println!("{t}");
    println!("{a} {b} {c}");
}
```

```text
hello world!
hello-world-!
hello world !
```

**What happens:** `String::from`, `.to_string()` and `.to_owned()` all turn a `&str` into an owned `String`. `push_str` appends a `&str` (so we pass `&a`, a `&String`, which Rust converts — see section 5). `push` appends one `char`. `format!` works like `println!` but returns a new `String`, and it only borrows its arguments (the last line shows `a`, `b`, `c` are still ours).

The `+` operator on strings has a surprise: it **moves** its left side.

```rust
fn main() {
    let s1 = String::from("hello, ");
    let s2 = String::from("world");
    let s3 = s1 + &s2;
    println!("{s3}");
    println!("{s1}");
}
```

```text
error[E0382]: borrow of moved value: `s1`
 --> src/main.rs:6:16
  |
2 |     let s1 = String::from("hello, ");
  |         -- move occurs because `s1` has type `String`, which does not implement the `Copy` trait
3 |     let s2 = String::from("world");
4 |     let s3 = s1 + &s2;
  |              -- value moved here
5 |     println!("{s3}");
6 |     println!("{s1}");
  |                ^^ value borrowed here after move
```

`s1 + &s2` takes ownership of `s1`, appends the text of `s2` to it (reusing `s1`'s buffer, so no new allocation is needed in the best case), and returns it as `s3`. The right side must be a `&str`: passing `s2` itself is the type error `expected &str, found String` (`E0308`), fixed by writing `&s2`. The corrected program keeps `s2` and loses `s1`:

```rust
fn main() {
    let s1 = String::from("hello, ");
    let s2 = String::from("world");
    let s3 = s1 + &s2; // s1 is moved into s3, s2 is only borrowed
    println!("{s3}");
    println!("{s2}");
}
```

```text
hello, world
world
```

For more than two pieces, `format!` is clearer than a chain of `+`.

A `&str` cannot be changed, because it is a read-only view:

```rust
fn main() {
    let s: &str = "hello";
    s.push_str(" world");
    println!("{s}");
}
```

```text
error[E0599]: no method named `push_str` found for reference `&str` in the current scope
 --> src/main.rs:3:7
  |
3 |     s.push_str(" world");
  |       ^^^^^^^^ method not found in `&str`
```

If you need to change text, you need a `String`.

## 5. Use `&str` in function parameters

This is the most important convention of the lesson:

> 📏 **Convention:** when a function only reads text, take **`&str`**, not `&String`.

Why? A `&str` parameter accepts *everything*: a literal, a whole `String` and a part of a string.

```rust
fn shout(text: &str) -> String {
    text.to_uppercase()
}

fn main() {
    let owned = String::from("owned string");
    println!("{}", shout(&owned));          // &String becomes &str
    println!("{}", shout("a literal"));     // a literal is already &str
    println!("{}", shout(&owned[0..5]));    // a slice is &str too
}
```

```text
OWNED STRING
A LITERAL
OWNED
```

**What happens:** the compiler quietly turns `&String` into `&str` where a `&str` is expected. This is **deref coercion**. A function with a `&String` parameter would refuse the literal and the slice, so it would be less useful. `cargo clippy` even warns about `&String` parameters (you will see it in Lesson 03.7). The same rule applies to vectors: take `&[T]` instead of `&Vec<T>`.

## 6. Slices of other things: `&[T]`

Slices are not only for text. Any continuous sequence of items — an array or a `Vec` — can be sliced. The type is `&[T]`, "a slice of `T`s".

```rust
fn sum(numbers: &[i32]) -> i32 {
    let mut total = 0;
    for n in numbers {
        total += n;
    }
    total
}

fn main() {
    let array = [1, 2, 3, 4, 5];
    let vector = vec![10, 20, 30];

    println!("whole array:  {}", sum(&array));
    println!("array [1..4]: {}", sum(&array[1..4]));
    println!("vector:       {}", sum(&vector));
    println!("vector [..2]: {}", sum(&vector[..2]));
    println!("size of &[i32]: {} bytes", size_of::<&[i32]>());
    println!("size of &str:   {} bytes", size_of::<&str>());
}
```

```text
whole array:  15
array [1..4]: 9
vector:       60
vector [..2]: 30
size of &[i32]: 16 bytes
size of &str:   16 bytes
```

One function now works for arrays, vectors and parts of both. A `&str` is really the same idea as `&[u8]` with the promise that the bytes are valid UTF-8.

There is also a mutable slice, `&mut [T]`, that lets a function change the items (but not the length):

```rust
fn zero_first_two(numbers: &mut [i32]) {
    numbers[0] = 0;
    numbers[1] = 0;
}

fn main() {
    let mut data = [9, 8, 7, 6];
    zero_first_two(&mut data);
    println!("{:?}", data);
    zero_first_two(&mut data[2..]);
    println!("{:?}", data);
}
```

```text
[0, 0, 7, 6]
[0, 0, 0, 0]
```

The second call passes only the part `data[2..]`. The function sees a slice of length 2 and changes its first two items, which are really `data[2]` and `data[3]`. The borrowing rules of Lesson 03.5 apply to slices, too.

## 7. Handy `&str` methods

You will use these constantly (including in the project). They all work on `&str` and `String`:

```rust
fn main() {
    let line = "  Rust is fast, safe and fun  ";
    println!("trim:      '{}'", line.trim());
    println!("words:     {}", line.split_whitespace().count());
    println!("starts:    {}", line.trim().starts_with("Rust"));
    println!("contains:  {}", line.contains("safe"));
    println!("find ',':  {:?}", line.find(','));
    println!("find 'z':  {:?}", line.find('z'));
    println!("lower:     '{}'", line.trim().to_lowercase());
    println!("replace:   '{}'", line.trim().replace("fast", "quick"));

    let poem = "one\ntwo\nthree";
    for l in poem.lines() {
        println!("line: {l}");
    }
    for w in "to be or not".split_whitespace() {
        println!("word: {w}");
    }
}
```

```text
trim:      'Rust is fast, safe and fun'
words:     6
starts:    true
contains:  true
find ',':  Some(14)
find 'z':  None
lower:     'rust is fast, safe and fun'
replace:   'Rust is quick, safe and fun'
line: one
line: two
line: three
word: to
word: be
word: or
word: not
```

| Method | What it does | Returns |
|---|---|---|
| `trim()` | removes spaces and line breaks at both ends | `&str` (a slice of the original!) |
| `split_whitespace()` | goes through the words separated by any whitespace | something you can use in `for` |
| `lines()` | goes through the lines | something you can use in `for` |
| `chars()` | goes through the characters | something you can use in `for` |
| `starts_with(x)`, `contains(x)` | tests | `bool` |
| `find(x)` | position (in bytes) of the first match | `Option<usize>`: `Some(14)` or `None` |
| `to_lowercase()`, `to_uppercase()`, `replace(a, b)` | build changed text | a new `String` |
| `len()`, `is_empty()` | bytes, and "no bytes" | `usize`, `bool` |

An important detail: methods that only *look* (`trim`, `split_whitespace`, `lines`) return slices of the original text. They are cheap: no allocation. Methods that *change* text (`to_lowercase`, `replace`) must build a new `String`.

`find` returns an **`Option`**, because there might be no match. `Some(14)` means "found at byte 14", and `None` means "not found". Lesson 04.4 teaches `Option` and `match` in full.

## 8. A few related types

- **`char`** — one Unicode character, written with single quotes: `'a'`, `'я'`, `'🦀'`. It is 4 bytes.
- **`&[u8]`** — raw bytes with no promise about text. Files and networks give bytes; you convert them to text (checking that they are valid UTF-8).
- **`Path` / `OsStr`** — file names are *not always* valid UTF-8 on a real operating system, so the standard library has special types for them. You will meet them in Phase 11.

## Common mistakes

**1. Using `s.len()` as the number of letters.** It counts bytes.

**2. Slicing at a computed position inside a character.** The program panics. Use `char_indices()` or `find` to get valid positions.

**3. Writing `fn f(s: &String)` or `fn f(v: &Vec<T>)`.** Use `&str` and `&[T]`: more flexible, same cost.

**4. Writing `s1 + s2` instead of `s1 + &s2`.** Error `E0308`: expected `&str`, found `String`.

**5. Forgetting that `s1 + &s2` consumes `s1`.** Use `format!` or `s1.clone() + &s2`.

**6. Trying to `push_str` on a `&str`.** Error `E0599`. Make a `String` (`to_string()`).

**7. Holding a slice and then changing the string.** Error `E0502`. This is the compiler protecting you from a stale view.

## Try it yourself

1. In `~/projects/rust`, run `cargo new code/p03-slices` (if you have not already done it in "Before you start"). Write the `first_word` program. Run it.
2. Move `text.clear();` to just *before* the first `println!` that uses `word`. Read the error; find the three labels (immutable borrow, mutable borrow, "later used here").
3. Print the `len()` and `chars().count()` of `"café"`, `"naïve"` and `"Привет, мир"`. Predict first.
4. Write `fn shout(text: &str) -> String` and call it with a literal, with `&String` and with a slice.
5. Commit: `git add code/p03-slices && git commit -m "p03: slices and strings"`.

## Exercises

### Exercise 1 — Predict the slices (easy)

What does each line print?

```rust
fn main() {
    let s = "systems";
    println!("1: '{}'", &s[..3]);
    println!("2: '{}'", &s[3..]);
    println!("3: '{}'", &s[1..=2]);
    println!("4: '{}'", &s[2..2]);
    println!("5: '{}'", &s[..]);
}
```

<details>
<summary>Hint</summary>

The letters are numbered from 0: `s`=0, `y`=1, `s`=2, `t`=3, `e`=4, `m`=5, `s`=6. The end of `a..b` is not included; the end of `a..=b` is.

</details>

<details>
<summary>Answer</summary>

```text
1: 'sys'
2: 'tems'
3: 'ys'
4: ''
5: 'systems'
```

Why it works: `..3` is bytes 0, 1, 2. `3..` is bytes 3 to the end. `1..=2` is bytes 1 and 2. `2..2` is an empty range, and an empty slice is valid. `..` is everything.

</details>

### Exercise 2 — Bytes or characters? (easy)

Predict `len()` and `chars().count()` for each string, then run the program to check:

```rust
fn main() {
    for text in ["abc", "héllo", "日本語", "👍🏽", ""] {
        println!("{:?}: {} bytes, {} chars", text, text.len(), text.chars().count());
    }
}
```

<details>
<summary>Hint</summary>

ASCII letters take 1 byte, `é` takes 2, each of these Chinese-script characters takes 3, and an emoji takes 4. The thumbs-up with a skin tone is actually two characters.

</details>

<details>
<summary>Answer</summary>

```text
"abc": 3 bytes, 3 chars
"héllo": 6 bytes, 5 chars
"日本語": 9 bytes, 3 chars
"👍🏽": 8 bytes, 2 chars
"": 0 bytes, 0 chars
```

Why it works: `{:?}` prints the string with quotes. `héllo` is 4 ASCII letters (4 bytes) plus `é` (2 bytes) = 6. The thumbs-up emoji (4 bytes) is followed by a skin-tone modifier character (4 bytes): 2 chars, 8 bytes, but one image on screen.

</details>

### Exercise 3 — File extension, with tests (medium)

Write `fn file_extension(name: &str) -> &str` that returns the part after the **last** dot, or an empty string when there is no dot. Write tests for `"photo.png"` → `"png"`, `"archive.tar.gz"` → `"gz"`, `"README"` → `""` and `"odd."` → `""`.

<details>
<summary>Hint</summary>

`name.rfind('.')` searches from the right and returns an `Option<usize>`. Use a `match` with two arms: `Some(position) => &name[position + 1..]` and `None => ""`. (`match` on an `Option` is covered in Lesson 04.4; the shape of the code is given here.)

</details>

<details>
<summary>Answer</summary>

```rust
fn file_extension(name: &str) -> &str {
    match name.rfind('.') {
        Some(position) => &name[position + 1..],
        None => "",
    }
}

fn main() {
    for name in ["photo.png", "archive.tar.gz", "README"] {
        println!("{name} -> '{}'", file_extension(name));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn simple_extension() {
        assert_eq!(file_extension("photo.png"), "png");
    }

    #[test]
    fn last_dot_wins() {
        assert_eq!(file_extension("archive.tar.gz"), "gz");
    }

    #[test]
    fn no_dot_means_empty() {
        assert_eq!(file_extension("README"), "");
    }

    #[test]
    fn trailing_dot_means_empty() {
        assert_eq!(file_extension("odd."), "");
    }
}
```

```text
photo.png -> 'png'
archive.tar.gz -> 'gz'
README -> ''
```

Why it works: the result is a slice of `name`, so no new text is allocated. A dot is one byte (ASCII), so `position + 1` is always a valid boundary. For `"odd."` the slice `[4..]` is empty.

</details>

### Exercise 4 — Join with a dash (medium)

Write `fn join_with_dash(a: &str, b: &str) -> String` that returns `a-b`. Call it with a `String` borrowed and a literal, and show that the `String` is still usable. Add two tests.

<details>
<summary>Hint</summary>

`format!("{a}-{b}")` builds a new `String`. In `main`, pass `&first` for the `String`.

</details>

<details>
<summary>Answer</summary>

```rust
fn join_with_dash(a: &str, b: &str) -> String {
    format!("{a}-{b}")
}

fn main() {
    let first = String::from("rust");
    let joined = join_with_dash(&first, "lang");
    println!("{joined}");
    println!("{first} is still mine");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn joins_two_parts() {
        assert_eq!(join_with_dash("a", "b"), "a-b");
    }

    #[test]
    fn works_with_string_borrows() {
        let x = String::from("left");
        let y = String::from("right");
        assert_eq!(join_with_dash(&x, &y), "left-right");
    }
}
```

```text
rust-lang
rust is still mine
```

Why it works: both parameters are `&str`, so the function accepts literals and borrowed `String`s. `format!` only borrows, so nothing is moved.

</details>

### Exercise 5 — The second word (hard)

Write `fn second_word(text: &str) -> &str` that returns the second word of the text (words are separated by any whitespace; extra spaces do not matter), or `""` if there are fewer than two words. The result must be a slice of the input. Add tests for normal text, extra spaces, and too few words. Then, in `main`, keep the result of `second_word(&line)` in a variable and try to call `line.clear()` before using it — read the error.

<details>
<summary>Hint</summary>

Loop over `text.split_whitespace()` with a counter. When the counter reaches 2, `return word;`. After the loop, return `""`. Each `word` is already a slice of `text`.

</details>

<details>
<summary>Answer</summary>

```rust
fn second_word(text: &str) -> &str {
    let mut position = 0;
    for word in text.split_whitespace() {
        position += 1;
        if position == 2 {
            return word;
        }
    }
    ""
}

fn main() {
    println!("'{}'", second_word("one two three"));
    println!("'{}'", second_word("alone"));
    let line = String::from("  borrowed   slices  ");
    let word = second_word(&line);
    println!("'{word}' (borrowed from line: {line:?})");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_second_word() {
        assert_eq!(second_word("one two three"), "two");
    }

    #[test]
    fn extra_spaces_do_not_matter() {
        assert_eq!(second_word("  a    b  "), "b");
    }

    #[test]
    fn too_few_words_gives_empty() {
        assert_eq!(second_word(""), "");
        assert_eq!(second_word("single"), "");
    }
}
```

```text
'two'
''
'slices' (borrowed from line: "  borrowed   slices  ")
```

If `line.clear()` is called while `word` is still used later, the compiler reports `E0502` (mutable borrow while an immutable borrow is alive).

Why it works: `split_whitespace` skips extra spaces and hands out slices of the original text. The empty string `""` is a `&'static str` literal, which can be returned too, because it lives as long as the program (Lesson 05.3 explains why this is allowed).

</details>

## Quiz

1. What is a string slice, and how big is a `&str` on a 64-bit machine?

<details>
<summary>Answer</summary>

A borrowed view of part of a UTF-8 text, made of a pointer and a length. It is 16 bytes.

</details>

2. Why is returning a slice safer than returning an index?

<details>
<summary>Answer</summary>

A slice is a borrow of the original string, so the compiler stops you from changing or dropping the string while the slice is in use. An index is just a number and can go stale silently.

</details>

3. Why does `"привет".len()` return 12?

<details>
<summary>Answer</summary>

`len()` counts bytes. Each Cyrillic letter takes 2 bytes in UTF-8, and there are 6 letters.

</details>

4. Why can you not write `s[0]` on a `String`?

<details>
<summary>Answer</summary>

Characters have different byte lengths in UTF-8, so a single number is ambiguous (byte or character?) and a byte position may fall inside a character. Rust forbids it; use `chars()`, `bytes()` or a range.

</details>

5. Which parameter type should a function that only reads text use, and why?

<details>
<summary>Answer</summary>

`&str`. It accepts string literals, `&String` (by deref coercion) and slices.

</details>

6. What does `s1 + &s2` do to `s1` and `s2`?

<details>
<summary>Answer</summary>

It moves `s1` (it cannot be used afterwards) and only borrows `s2`. The result is a new owner of the combined text.

</details>

7. What type is the string literal `"hello"`?

<details>
<summary>Answer</summary>

`&str` — a borrowed slice of text stored inside the program.

</details>

## Summary

- A slice (`&str`, `&[T]`) is a borrowed view of a continuous part of a string or sequence: a pointer plus a length. It owns nothing, and the borrowing rules apply to it.
- Returning a slice instead of an index lets the compiler prevent stale-index bugs.
- Strings are UTF-8: `len()` counts bytes, `chars().count()` counts characters, slice ranges are in bytes and must lie on char boundaries, and `s[0]` is not allowed.
- `String` owns and can grow; `&str` borrows and is read-only. Literals are `&str`.
- Take `&str` (not `&String`) and `&[T]` (not `&Vec<T>`) in parameters: they accept more.
- `s1 + &s2` moves `s1`; use `format!` to combine without moving anything.
- `trim`, `split_whitespace`, `lines` return cheap slices; `to_lowercase` and `replace` build new `String`s; `find` returns an `Option`.

## Sources and further reading

- [The Rust Programming Language — 4.3 The Slice Type](https://doc.rust-lang.org/book/ch04-03-slices.html) — the official chapter on slices.
- [The Rust Programming Language — 8.2 Storing UTF-8 Encoded Text with Strings](https://doc.rust-lang.org/book/ch08-02-strings.html) — why indexing is forbidden, bytes versus characters.
- [std::primitive::str](https://doc.rust-lang.org/std/primitive.str.html) — all methods of `&str`.
- [std::string::String](https://doc.rust-lang.org/std/string/struct.String.html) — all methods of `String`.
- [std::primitive::slice](https://doc.rust-lang.org/std/primitive.slice.html) — all methods of `&[T]`.

---

[🏠 Start](../START-HERE.md) · [← 03.5 Mutable references and the borrowing rules](05-mutable-references-and-the-borrow-rules.md) · [📋 Phase 03](README.md) · [03.7 Fixing common ownership errors →](07-fixing-common-ownership-errors.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
