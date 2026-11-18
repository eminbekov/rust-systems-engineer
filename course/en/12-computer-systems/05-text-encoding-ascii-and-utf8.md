<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 12.4 Bitwise operations and bit tricks](04-bitwise-operations.md) · [📋 Phase 12](README.md) · [12.6 How the CPU runs instructions →](06-how-the-cpu-runs-instructions.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/05-text-encoding-ascii-and-utf8.md) <!-- nav -->

# Lesson 12.5 — Text encoding: ASCII, Unicode and UTF-8

> **Phase 12 — Computer systems** · Lesson 5 of 15 · ⏱ about 120 minutes

## What you will learn

- How the ASCII table is organised, and the bit tricks behind digits and upper/lower case.
- What a Unicode code point is, which numbers are valid, and why a `char` is 4 bytes.
- The exact bit patterns of UTF-8, so that you can encode and decode a character by hand and in code.
- What happens with invalid bytes: errors, `from_utf8_lossy` and the replacement character.
- Why one "letter" on the screen can be several `char`s (grapheme clusters), and where UTF-16 still appears.

## Before you start

- You finished [Lesson 12.4 — Bitwise operations and bit tricks](04-bitwise-operations.md).
- [Lesson 04.8 — Strings in depth](../04-structs-enums-collections-errors/08-strings-in-depth.md) showed that a `String` is UTF-8, that one character takes 1 to 4 bytes, and that `len()` counts bytes while `chars()` counts characters. This lesson opens up those bytes.
- Create the package for this lesson: `cargo new code/p12-text`.

## New words

| Word | Meaning |
|---|---|
| **character encoding** | A rule that turns characters into bytes and back. ASCII and UTF-8 are encodings. |
| **ASCII** (American Standard Code for Information Interchange) | The 1960s table of 128 characters (English letters, digits, punctuation, control codes), numbered 0 to 127, 7 bits each. |
| **control character** | An ASCII code that is not a visible symbol but an instruction, such as newline (`0x0a`) or tab (`0x09`). |
| **Unicode** | The world-wide table of characters. It gives every character a number, its *code point*. |
| **code point** | The number of a character in Unicode, written `U+` and hex digits: `U+044F` is `я`. |
| **Unicode scalar value** | A code point that may appear in text: 0 to `0x10FFFF` except the surrogate range `0xD800`–`0xDFFF`. A Rust `char` is exactly one of these. |
| **surrogate** | A code point from `0xD800` to `0xDFFF`, reserved for UTF-16; it is never a character by itself. |
| **UTF-8** (Unicode Transformation Format, 8-bit) | The encoding that stores a code point in 1 to 4 bytes. Rust's `str` and `String` always contain valid UTF-8. |
| **UTF-16** | An encoding that uses 16-bit units: one unit for most characters, two (a *surrogate pair*) for the others. Windows and some file formats use it. |
| **continuation byte** | A byte of a multi-byte UTF-8 character after the first one. It always looks like `10xxxxxx`. |
| **overlong encoding** | A forbidden way to write a character with more bytes than necessary. Valid UTF-8 never contains it. |
| **replacement character** | `U+FFFD`, shown as `�`. It stands for bytes that could not be decoded. |
| **grapheme cluster** | What a reader sees as one character. It can be several code points, for example a letter plus an accent. |

## 1. ASCII: the first 128 numbers

ASCII uses 7 bits, so it has 128 codes. The table is not random: it is arranged so that simple bit operations do useful things.

```rust
fn main() {
    for b in [b'\0', b'\t', b'\n', b'\r', b' ', b'0', b'9', b'A', b'Z', b'a', b'z', b'~'] {
        println!("{:>3} = {b:#04x} = {b:07b}  {:?}", b, b as char);
    }
    let digit = b'7' - b'0';
    println!("'7' - '0' = {digit}");
    println!("'a' ^ 0x20 = {}", (b'a' ^ 0x20) as char);
    println!("'Q' | 0x20 = {}", (b'Q' | 0x20) as char);
    println!("{}", "Hello, World".to_ascii_uppercase());
    println!("{}", "OK".eq_ignore_ascii_case("ok"));
}
```

**What happens:**

