<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.3 The OWASP Top 10](03-the-owasp-top-10.md) · [📋 Phase 10](README.md) · [10.5 Injection attacks: SQL, commands and paths →](05-injection-attacks.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/04-input-validation.md) <!-- nav -->

# Lesson 10.4 — Input validation: never trust input

> **Phase 10 — Security fundamentals** · Lesson 4 of 15 · ⏱ about 135 minutes

## What you will learn

- The difference between validating, sanitizing and escaping, and why an allow-list beats a deny-list.
- The right order of checks: size, encoding, canonical form, syntax, meaning.
- How to "parse, don't validate": turn raw text into a type that can only hold good values.
- How to validate numbers, avoid overflow, and use a regular expression safely.
- How to test your validation with hostile input.

## Before you start

- You finished [Lesson 10.3 — The OWASP Top 10](03-the-owasp-top-10.md).
- You know newtypes and making invalid states impossible ([Lesson 08.5](../08-software-design/05-type-driven-design.md)), `Result` and custom errors ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)), and `FromStr` parsing.
- Create the package: `cargo new --lib code/p10-validate`, run `cargo add regex@1.13` inside it, then `mkdir src/bin`.

## New words

| Word | Meaning |
|---|---|
| **validation** | Checking that input has the right form and meaning, and **rejecting** it when it does not. |
| **sanitization** | Changing input to remove the dangerous parts (for example deleting characters). Easy to get wrong; prefer rejecting. |
| **escaping** (or *encoding*) | Writing a value so that the place that receives it treats it as plain data, not as a command (for example `<` becomes `&lt;` in HTML). Lesson 10.5. |
| **allow-list** | A list of what **is** allowed; everything else is rejected. Also called a whitelist. |
| **deny-list** | A list of what is **not** allowed; everything else passes. Also called a blacklist. Weak: attackers find what you forgot. |
| **canonical form** | The one standard way to write a value (lowercase, no extra spaces, one Unicode form), so equal values look equal. |
| **Unicode** | The standard that gives a number to every character of every writing system, and to emoji. In Rust a `char` is one such value. |
| **homoglyph** | A character that looks like another one, such as the Cyrillic `а` and the Latin `a`. Used to fake names. |
| **newtype** | A struct with one field that gives a new meaning to an existing type, such as `struct Username(String)`. |
| **regular expression** (regex) | A short pattern that describes a set of texts, such as `^[a-z]+$` for "only lowercase letters". |
| **ReDoS** (Regular expression Denial of Service) | An attack with a text that makes a slow regex engine run for minutes or longer. |
| **ASCII** (American Standard Code for Information Interchange) | The oldest and smallest character set: 128 characters (English letters, digits, some symbols). Each takes one byte. |
| **UTF-8** (Unicode Transformation Format, 8-bit) | The usual way to write Unicode text as bytes. ASCII characters take one byte, other characters take two to four. Rust text is always UTF-8. |
| **HTML** (HyperText Markup Language) | The language of web pages, with tags such as `<p>` and `<script>`. |
| **SQL** (Structured Query Language) | The language for asking a database to read or change data, for example `SELECT * FROM users`. Lesson 10.5 shows why it must never be built from raw input. |

## 1. Never trust input

Every value that comes from outside your program is **input**: command-line arguments, environment
variables, files, network data, form fields, even the result of another program. Rule one of this
lesson is simple: input is hostile until your code has checked it, and the check happens **at the
trust boundary** ([Lesson 10.1](01-security-mindset-and-the-cia-triad.md)), once, as early as possible.

Three words are often mixed up:

| Technique | What it does | When |
|---|---|---|
| **Validate** | Checks the input and rejects bad input | Always, at the boundary |
| **Sanitize** | Rewrites the input to remove bad parts | Rarely; only when rejecting is impossible |
| **Escape** | Makes a value safe for one specific destination (HTML, SQL, shell) | Where the value is **used**, every time ([10.5](05-injection-attacks.md)) |