1. `b'A'` is a byte literal: the ASCII code of `A` as a `u8` (Lesson 02.2). The loop prints each code in decimal, hex and 7-bit binary, and `{:?}` shows the character with escapes for invisible ones.
2. Codes 0 to 31 are **control characters**: `\0` (zero byte), `\t` (tab), `\n` (newline, "line feed"), `\r` (carriage return). Code 32 is the space, 127 is another control code; everything between is visible.
3. Digits start at `0x30`, so the value of a digit is its code minus `b'0'`. Every number parser does this.
4. Upper case starts at `0x41`, lower case at `0x61`. They differ only in **bit 5** (`0x20`). XOR (exclusive OR, Lesson 12.4) with `0x20` swaps the case, OR sets lower case, AND with `!0x20` sets upper case. Only for ASCII letters, of course.
5. The standard library has these as methods: `to_ascii_uppercase`, `eq_ignore_ascii_case`, `is_ascii_digit` and others. They leave non-ASCII characters unchanged.

```text
  0 = 0x00 = 0000000  '\0'
  9 = 0x09 = 0001001  '\t'
 10 = 0x0a = 0001010  '\n'
 13 = 0x0d = 0001101  '\r'
 32 = 0x20 = 0100000  ' '
 48 = 0x30 = 0110000  '0'
 57 = 0x39 = 0111001  '9'
 65 = 0x41 = 1000001  'A'
 90 = 0x5a = 1011010  'Z'
 97 = 0x61 = 1100001  'a'
122 = 0x7a = 1111010  'z'
126 = 0x7e = 1111110  '~'
'7' - '0' = 7
'a' ^ 0x20 = A
'Q' | 0x20 = q
HELLO, WORLD
true
```

> 💡 **Tip:** Lines in Linux and macOS files end with `\n`. Files from Windows often end lines with `\r\n`. If you see a stray `\r` at the end of your parsed lines, that is the reason; `str::lines()` removes both.

## 2. Unicode code points and `char`

Unicode gives every character a number, the **code point**. The range is 0 to `0x10FFFF` (about 1.1 million numbers). The first 128 are exactly ASCII. A Rust `char` holds one code point, in 4 bytes, because the largest one needs 21 bits:

```rust
fn main() {
    for c in ['A', 'я', '€', '世', '😀'] {
        println!("{c}  U+{:04X}  as u32 = {:>6}  utf-8 bytes = {}", c as u32, c as u32, c.len_utf8());
    }
    println!("{:?}", char::from_u32(0x44F));
    println!("{:?}", char::from_u32(0xD800));
    println!("{:?}", char::from_u32(0x110000));
    println!("{}", '\u{1F600}');
    println!("char is {} bytes", std::mem::size_of::<char>());
}
```

**What happens:**

1. `c as u32` gives the code point. It is printed in the usual `U+` form with at least four hex digits.
2. `len_utf8()` says how many bytes this character needs in UTF-8.
3. `char::from_u32` builds a `char` from a number, but only if the number is a **Unicode scalar value**. `0xD800` is a **surrogate** (reserved for UTF-16) and `0x110000` is too large, so both give `None`. That is why a `char` can never hold an invalid value.
4. `'\u{1F600}'` writes a character by its code point inside a literal.

```text
A  U+0041  as u32 =     65  utf-8 bytes = 1
я  U+044F  as u32 =   1103  utf-8 bytes = 2
€  U+20AC  as u32 =   8364  utf-8 bytes = 3
世  U+4E16  as u32 =  19990  utf-8 bytes = 3
😀  U+1F600  as u32 = 128512  utf-8 bytes = 4
Some('я')
None
None
😀
char is 4 bytes
```

Storing every character in 4 bytes would be simple but wasteful: English text would become four times bigger. UTF-8 is the clever compromise.

## 3. UTF-8, bit by bit

UTF-8 cuts the bits of the code point into pieces and puts them into bytes with fixed "headers". The header of the first byte says how many bytes the character has:

```text
code points          bits   byte 1     byte 2     byte 3     byte 4
U+0000  .. U+007F      7    0xxxxxxx
U+0080  .. U+07FF     11    110xxxxx   10xxxxxx
U+0800  .. U+FFFF     16    1110xxxx   10xxxxxx   10xxxxxx
U+10000 .. U+10FFFF   21    11110xxx   10xxxxxx   10xxxxxx   10xxxxxx
```

Let us encode `я` (U+044F) by hand. `0x44F` in binary is `100 0100 1111`: 11 bits, so 2 bytes. Fill the `x` places from the right: the lowest 6 bits `001111` go into byte 2, the remaining 5 bits `10001` into byte 1:

```text
code point 0x44F:          10001 001111
byte 1:   110 + 10001  =   11010001  = 0xd1
byte 2:    10 + 001111 =   10001111  = 0x8f
```

That is exactly `d1 8f`, the bytes you saw in Lesson 04.8. Let the computer show the patterns for more characters:

```rust
fn main() {
    for c in ['a', 'я', '€', '😀'] {
        let mut buf = [0u8; 4];
        let bytes = c.encode_utf8(&mut buf).as_bytes();
        let bits: Vec<String> = bytes.iter().map(|b| format!("{b:08b}")).collect();
        println!("{c} U+{:05X} {:>21b} -> {}", c as u32, c as u32, bits.join(" "));
    }
}
```

**What happens:**

1. `encode_utf8` writes the UTF-8 bytes of a `char` into a small buffer (4 bytes is always enough) and returns them as a `&str`; `as_bytes()` gives the bytes.
2. Each byte is printed in binary. Compare the headers (`0`, `110`, `1110`, `11110`, `10`) with the table above, and find the bits of the code point in the `x` places.

```text
a U+00061               1100001 -> 01100001
я U+0044F           10001001111 -> 11010001 10001111
€ U+020AC        10000010101100 -> 11100010 10000010 10101100
😀 U+1F600     11111011000000000 -> 11110000 10011111 10011000 10000000
```

This design has four very useful properties:

- **ASCII stays ASCII.** Every ASCII text is already valid UTF-8, byte for byte.
- **No confusion with ASCII.** All bytes of a multi-byte character are ≥ `0x80`, so searching UTF-8 text for `/` or `\n` byte by byte can never find a false match inside another character.
- **Self-synchronising.** Continuation bytes always start with `10`. From any position you can find the start of the current character by stepping back over at most three `10xxxxxx` bytes. This is what `str::is_char_boundary` checks, and why slicing a `&str` in the middle of a character panics (Lesson 04.8).
- **Sorting bytes sorts code points.** Comparing UTF-8 bytes gives the same order as comparing the code points.

> ⚡ **Performance:** Because of these properties, many string operations work on bytes without decoding characters. For example, `str::find` with an ASCII pattern can use a fast byte search. When your input is pure ASCII (log files, the headers of HTTP, the HyperText Transfer Protocol of the web), working with `&[u8]` and the `u8::is_ascii_*` methods avoids all decoding.

## 4. Invalid bytes

Not every sequence of bytes is valid UTF-8. A `&str` in Rust is **guaranteed** to be valid, so the conversion from bytes must check:

```rust
fn main() {
    let good = vec![0x68, 0x69, 0xd1, 0x8f];
    let cut = vec![0x68, 0x69, 0xd1];
    let overlong = vec![0xc0, 0xaf];
    let surrogate = vec![0xed, 0xa0, 0x80];
    for bytes in [&good, &cut, &overlong, &surrogate] {
        match std::str::from_utf8(bytes) {
            Ok(text) => println!("{bytes:02x?} ok: {text:?}"),
            Err(e) => println!("{bytes:02x?} error: {e} (valid up to {})", e.valid_up_to()),
        }
    }
    println!("lossy: {}", String::from_utf8_lossy(&cut));
}
```

**What happens:**

1. `std::str::from_utf8` checks the bytes and returns `Ok(&str)` without copying, or an error that says where the valid part ends (`valid_up_to`).
2. `cut` ends in the middle of `я`: the first byte `d1` promises one continuation byte that never comes.
3. `c0 af` is an **overlong encoding** of `/` (U+002F) in two bytes. It is forbidden: every character has exactly one valid encoding.
4. `ed a0 80` would be the surrogate U+D800, which is not a character, so it is rejected too.
5. `String::from_utf8_lossy` never fails: it replaces each bad part with the **replacement character** `U+FFFD` (`�`).

```text
[68, 69, d1, 8f] ok: "hiя"
[68, 69, d1] error: incomplete utf-8 byte sequence from index 2 (valid up to 2)
[c0, af] error: invalid utf-8 sequence of 1 bytes from index 0 (valid up to 0)
[ed, a0, 80] error: invalid utf-8 sequence of 1 bytes from index 0 (valid up to 0)
lossy: hi�
```