Validation answers "is this a legal value?". Escaping answers "how do I put this value into that
language safely?". You need both, and neither replaces the other.

## 2. Allow-list, not deny-list

Suppose a program must accept a file name. The deny-list approach says: "reject names that contain
`;`, `|` or `&`". The attacker writes `$(whoami)`, a backtick, a newline, or a character you never
heard of. Every forgotten character is a hole.

The allow-list approach says: "accept only letters, digits, `_` and `-`, from 3 to 32 characters". The
attacker can only use what you listed, and you listed it on purpose. When the program grows, new
dangerous characters do not slip in, because new characters are rejected by default. This is
"secure by default" ([Lesson 10.1](01-security-mindset-and-the-cia-triad.md)) applied to data.

## 3. The order of the checks

Do the cheap, general checks first and the specific ones last:

1. **Size.** Is it small enough? Refuse huge input before you process it (Lesson 10.1, Section 3.3).
2. **Encoding.** Is it valid text? Rust's `String` and `&str` are always valid UTF-8 (the usual way to write
   Unicode as bytes). Bytes from outside must go through `std::str::from_utf8` or
   `String::from_utf8`, which return an error for bad bytes.
3. **Canonical form.** Convert to the standard form (for example lowercase) **before** you check or compare.
4. **Syntax.** Does it have the right shape and characters? (The allow-list.)
5. **Meaning.** Is the value reasonable? A number in range, a date in the past, a user who exists.

## 4. Parse, don't validate: a `Username` type

A function `is_valid(&str) -> bool` has a weakness: it returns `true`, and then the program goes on
with the same plain `String`. Ten functions later nobody remembers whether this string was checked. A
better design **parses** the text into a type that exists only for good values. This is the type-driven
design of Lesson 08.5, now used for security.

Here is the library `src/lib.rs`: a `Username` type with a private field, so the only way to make one is
`Username::parse`:

```rust
use std::fmt;
use std::str::FromStr;

pub const MIN_USERNAME_LEN: usize = 3;
pub const MAX_USERNAME_LEN: usize = 32;

#[derive(Debug, PartialEq, Eq)]
pub enum UsernameError {
    TooShort,
    TooLong,
    MustStartWithLetter,
    BadCharacter(char),
}

impl fmt::Display for UsernameError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            UsernameError::TooShort => {
                write!(f, "must have at least {MIN_USERNAME_LEN} characters")
            }
            UsernameError::TooLong => write!(f, "must have at most {MAX_USERNAME_LEN} characters"),
            UsernameError::MustStartWithLetter => write!(f, "must start with a letter a-z"),
            UsernameError::BadCharacter(c) => {
                write!(f, "contains a character that is not allowed: {c:?}")
            }
        }
    }
}

impl std::error::Error for UsernameError {}

/// A user name that has already been checked. The field is private,
/// so the only way to get a `Username` is through `parse`.
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
pub struct Username(String);

impl Username {
    pub fn parse(input: &str) -> Result<Username, UsernameError> {
        // 1. Size first: a cheap check before we look at every character.
        //    (`len()` counts bytes; a character is at most 4 bytes, so this is safe.)
        if input.len() > MAX_USERNAME_LEN * 4 {
            return Err(UsernameError::TooLong);
        }
        // 2. The first character.
        let first = input.chars().next().ok_or(UsernameError::TooShort)?;
        if !first.is_ascii_alphabetic() {
            return Err(UsernameError::MustStartWithLetter);
        }
        // 3. Allow-list: only these characters are accepted.
        if let Some(bad) = input
            .chars()
            .find(|c| !(c.is_ascii_alphanumeric() || *c == '_' || *c == '-'))
        {
            return Err(UsernameError::BadCharacter(bad));
        }
        // 4. Now the text is plain ASCII, so len() is the number of characters.
        if input.len() < MIN_USERNAME_LEN {
            return Err(UsernameError::TooShort);
        }
        if input.len() > MAX_USERNAME_LEN {
            return Err(UsernameError::TooLong);
        }
        // 5. Canonical form: always lowercase, so "Ana" and "ana" are the same user.
        Ok(Username(input.to_ascii_lowercase()))
    }

    pub fn as_str(&self) -> &str {
        &self.0
    }
}

impl FromStr for Username {
    type Err = UsernameError;

    fn from_str(s: &str) -> Result<Self, Self::Err> {
        Username::parse(s)
    }
}
```