> 🔒 **Security:** Overlong encodings used to be a real attack: old programs checked a path for `../` and then decoded `c0 af` into another `/`, so `..` + `c0 af` slipped through the check. Rust's strict validation rejects such bytes. Two other text tricks still matter: **look-alike characters** (the Cyrillic `а` U+0430 looks like the Latin `a` U+0061, so `pаypal` may not be `paypal`), and **invisible characters** (U+200B zero width space). Compare identifiers byte for byte, and do not trust what text *looks* like.

## 5. More than one `char` per letter

A `char` is one code point, but what a person sees as one letter (a **grapheme cluster**) can be several:

```rust
fn main() {
    let composed = "\u{e9}";
    let decomposed = "e\u{301}";
    let flag = "\u{1F1EF}\u{1F1F5}";
    for (name, s) in [("composed", composed), ("decomposed", decomposed), ("flag", flag)] {
        println!("{name:<10} {s}  chars = {}  bytes = {}  utf-16 units = {}", s.chars().count(), s.len(), s.encode_utf16().count());
    }
    println!("same text? {}", composed == decomposed);
    let reversed: String = decomposed.chars().rev().collect();
    println!("reversed by chars: {reversed:?}");
}
```

**What happens:**

1. `é` can be one code point (U+00E9) or two: `e` plus U+0301, a *combining accent* that attaches to the letter before it. Both look the same on screen.
2. A flag is two "regional indicator" code points (here J and P, for Japan). The terminal draws them as one picture.
3. `encode_utf16()` turns the text into 16-bit **UTF-16** units. Characters above U+FFFF, like each regional indicator, need two units (a surrogate pair), so the flag is 4 units.
4. The two forms of `é` are **not equal** as strings: `==` compares bytes. Making them equal is called *normalisation* (the two forms are called NFC, Normalization Form C for "composed", and NFD, Normalization Form D for "decomposed"), and the crate `unicode-normalization` does it.
5. Reversing by `char` puts the accent before the `e`, which breaks the letter. The crate `unicode-segmentation` splits text into grapheme clusters when you need "letters as people see them".

```text
composed   é  chars = 1  bytes = 2  utf-16 units = 1
decomposed é  chars = 2  bytes = 3  utf-16 units = 2
flag       🇯🇵  chars = 2  bytes = 8  utf-16 units = 4
same text? false
reversed by chars: "\u{301}e"
```

So "the length of a string" has at least four answers: bytes (`len`), code points (`chars().count()`), UTF-16 units (what some other systems count) and grapheme clusters (what people count). Choose the one your task really needs. For limits on input size, bytes are usually right: they are what memory, files and networks measure.

> 🧪 **Testing:** Every text function deserves tests with: an empty string, pure ASCII, Cyrillic or another 2-byte script, a 3-byte character (`€`), a 4-byte emoji, a combining accent, and invalid bytes (for functions that take `&[u8]`). Exercise 2 even tests all 1.1 million characters.

## Common mistakes

> ⚠️ **Common mistake: reading a file that is not UTF-8 with `read_to_string`.** Old text files are often in another encoding, such as Latin-1, where `é` is the single byte `0xe9`:

```rust
use std::fs;

fn main() {
    fs::write("latin1.txt", [b'c', b'a', b'f', 0xe9]).unwrap();
    match fs::read_to_string("latin1.txt") {
        Ok(text) => println!("text: {text}"),
        Err(e) => println!("error: {e}"),
    }
    let bytes = fs::read("latin1.txt").unwrap();
    println!("bytes: {bytes:02x?}");
    println!("lossy: {}", String::from_utf8_lossy(&bytes));
}
```

```text
error: stream did not contain valid UTF-8
bytes: [63, 61, 66, e9]
lossy: caf�
```

`read_to_string` refuses invalid UTF-8. Read bytes with `fs::read` when you do not control the input, then decide: reject it, use `from_utf8_lossy`, or convert from the real encoding with a crate such as `encoding_rs`. (Delete `latin1.txt` afterwards.)

> ⚠️ **Common mistake: ASCII tricks on non-ASCII text.** `b ^ 0x20` swaps the case of `a`–`z` and `A`–`Z` only. Applied to the bytes of `я` it destroys the UTF-8. Use `to_ascii_uppercase` (which skips non-ASCII bytes) or `to_uppercase` (which knows all of Unicode).