**What happens:**

1. `UsernameError` lists every reason to refuse. The `Display` code gives a message that a user can
   understand. The message says *what is wrong*, and it does not reveal anything about the system.
2. `struct Username(String)` has a **private** field. Code outside the module cannot write
   `Username("../etc".to_string())`; it must call `parse`. So **any `Username` in the program is valid**.
3. Step 1 in `parse` rejects huge input at once. A `char` takes at most 4 bytes, so a valid name is
   never longer than `MAX_USERNAME_LEN * 4` bytes. This check costs nothing and stops a long string
   from reaching the loop.
4. Step 2 looks at the first character (`chars().next()` is `None` for an empty text, which becomes
   `TooShort`). `is_ascii_alphabetic` is true only for the 52 letters `a-z` and `A-Z`.
5. Step 3 is the **allow-list**: `find` returns the first character that is not an ASCII letter or digit,
   an underscore or a dash. That character goes into the error, so the message can name it.
6. Step 4 checks the length. After step 3 the text is plain ASCII, so one byte is one character and
   `len()` is the number of characters. (For text that can contain any Unicode, `len()` counts
   bytes and `chars().count()` counts characters; this difference is a classic source of bugs.)
7. Step 5 builds the **canonical form**: `to_ascii_lowercase`. `Ana` and `ana` become the same user.
8. `impl FromStr` lets you write `"ana".parse::<Username>()`.

Now a number type. Add this to the same file:

```rust
#[derive(Debug, PartialEq, Eq)]
pub enum QuantityError {
    NotANumber,
    OutOfRange,
}

/// An order quantity from 1 to 1000.
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct Quantity(u32);

impl Quantity {
    pub fn parse(text: &str) -> Result<Quantity, QuantityError> {
        let n: u32 = text.trim().parse().map_err(|_| QuantityError::NotANumber)?;
        if (1..=1000).contains(&n) {
            Ok(Quantity(n))
        } else {
            Err(QuantityError::OutOfRange)
        }
    }

    pub fn get(self) -> u32 {
        self.0
    }
}

/// Total price in cents, or `None` if the number does not fit in a `u32`.
pub fn total_cents(unit_price_cents: u32, quantity: Quantity) -> Option<u32> {
    unit_price_cents.checked_mul(quantity.get())
}
```

**What happens:**

1. `Quantity::parse` first parses into `u32`. A `u32` cannot be negative, so `-3` fails with
   `NotANumber`. `trim()` removes spaces around the text. `map_err(|_| …)` replaces the standard parse
   error with ours.
2. `(1..=1000).contains(&n)` checks the **range**: the business rule is "1 to 1000". Syntax (it is a
   number) and meaning (it is a sensible number) are two separate checks.
3. `total_cents` multiplies with `checked_mul`. Even valid quantities can produce a number that does not
   fit: `5_000_000 × 1000` is more than `u32::MAX`. Without `checked_mul`, a release build would wrap
   around silently (Lesson 10.1, Section 3.2). With it, the caller gets `None` and must handle it.

Finally a program that uses both. Save it as `src/bin/signup.rs` and run `cargo run --bin signup`:

```rust
use p10_validate::{Quantity, Username, total_cents};

fn main() {
    let names = [
        "Ana_01",
        "ab",
        "1abc",
        "ana smith",
        "ana;rm",
        "\u{430}na_smith",
        "../etc",
    ];
    for name in names {
        match Username::parse(name) {
            Ok(user) => println!("{name:<14?} accepted as {}", user.as_str()),
            Err(error) => println!("{name:<14?} rejected: {error}"),
        }
    }

    for text in ["12", " 7 ", "0", "-3", "1e3", "99999999999"] {
        match Quantity::parse(text) {
            Ok(q) => println!(
                "{text:<14?} quantity {}, total {:?}",
                q.get(),
                total_cents(2_500, q)
            ),
            Err(error) => println!("{text:<14?} rejected: {error:?}"),
        }
    }
}
```

```text
"Ana_01" accepted as ana_01
"ab" rejected: must have at least 3 characters
"1abc" rejected: must start with a letter a-z
"ana smith" rejected: contains a character that is not allowed: ' '
"ana;rm" rejected: contains a character that is not allowed: ';'
"аna_smith" rejected: must start with a letter a-z
"../etc" rejected: must start with a letter a-z
"12" quantity 12, total Some(30000)
" 7 " quantity 7, total Some(17500)
"0" rejected: OutOfRange
"-3" rejected: NotANumber
"1e3" rejected: NotANumber
"99999999999" rejected: NotANumber
```

**What happens:**

1. `{name:<14?}` prints the text with `Debug` quotes, padded to 14 characters, so you can see the exact
   text, including the text `аna_smith`. Look carefully at it: the first letter is the Cyrillic `а`
   (U+0430), not the Latin `a`. To a human the two look the same; the allow-list sees the
   difference and rejects it. A deny-list would have to know about this character.
2. `"ana smith"` and `"ana;rm"` fail on a specific character, which the message shows.
3. `"../etc"` is rejected because it does not start with a letter, so it could never reach the
   file system.
4. `"1e3"` (one thousand in scientific notation) and `"99999999999"` (too big for `u32`) are not accepted
   as numbers: `parse::<u32>` accepts only digits.

> 🔒 **Security:** **do not "fix" bad input silently.** If you remove the strange characters and carry on,
> two different inputs can end up as the same value, or the cleaned value can still be dangerous.
> Reject with a clear message and let the caller try again.

## 5. Test with hostile input

Validation code is the best place for tests, because the bugs hide in the edges. Add the tests to
the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn accepts_good_names_and_lowercases_them() {
        assert_eq!(Username::parse("Ana_01").unwrap().as_str(), "ana_01");
    }

    #[test]
    fn border_lengths() {
        assert_eq!(Username::parse("ab"), Err(UsernameError::TooShort));
        assert!(Username::parse("abc").is_ok());
        assert!(Username::parse(&"a".repeat(32)).is_ok());
        assert_eq!(
            Username::parse(&"a".repeat(33)),
            Err(UsernameError::TooLong)
        );
    }

    #[test]
    fn rejects_nasty_input_without_panicking() {
        for input in [
            "",
            "1abc",
            "ana smith",
            "ana;rm",
            "../etc",
            "a\0b",
            "ñandú",
            "😀😀😀",
        ] {
            assert!(Username::parse(input).is_err(), "should reject {input:?}");
        }
        assert!(Username::parse(&"é".repeat(100_000)).is_err());
    }

    #[test]
    fn a_lookalike_letter_is_rejected() {
        // The first letter is the Cyrillic "а" (U+0430), which looks like the Latin "a".
        assert_eq!(
            Username::parse("\u{430}na_smith"),
            Err(UsernameError::MustStartWithLetter)
        );
    }

    #[test]
    fn quantity_range_and_total() {
        assert_eq!(Quantity::parse(" 12 ").unwrap().get(), 12);
        assert_eq!(Quantity::parse("0"), Err(QuantityError::OutOfRange));
        assert_eq!(Quantity::parse("1001"), Err(QuantityError::OutOfRange));
        assert_eq!(Quantity::parse("-3"), Err(QuantityError::NotANumber));
        assert_eq!(Quantity::parse("abc"), Err(QuantityError::NotANumber));
        let q = Quantity::parse("1000").unwrap();
        assert_eq!(total_cents(5_000_000, q), None);
        assert_eq!(total_cents(2_000, q), Some(2_000_000));
    }
}
```

```bash
cargo test
```

```text
running 5 tests
test tests::a_lookalike_letter_is_rejected ... ok
test tests::quantity_range_and_total ... ok
test tests::accepts_good_names_and_lowercases_them ... ok
test tests::border_lengths ... ok
test tests::rejects_nasty_input_without_panicking ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**What happens:**