> ⚠️ **Common mistake: `chars().count()` as "the number of letters".** It counts code points, so `"e\u{301}"` gives 2. Use grapheme clusters when the visible length matters (for example, aligning text in a terminal).

## Try it yourself

1. Print `"Ünïcödé".to_uppercase()` and `"Ünïcödé".to_ascii_uppercase()` and explain the difference.
2. Take the string `"aя€😀"` and print, for every byte, its index, its hex value and whether `is_char_boundary(index)` is `true`. Check that the `true` lines are exactly the bytes that do **not** start with `10`.
3. Print `"\u{FEFF}hi".as_bytes()`. The first three bytes `ef bb bf` are the **BOM** (Byte Order Mark), which some Windows editors put at the start of UTF-8 files. If a parser fails on the first word of such a file, strip it with `trim_start_matches('\u{FEFF}')`.

## Exercises

### Exercise 1 — Encode by hand (easy)

Encode `é` (U+00E9) and `€` (U+20AC) into UTF-8 on paper, using the table in Section 3. Then check with `as_bytes()`.

<details>
<summary>Hint</summary>

`0xE9` = `1110 1001` needs 8 bits → the 2-byte form `110xxxxx 10xxxxxx` with 11 places; pad with zeros on the left: `000 1110 1001`. `0x20AC` needs 14 bits → the 3-byte form with 16 places.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    println!("{:02x?} {:02x?}", "é".as_bytes(), "€".as_bytes());
}
```

```text
[c3, a9] [e2, 82, ac]
```

Why it works: `00011 101001` → `110 00011` = `c3`, `10 101001` = `a9`. For `€`: `0010 000010 101100` → `1110 0010` = `e2`, `10 000010` = `82`, `10 101100` = `ac`.

</details>

### Exercise 2 — Your own UTF-8 encoder (medium)

In `src/lib.rs` of `code/p12-text`, write `pub fn encode_utf8(cp: u32) -> Option<Vec<u8>>` that returns the UTF-8 bytes of a code point, or `None` for surrogates and values above `0x10FFFF`. Write a test that compares your function with `char::encode_utf8` for **every** `char` (`char::MIN..=char::MAX` is a range you can loop over), and a test for the invalid values.

<details>
<summary>Hint</summary>

Use `match` with ranges: `0..=0x7f`, `0x80..=0x7ff`, `0xd800..=0xdfff` (→ `None`; put it before the 3-byte range), `0x800..=0xffff`, `0x1_0000..=0x10_ffff`, `_`. For each byte, take the right 6-bit piece with "shift, then mask with `0x3f`" and OR it with the header, like `0b1000_0000 | (cp & 0x3f) as u8`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Encodes one Unicode code point as UTF-8, or returns None if it is not a valid scalar value.
pub fn encode_utf8(cp: u32) -> Option<Vec<u8>> {
    match cp {
        0..=0x7f => Some(vec![cp as u8]),
        0x80..=0x7ff => Some(vec![0b1100_0000 | (cp >> 6) as u8, 0b1000_0000 | (cp & 0x3f) as u8]),
        0xd800..=0xdfff => None,
        0x800..=0xffff => Some(vec![
            0b1110_0000 | (cp >> 12) as u8,
            0b1000_0000 | ((cp >> 6) & 0x3f) as u8,
            0b1000_0000 | (cp & 0x3f) as u8,
        ]),
        0x1_0000..=0x10_ffff => Some(vec![
            0b1111_0000 | (cp >> 18) as u8,
            0b1000_0000 | ((cp >> 12) & 0x3f) as u8,
            0b1000_0000 | ((cp >> 6) & 0x3f) as u8,
            0b1000_0000 | (cp & 0x3f) as u8,
        ]),
        _ => None,
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn matches_std_for_every_char() {
        for c in char::MIN..=char::MAX {
            let mut buf = [0u8; 4];
            let expected = c.encode_utf8(&mut buf).as_bytes().to_vec();
            assert_eq!(encode_utf8(c as u32), Some(expected), "char U+{:04X}", c as u32);
        }
    }

    #[test]
    fn rejects_invalid_code_points() {
        assert_eq!(encode_utf8(0xd800), None);
        assert_eq!(encode_utf8(0xdfff), None);
        assert_eq!(encode_utf8(0x11_0000), None);
    }
}
```