1. `border_lengths` tests the values **at** the border: 2 (too short), 3 (ok), 32 (ok), 33 (too long).
   Off-by-one mistakes live exactly there.
2. `rejects_nasty_input_without_panicking` goes through a list of nasty texts: empty, a digit first,
   a space, a semicolon, a path, a NUL character (the byte with value zero, `\0`), non-ASCII letters, emoji, and a text of
   100 000 characters. The test is also a promise: **no input makes the validator panic**.
3. `a_lookalike_letter_is_rejected` uses `\u{430}`, the escape for the Cyrillic letter, so the test file
   itself is clear about what it tests.
4. The last test checks both range errors, the not-a-number cases, and the overflow in `total_cents`.

> 🧪 **Testing:** unit tests like these find the bugs you thought of. To find the bugs you did not think
> of, feed random input to the validator: with property-based testing ([Lesson 09.4](../09-testing-debugging-benchmarking/04-property-based-testing.md))
> or fuzzing ([Lesson 09.8](../09-testing-debugging-benchmarking/08-fuzzing.md)). The property to check is: "for any
> input, `parse` returns `Ok` or `Err` and never panics".

## 6. Regular expressions as an allow-list

A **regex** is a compact way to write an allow-list. The `regex` crate is the standard choice in Rust. The
most common mistake is forgetting the **anchors**: `^` means "start of the text" and `$` means "end of the
text". Without them a regex only asks "is there a match *somewhere inside*?". Save `src/bin/anchors.rs`:

```rust
use regex::Regex;

fn main() {
    let loose = Regex::new("[a-z]+").unwrap();
    let strict = Regex::new("^[a-z]+$").unwrap();

    for input in ["hello", "hello; echo INJECTED", "HELLO", "hello\n"] {
        println!(
            "{input:<24?} loose: {:<5} strict: {}",
            loose.is_match(input),
            strict.is_match(input)
        );
    }
}
```

```text
"hello" loose: true  strict: true
"hello; echo INJECTED" loose: true  strict: false
"HELLO" loose: false strict: false
"hello\n" loose: true  strict: false
```

**What happens:**

1. `loose` has no anchors. It says "there is a run of lowercase letters somewhere", so
   `hello; echo INJECTED` passes: the word `hello` is in it.
2. `strict` has both anchors: the **whole** text must be lowercase letters. The attack text, the capitals
   and the text with a trailing newline are all refused.
3. In this crate `$` matches only at the very end of the text, not before a final newline, so `hello\n` is refused. (Some other
   regex engines treat a trailing newline differently, which has caused real bugs, so always test it.)

**ReDoS.** In some regex engines a carefully built text can make matching take exponential time. The
`regex` crate was designed to avoid this: it does not use the "backtracking" method, and matching time
grows in proportion to the length of the text (and of the pattern). That protects you when the **text** comes from outside. If
the **pattern** itself comes from outside, a huge pattern still costs time and memory when it is
compiled, so limit its length and the compile size (see the crate documentation, "Avoiding pathological patterns").

> 💡 **Tip:** for a simple rule like our username, a hand-written check with `chars()` is easier to read and
> to test than a regex. Use a regex when the shape is really a pattern (dates, codes, versions).

## 7. Limits on everything

Besides the shape of a value, limit its **amount**: the size of a request or file (Lesson 10.1), the
**number** of items in a list, the nesting depth of JSON (`serde_json` stops at a depth of 128 by default),
and the **time** a request may take. A `Vec` that grows with every item the client sends is a memory
attack. Choose a maximum, return an error above it, and write a test for it.

## Common mistakes

> ⚠️ **Common mistake:** validating in one place and using in another. If the check is in the web form
> and the same function is also called from a file import, the file import is unprotected. Put the check
> in the type (`Username::parse`), so every path goes through it.

> ⚠️ **Common mistake:** checking **before** converting to the canonical form. If you check for the
> text `admin` and later lowercase it, then `ADMIN` passes the check and becomes `admin`. Canonicalize first.

> ⚠️ **Common mistake:** `if name.len() > 32` for non-ASCII text, thinking `len()` counts characters. It
> counts bytes. `"ñandú"` has 5 characters and 7 bytes.

> ⚠️ **Common mistake:** error messages that echo the input into a page or a log without escaping. The
> message becomes an attack. Print it with `{:?}` as in Lesson 10.3, Section 6, or escape it for HTML.

## Try it yourself

1. Type the library, the two programs and the tests. Run `cargo test` and the programs.
2. Change `MAX_USERNAME_LEN` to 8. Which test fails (`border_lengths`), and why? Fix the test, not the code, if the new
   limit is what you want.

## Exercises

### Exercise 1 — Allow-list or deny-list? (easy)

For each rule say whether it is an allow-list or a deny-list, and name one input that beats it:
(a) "reject the characters `<`, `>` and `&`"; (b) "accept `[a-z0-9]{8}`"; (c) "reject `..`"; (d) "accept
only the extensions `.png` and `.jpg`".

<details>
<summary>Hint</summary>

Ask: does the rule list what is **good** or what is **bad**?

</details>

<details>
<summary>Answer</summary>

(a) Deny-list. A quote character (`"` or `'`) still gets through. (b) Allow-list;
nothing outside the listed characters and the exact length of 8 gets through. (c) Deny-list. A path such as
`/etc/passwd` has no `..` at all, and encoded forms such as `%2e%2e` may be decoded later. (d) Allow-list
(but check the extension in canonical form: `PNG`, and `a.png.exe` has the extension `.exe`).

</details>

### Exercise 2 — A `Port` type (medium)

Write `Port::parse(&str) -> Result<Port, PortError>`. A port is a whole number from 1 to 65 535. Write
**tests** for `"0"`, `"1"`, `"65535"`, `"65536"`, `""`, `"-1"`, `" 80 "` and `"8o"`.

<details>
<summary>Hint</summary>

`u16` holds 0 to 65 535 exactly. Parse with `parse::<u16>()`, then reject 0. Note that `"65536"` fails the
**parse** (it does not fit), not the range check, so both errors can be one `Invalid` variant.

</details>

<details>
<summary>Answer</summary>

```rust
#[derive(Debug, PartialEq, Eq)]
pub struct Port(u16);

#[derive(Debug, PartialEq, Eq)]
pub enum PortError {
    Invalid,
}

impl Port {
    pub fn parse(text: &str) -> Result<Port, PortError> {
        match text.trim().parse::<u16>() {
            Ok(0) | Err(_) => Err(PortError::Invalid),
            Ok(n) => Ok(Port(n)),
        }
    }

    pub fn get(&self) -> u16 {
        self.0
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn port_borders() {
        assert_eq!(Port::parse("0"), Err(PortError::Invalid));
        assert_eq!(Port::parse("1").unwrap().get(), 1);
        assert_eq!(Port::parse("65535").unwrap().get(), 65535);
        assert_eq!(Port::parse("65536"), Err(PortError::Invalid));
        assert_eq!(Port::parse(""), Err(PortError::Invalid));
        assert_eq!(Port::parse("-1"), Err(PortError::Invalid));
        assert_eq!(Port::parse(" 80 ").unwrap().get(), 80);
        assert_eq!(Port::parse("8o"), Err(PortError::Invalid));
    }
}
```