Why it works: the first byte takes the high bits (`cp >> 6`, `>> 12` or `>> 18`); its header leaves exactly the right number of places, so no mask is needed there. Each continuation byte takes the next 6 bits. A `match` arm is chosen top-down, so the surrogate arm wins over the 3-byte arm. The range `char::MIN..=char::MAX` skips the surrogates by itself, so the test checks all 1,112,064 valid characters in a fraction of a second.

</details>

### Exercise 3 — Count characters without decoding (medium)

Write `pub fn count_chars(text: &str) -> usize` that counts the characters by looking only at bytes: count every byte that is **not** a continuation byte. Test it against `text.chars().count()` for `""`, `"hello"`, `"привет"`, `"a€😀"` and `"e\u{301}"`.

<details>
<summary>Hint</summary>

A continuation byte matches `10xxxxxx`: `b & 0b1100_0000 == 0b1000_0000`. Use `text.bytes().filter(…).count()`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Counts the characters of UTF-8 text by counting the bytes that are not continuation bytes.
pub fn count_chars(text: &str) -> usize {
    text.bytes().filter(|&b| b & 0b1100_0000 != 0b1000_0000).count()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_like_chars() {
        for text in ["", "hello", "привет", "a€😀", "e\u{301}"] {
            assert_eq!(count_chars(text), text.chars().count(), "text {text:?}");
        }
    }
}
```

Why it works: every character has exactly one first byte, and only first bytes are not of the form `10xxxxxx`. The function is correct only because a `&str` is guaranteed to be valid UTF-8; on arbitrary bytes it would count garbage.

</details>

### Exercise 4 — A strict UTF-8 decoder (hard)

Write `pub fn decode_first(bytes: &[u8]) -> Option<(char, usize)>` that decodes the first character of `bytes` and returns it with its length, or `None` if the bytes do not start with valid UTF-8. It must reject: an empty slice, a stray continuation byte, a cut sequence, overlong encodings, surrogates and values above `0x10FFFF`. Tests: decode every `char` from its own encoding; and the bad inputs `[]`, `[80]`, `[d1]`, `[c0 af]`, `[e0 80 af]`, `[ed a0 80]`, `[f4 90 80 80]`.

<details>
<summary>Hint</summary>

From the first byte, find the length (1–4), the payload bits of the first byte (masks `0x1f`, `0x0f`, `0x07`) and the smallest code point that really needs this length (`0x80`, `0x800`, `0x1_0000`). Check that enough bytes follow and that each is `10xxxxxx`; collect the bits with `cp = (cp << 6) | (b & 0x3f) as u32`. A result below the minimum is overlong. `char::from_u32` rejects the rest.

</details>

<details>
<summary>Answer</summary>

```rust
/// Decodes the first character of `bytes`. Returns the char and its length in bytes,
/// or None if the bytes do not start with a valid UTF-8 sequence.
pub fn decode_first(bytes: &[u8]) -> Option<(char, usize)> {
    let first = *bytes.first()?;
    let (len, mut cp, min) = match first {
        0x00..=0x7f => return Some((first as char, 1)),
        0xc0..=0xdf => (2, (first & 0x1f) as u32, 0x80),
        0xe0..=0xef => (3, (first & 0x0f) as u32, 0x800),
        0xf0..=0xf7 => (4, (first & 0x07) as u32, 0x1_0000),
        _ => return None,
    };
    if bytes.len() < len {
        return None;
    }
    for &b in &bytes[1..len] {
        if b & 0b1100_0000 != 0b1000_0000 {
            return None;
        }
        cp = (cp << 6) | (b & 0x3f) as u32;
    }
    if cp < min {
        return None; // overlong encoding
    }
    char::from_u32(cp).map(|c| (c, len)) // rejects surrogates and values above 0x10FFFF
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn decodes_every_char() {
        for c in char::MIN..=char::MAX {
            let mut buf = [0u8; 4];
            let bytes = c.encode_utf8(&mut buf).as_bytes();
            assert_eq!(decode_first(bytes), Some((c, bytes.len())));
        }
    }

    #[test]
    fn rejects_bad_sequences() {
        let bad: [&[u8]; 7] = [
            &[],
            &[0x80],
            &[0xd1],
            &[0xc0, 0xaf],
            &[0xe0, 0x80, 0xaf],
            &[0xed, 0xa0, 0x80],
            &[0xf4, 0x90, 0x80, 0x80],
        ];
        for bytes in bad {
            assert_eq!(decode_first(bytes), None, "bytes {bytes:02x?}");
            assert!(std::str::from_utf8(bytes).is_err() || bytes.is_empty());
        }
    }
}
```

Why it works: the first byte alone decides the length; the bytes `0x80`–`0xbf` (continuations) and `0xf8`–`0xff` (never used) are rejected at once. The minimum check catches overlong forms such as `c0 af` (which would decode to `0x2f`, below `0x80`). `f4 90 80 80` decodes to `0x110000`, which `char::from_u32` refuses. The second assertion uses the standard library as an oracle: it must agree that these inputs are invalid.

</details>

## Quiz

1. Why is the value of the ASCII digit `b` equal to `b - b'0'`?

<details>
<summary>Answer</summary>

The digits `'0'` to `'9'` have the consecutive codes `0x30` to `0x39`, so subtracting the code of `'0'` gives 0 to 9.

</details>

2. Which single bit separates `'A'` from `'a'` in ASCII?

<details>
<summary>Answer</summary>

Bit 5 (`0x20`): `'A'` is `0x41`, `'a'` is `0x61`.

</details>

3. Why does `char::from_u32(0xD800)` return `None`?

<details>
<summary>Answer</summary>

`0xD800` is a surrogate code point, reserved for UTF-16 pairs. It is not a Unicode scalar value, and a `char` may only hold scalar values.

</details>

4. How many bytes does U+20AC (`€`) take in UTF-8, and what does the first byte start with?

<details>
<summary>Answer</summary>

Three bytes; the first starts with `1110` (`e2`).

</details>

5. Why can you search UTF-8 text for the byte `b'/'` without decoding it?

<details>
<summary>Answer</summary>

All bytes of multi-byte characters are `0x80` or larger, so the ASCII byte `0x2f` can only appear as the character `/` itself.

</details>

6. What does `String::from_utf8_lossy` do with invalid bytes?

<details>
<summary>Answer</summary>

It replaces each invalid part with the replacement character U+FFFD (`�`) and never fails.

</details>

7. `"e\u{301}"` and `"\u{e9}"` look the same. Why is `==` false?

<details>
<summary>Answer</summary>

They are different code points and therefore different bytes (`65 cc 81` against `c3 a9`). String equality compares bytes; normalisation would make them equal.

</details>

## Summary

- ASCII is 7 bits: control codes 0–31, digits from `0x30`, upper case from `0x41`, lower case from `0x61`; bit 5 separates the cases.
- Unicode numbers characters with code points from 0 to `0x10FFFF`; a `char` is a scalar value (no surrogates) in 4 bytes.
- UTF-8 uses headers `0`, `110`, `1110`, `11110` for first bytes and `10` for continuation bytes. It is ASCII-compatible and self-synchronising.
- A `&str` is always valid UTF-8. Converting bytes needs a check: `str::from_utf8` (error), `String::from_utf8_lossy` (`�`); overlong forms and surrogates are rejected.
- One visible letter can be several code points; length can mean bytes, code points, UTF-16 units or grapheme clusters.
- Test text code with ASCII, multi-byte characters, emoji, combining marks and invalid bytes; do not trust how text looks.

## Sources and further reading

- [Rust standard library: `char`](https://doc.rust-lang.org/std/primitive.char.html) — `from_u32`, `encode_utf8`, `len_utf8` and the definition of a Unicode scalar value.
- [Rust standard library: `std::str::from_utf8`](https://doc.rust-lang.org/std/str/fn.from_utf8.html) — validation and the `Utf8Error` type.
- [RFC 3629: UTF-8](https://www.rfc-editor.org/rfc/rfc3629) — the official definition of UTF-8, including why overlong forms are forbidden. (RFC means Request for Comments, the name of Internet standards documents.)
- [The Unicode Standard](https://www.unicode.org/standard/standard.html) — the home of Unicode, with code charts for every script.
- [Unicode Standard Annex #29: Text Segmentation](https://www.unicode.org/reports/tr29/) — the rules for grapheme clusters.

---

[🏠 Start](../START-HERE.md) · [← 12.4 Bitwise operations and bit tricks](04-bitwise-operations.md) · [📋 Phase 12](README.md) · [12.6 How the CPU runs instructions →](06-how-the-cpu-runs-instructions.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