Why it works: the `u16` type is the range check for the upper end, `Ok(0)` is the lower end, and every
text that is not a plain number lands in `Err(_)`.

</details>

### Exercise 3 — Test an anchored regex (medium)

Write a function `is_word(text: &str) -> bool` that is true only for non-empty text made of lowercase
letters `a-z`, using a regex. Write a **test** with inputs that would pass an unanchored regex but must be
refused.

<details>
<summary>Hint</summary>

Use `^[a-z]+$`. Test `"hello world"`, `"hello\n"`, `"x; rm"`, `""` and `"Hello"`.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn is_word(text: &str) -> bool {
    regex::Regex::new("^[a-z]+$").unwrap().is_match(text)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn the_regex_is_anchored() {
        assert!(is_word("hello"));
        assert!(!is_word(""));
        assert!(!is_word("hello world"));
        assert!(!is_word("hello\n"));
        assert!(!is_word("x; rm"));
        assert!(!is_word("Hello"));
    }
}
```

Why it works: `^` and `$` force the **whole** text to match, `+` needs at least one letter, and the class
`[a-z]` contains no space, semicolon or capital letter. (In real code build the `Regex` once and reuse it:
compiling it on every call is slow.)

</details>

## Quiz

1. What is the difference between validating and escaping?

<details>
<summary>Answer</summary>

Validation checks that a value is a legal value and rejects bad ones, at the boundary. Escaping makes a
value safe for one destination (HTML, SQL, shell) at the place where it is used.

</details>

2. Why is an allow-list safer than a deny-list?

<details>
<summary>Answer</summary>

An allow-list rejects everything you did not list, including what you forgot or what will be invented later.
A deny-list passes everything you forgot.

</details>

3. What does "parse, don't validate" mean?

<details>
<summary>Answer</summary>

Convert raw input into a type that can hold only valid values (private field, one constructor that checks). Then
every value of that type in the program is known to be valid, and no later code has to check again.

</details>

4. Why does a regex like `[a-z]+` accept `hello; echo INJECTED`?

<details>
<summary>Answer</summary>

Without `^` and `$` the regex only needs a match somewhere in the text, and `hello` matches.

</details>

## Summary

- Input is hostile until checked; check once, at the trust boundary, as early as possible.
- Validate (reject) first; escape where the value is used; avoid silent sanitizing.
- Use an allow-list. A deny-list is a list of things you hope you remembered.
- Order: size, encoding, canonical form, syntax, meaning.
- Parse into types (`Username`, `Quantity`) with private fields, so valid values are the only values.
- Use `checked_*` arithmetic on numbers from outside, and check ranges.
- Anchor your regexes; Rust's `regex` matches in time proportional to the text length.
- Test the borders and hostile inputs; use fuzzing to look for the rest.

## Sources and further reading

- [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html) — practical rules for syntax and semantic validation, allow-lists and more.
- [Parse, don't validate (Alexis King)](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) — the article that made the "parse, don't validate" idea popular.
- [The `regex` crate documentation](https://docs.rs/regex/1.13.1/regex/) — syntax, anchors, and the section on avoiding pathological patterns.
- [The Rust standard library — primitive `char`](https://doc.rust-lang.org/std/primitive.char.html) — `is_ascii_alphabetic` and the other character methods.
- [Unicode Technical Standard #39 — Security Mechanisms](https://www.unicode.org/reports/tr39/) — confusable characters (homoglyphs) and how to detect them.
- [The Rust Book — To `panic!` or Not to `panic!`](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html) — when a program may panic and when it must return an error.

---

[🏠 Start](../START-HERE.md) · [← 10.3 The OWASP Top 10](03-the-owasp-top-10.md) · [📋 Phase 10](README.md) · [10.5 Injection attacks: SQL, commands and paths →](05-injection-attacks.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
