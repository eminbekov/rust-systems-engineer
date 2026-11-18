<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 02.1 Variables and mutability](01-variables-and-mutability.md) · [📋 Phase 02](README.md) · [02.3 Operators and expressions →](03-operators-and-expressions.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/02-programming-basics/02-data-types.md) <!-- nav -->

# Lesson 02.2 — Data types

> **Phase 02 — Programming basics in Rust** · Lesson 2 of 11 · ⏱ about 3 hours (2 sittings)

## What you will learn

- What a type is, and why Rust wants to know the type of every value.
- The integer types (`i32`, `u8`, `usize`…), their ranges, and what overflow is.
- Decimal numbers (`f64`), true/false values (`bool`) and single characters (`char`).
- Tuples and arrays: how to keep several values together.
- How to convert one number type into another with `as`, and what happens when a value does not fit.

## Before you start

- You finished [Lesson 02.1 — Variables and mutability](01-variables-and-mutability.md).
- You have the practice project `~/projects/rust/code/p02-basics`. Put the examples of this lesson in `src/bin/` and run them with `cargo run --bin <file name without .rs>`.

## New words

| Word | Meaning |
|---|---|
| **type** | The kind of a value. It decides what the value can hold and what you can do with it. |
| **statically typed** | The type of every variable is known when the program is compiled, before it runs. Rust is statically typed. |
| **type inference** | The compiler guesses the type from the value, so you often do not need to write it. |
| **type annotation** | A type that you write yourself: `let x: u8 = 5;`. |
| **scalar type** | A type that holds one single value: integer, float, `bool`, `char`. |
| **compound type** | A type that groups several values: tuple, array. |
| **bit** | The smallest piece of data: `0` or `1`. |
| **byte** | A group of 8 bits. It can hold 256 different values (0 to 255). |
| **integer** | A whole number, with no fractional part: `-3`, `0`, `42`. |
| **signed / unsigned** | A signed type (`i32`) can be negative or positive. An unsigned type (`u32`) is never negative. |
| **overflow** | The result of a calculation does not fit in the type, for example `255 + 1` in a `u8`. |
| **float** (floating-point number) | A number with a fractional part: `3.14`. Types `f32` and `f64`. |
| **IEEE 754** (Institute of Electrical and Electronics Engineers, standard 754) | The international standard that describes how computers store floats. |
| **NaN** (Not a Number) | A special float value for an impossible result, such as `0.0 / 0.0`. |
| **`bool`** (Boolean) | A type with two values: `true` and `false`. |
| **`char`** (character) | One character. It is written in single quotes: `'A'`. |
| **Unicode** | The world standard that gives a number to every letter, digit, symbol and emoji of every language. |
| **tuple** | A fixed group of values that may have different types: `(1, "a", 2.5)`. |
| **array** | A fixed-length list of values of the same type: `[1, 2, 3]`. |
| **index** | The position of an item in an array. The first item has index 0. |
| **unit** | The empty tuple `()`. It means "no value". |
| **ASCII** (American Standard Code for Information Interchange) | An old table that gives a number from 0 to 127 to English letters, digits and some symbols (`A` is 65). Unicode includes it. |
| **MB** (megabyte) | About one million bytes. |
| **CPU** (Central Processing Unit) | The chip that runs the instructions of a program. |
| **Option** | A type that is either `Some(value)` or `None` ("nothing"). Phase 04 teaches it fully. |

## 1. What is a type?

Every value in a program has a **type**. A type says what kind of data it is: a whole number, a number with a fraction, a letter, a true/false answer. The type decides how many bytes the value uses in memory and what operations are allowed. You can add two numbers, but you cannot add a number to a letter.

Rust is **statically typed**: the compiler must know the type of every variable before the program runs. That sounds strict, but it gives you a big gift. A whole class of mistakes ("I used text where a number was needed") is caught while you write, not when a user runs the program.

Usually you do not have to write the types. The compiler **infers** them from the values:

```rust
fn main() {
    let a = 5;
    let b = 2.5;
    let c = true;
    let d = 'x';
    let e = "text";
    let f: u8 = 5;
    println!("a: {}", std::any::type_name_of_val(&a));
    println!("b: {}", std::any::type_name_of_val(&b));
    println!("c: {}", std::any::type_name_of_val(&c));
    println!("d: {}", std::any::type_name_of_val(&d));
    println!("e: {}", std::any::type_name_of_val(&e));
    println!("f: {}", std::any::type_name_of_val(&f));
}
```

What happens:

1. Six variables are created. Five have no written type; the compiler infers it. For `f` we wrote a **type annotation**: `: u8` after the name.
2. `std::any::type_name_of_val(&a)` asks the standard library for the name of the type of `a`. (The `&` before `a` means "look at `a` without taking it away". Phase 03 explains this fully. For now, copy it.)
3. Each `println!` prints a letter and a type name.

```text
a: i32
b: f64
c: bool
d: char
e: &str
f: u8
```

So: a whole number without annotation becomes `i32`, a number with a dot becomes `f64`, and text in double quotes is a `&str` ("string slice", a piece of text; Phase 03 explains it).

> 💡 **Tip:** You do not need that long function to see a type. RustRover shows the inferred type in grey next to the variable. Another trick is to write a wrong annotation on purpose and read the error.

You write a type annotation when the compiler cannot guess, or when you want a specific type:

```rust
fn main() {
    let number = "42".parse().unwrap();
    println!("{number}");
}
```

```text
error[E0284]: type annotations needed
 --> src/main.rs:2:9
  |
2 |     let number = "42".parse().unwrap();
  |         ^^^^^^        ----- type must be known at this point
  |
help: consider giving `number` an explicit type
  |
2 |     let number: /* Type */ = "42".parse().unwrap();
  |               ++++++++++++
```

(`parse` turns text into a number; Lesson 02.7 teaches it.) Text like `"42"` could become an `i32`, a `u8` or a `f64`. The compiler asks you to choose: `let number: i32 = "42".parse().unwrap();`.

## 2. Integer types

An **integer** is a whole number. Rust has many integer types. Their names tell you two things: the first letter shows whether the type is **signed** (`i`) or **unsigned** (`u`), and the number is the count of **bits**.

| Length | Signed | Unsigned |
|---|---|---|
| 8 bits | `i8` | `u8` |
| 16 bits | `i16` | `u16` |
| 32 bits | `i32` | `u32` |
| 64 bits | `i64` | `u64` |
| 128 bits | `i128` | `u128` |
| architecture-dependent | `isize` | `usize` |

A **bit** is a `0` or a `1`. With *n* bits you can make 2ⁿ different combinations. 8 bits give 256 combinations, so a `u8` holds `0` to `255`. A signed type uses one combination for the sign, so an `i8` holds `-128` to `127`.

The formulas:

- unsigned, *n* bits: from `0` to 2ⁿ − 1
- signed, *n* bits: from −2ⁿ⁻¹ to 2ⁿ⁻¹ − 1

Every type has constants for its limits: `i32::MAX`, `u8::MIN`.

```rust
fn main() {
    println!("i8:   {} to {}", i8::MIN, i8::MAX);
    println!("u8:   {} to {}", u8::MIN, u8::MAX);
    println!("i16:  {} to {}", i16::MIN, i16::MAX);
    println!("i32:  {} to {}", i32::MIN, i32::MAX);
    println!("u32:  {} to {}", u32::MIN, u32::MAX);
    println!("i64:  {} to {}", i64::MIN, i64::MAX);
    println!("u64:  {} to {}", u64::MIN, u64::MAX);
    println!("usize max: {}", usize::MAX);
}
```

```text
i8:   -128 to 127
u8:   0 to 255
i16:  -32768 to 32767
i32:  -2147483648 to 2147483647
u32:  0 to 4294967295
i64:  -9223372036854775808 to 9223372036854775807
u64:  0 to 18446744073709551615
usize max: 18446744073709551615
```

**`isize` and `usize`** have the size of a memory address on the computer where the program runs. On your Apple M4 Mac (a 64-bit CPU, Central Processing Unit) they are 64 bits. Rust uses `usize` for **counting things and for indexes**: the length of an array is a `usize`.

**Which one should I choose?**

- If you have no reason to choose, use `i32`. It is the default.
- Use `usize` for indexes and sizes of collections.
- Use `u8` for a raw byte (0 to 255) and for small bounded numbers.
- Use `u64` or `i64` for big counters, money in cents, file sizes, time in seconds.
- Use an unsigned type only when negative numbers make no sense **and** you are sure you will never subtract below zero.

How much memory does each type take? The function `size_of` tells you, in bytes (a **byte** is 8 bits):

```rust
use std::mem::size_of;

fn main() {
    println!("u8:    {} byte", size_of::<u8>());
    println!("i32:   {} bytes", size_of::<i32>());
    println!("i64:   {} bytes", size_of::<i64>());
    println!("u128:  {} bytes", size_of::<u128>());
    println!("usize: {} bytes", size_of::<usize>());
    println!("f32:   {} bytes", size_of::<f32>());
    println!("f64:   {} bytes", size_of::<f64>());
    println!("bool:  {} byte", size_of::<bool>());
    println!("char:  {} bytes", size_of::<char>());
    println!("():    {} bytes", size_of::<()>());
}
```

What happens:

1. `use std::mem::size_of;` brings the function `size_of` into the file so you can write its short name. (`use` is explained in Phase 04, Lesson 6. For now: it is a shortcut for a long name.)
2. `size_of::<u8>()` — the part `::<u8>` tells the function *which type* to measure. (This is called a *generic argument*; Phase 05 explains generics.)
3. Each line prints the size in bytes.

```text
u8:    1 byte
i32:   4 bytes
i64:   8 bytes
u128:  16 bytes
usize: 8 bytes
f32:   4 bytes
f64:   8 bytes
bool:  1 byte
char:  4 bytes
():    0 bytes
```

### Number literals

A **literal** is a value written directly in the code. Rust accepts several forms:

```rust
fn main() {
    let million = 1_000_000;
    let hex = 0xff;
    let octal = 0o77;
    let binary = 0b1111_0000;
    let byte = b'A';
    let typed = 42u8;
    println!("{million} {hex} {octal} {binary} {byte} {typed}");
}
```

```text
1000000 255 63 240 65 42
```

- `1_000_000` — underscores are ignored; they make long numbers readable.
- `0xff` — hexadecimal (base 16). `0o77` — octal (base 8). `0b1111_0000` — binary (base 2).
- `b'A'` — the number of the ASCII (American Standard Code for Information Interchange) code of the letter `A`, as a `u8`: `65`.
- `42u8` — a suffix after the number sets its type.

> 📏 **Convention:** Put underscores in numbers with more than four digits (`1_000_000`). Write binary and hexadecimal numbers when you work with bits and bytes: that is how systems engineers think.

## 3. Overflow

What happens if a `u8` holds `255` and you add `1`? The result `256` does not fit in 8 bits. This is called **overflow**. Rust's behaviour depends on how you build the program:

```rust
fn add_one(x: u8) -> u8 {
    x + 1
}

fn main() {
    println!("{}", add_one(254));
    println!("{}", add_one(255));
}
```

(A *function* is a named piece of code with inputs and an output. Lesson 02.4 teaches functions; here `add_one` takes a `u8` and returns a `u8`.)

In the default **debug build** (what `cargo run` makes), the program panics:

```text
255

thread 'main' (3887855) panicked at src/main.rs:2:5:
attempt to add with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

In a **release build** (`cargo run --release`), checks for overflow are switched off to make the program faster, and the number silently **wraps around** like an odometer in a car:

```text
255
0
```

Neither result is the "right" answer. So when overflow is possible, say clearly what you want. Every integer type has methods for this:

```rust
fn main() {
    let big: u8 = 250;
    println!("checked_add(10):    {:?}", big.checked_add(10));
    println!("checked_add(5):     {:?}", big.checked_add(5));
    println!("wrapping_add(10):   {}", big.wrapping_add(10));
    println!("saturating_add(10): {}", big.saturating_add(10));
    println!("overflowing_add(10): {:?}", big.overflowing_add(10));
}
```

```text
checked_add(10):    None
checked_add(5):     Some(255)
wrapping_add(10):   4
saturating_add(10): 255
overflowing_add(10): (4, true)
```

| Method | What it does on overflow |
|---|---|
| `checked_add` | Returns `None` ("nothing") if it overflows, or `Some(result)` if it fits. `Some`/`None` belong to the type **Option**, taught in Phase 04. |
| `wrapping_add` | Wraps around: `250 + 10` becomes `4`. |
| `saturating_add` | Stops at the limit: `255`. |
| `overflowing_add` | Returns a pair: the wrapped result and `true` if overflow happened. |

The same family exists for subtraction and multiplication (`checked_sub`, `wrapping_mul`, …). The format `{:?}` prints a value in "debug" form; Lesson 02.6 explains it.

> 🔒 **Security:** Integer overflow is a real source of security bugs. If a program computes a buffer size as `count * size` and the result wraps to a small number, the program may reserve too little memory and then write past its end. When numbers come from outside (a file, the network, a user), use `checked_*` methods. In Phase 10 you will see real examples.

The compiler also catches overflow that it can see at compile time:

```rust
fn main() {
    let x: u8 = 256;
    println!("{x}");
}
```

```text
error: literal out of range for `u8`
 --> src/main.rs:2:17
  |
2 |     let x: u8 = 256;
  |                 ^^^
  |
  = note: the literal `256` does not fit into the type `u8` whose range is `0..=255`
  = note: `#[deny(overflowing_literals)]` on by default
```

And a negative number cannot be put into an unsigned type:

```text
error[E0600]: cannot apply unary operator `-` to type `u32`
 --> src/main.rs:2:24
  |
2 |     let balance: u32 = -5;
  |                        ^^ cannot apply unary operator `-`
  |
  = note: unsigned values cannot be negated
```

## 4. Floating-point numbers

For numbers with a fractional part, Rust has two **float** types: `f32` (32 bits) and `f64` (64 bits). The default is `f64`: on modern CPUs it is almost as fast as `f32` and it is much more precise.

```rust
fn main() {
    let a = 0.1 + 0.2;
    println!("0.1 + 0.2 = {a}");
    println!("equal to 0.3? {}", a == 0.3);
    let close = (a - 0.3_f64).abs() < 1e-9;
    println!("close to 0.3? {close}");
    let x: f32 = 1.0 / 3.0;
    let y: f64 = 1.0 / 3.0;
    println!("f32: {x}");
    println!("f64: {y}");
    println!("1.0 / 0.0 = {}", 1.0_f64 / 0.0);
    println!("0.0 / 0.0 = {}", 0.0_f64 / 0.0);
}
```

What happens:

1. `0.1 + 0.2` is not exactly `0.3` in a computer. Floats are stored in binary (the **IEEE 754** standard), and numbers like `0.1` cannot be written exactly in binary — just as 1/3 cannot be written exactly in decimal.
2. `a == 0.3` compares with `==` ("is equal to") and gives `false`.
3. The right way to compare floats is to ask "is the difference tiny?": `(a - 0.3).abs() < 1e-9`. The method `abs()` removes the minus sign. `1e-9` means 1 × 10⁻⁹ = 0.000000001. The suffix in `0.3_f64` sets the type.
4. `f32` keeps about 7 correct digits, `f64` about 15–16.
5. Dividing a float by zero does not crash: `1.0 / 0.0` is `inf` (infinity), and `0.0 / 0.0` is `NaN` (Not a Number).

```text
0.1 + 0.2 = 0.30000000000000004
equal to 0.3? false
close to 0.3? true
f32: 0.33333334
f64: 0.3333333333333333
1.0 / 0.0 = inf
0.0 / 0.0 = NaN
```

> ⚠️ **Common mistake:** Never compare floats with `==`, and never store money in floats. Use whole numbers: keep prices in cents (`i64`).

## 5. Booleans

The type `bool` has exactly two values: `true` and `false`. It is the result of every comparison and the input of every condition (Lesson 02.5).

```rust
fn main() {
    let is_ready = true;
    let is_big: bool = 10 > 5;
    println!("{is_ready} {is_big}");
}
```

```text
true true
```

A `bool` takes 1 byte. Rust is strict: a number is not a `bool`. `let h: bool = 1;` is an error (`E0308 mismatched types`). If you mean "is this number not zero?", write the comparison: `n != 0`.

## 6. Characters

A `char` holds **one character**, written in **single quotes**. In Rust it is a **Unicode** character, so it can be a Latin letter, a Cyrillic letter, an emoji, and more. It takes 4 bytes.

```rust
fn main() {
    let letter = 'A';
    let heart = '❤';
    let cyrillic = 'Ж';
    let crab = '🦀';
    println!("{letter} {heart} {cyrillic} {crab}");
    println!("'A' as number: {}", letter as u32);
    println!("is 'Ж' alphabetic? {}", cyrillic.is_alphabetic());
    println!("is '7' a digit? {}", '7'.is_ascii_digit());
}
```

```text
A ❤ Ж 🦀
'A' as number: 65
is 'Ж' alphabetic? true
is '7' a digit? true
```

Remember: `'A'` (single quotes) is a `char`; `"A"` (double quotes) is text of type `&str`. They are different types. Text is more complicated than it looks, and Rust is careful with it; Phase 03 (Lesson 6) and Phase 04 (Lesson 8) explain text in depth. For now you can use text literals with `println!` and compare them.

## 7. Converting numbers with `as`

Rust never mixes number types silently. Adding an `i32` to an `i64` is an error:

```rust
fn main() {
    let small: i32 = 10;
    let big: i64 = 20;
    let sum = small + big;
    println!("{sum}");
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:4:23
  |
4 |     let sum = small + big;
  |                       ^^^ expected `i32`, found `i64`

error[E0277]: cannot add `i64` to `i32`
 --> src/main.rs:4:21
  |
4 |     let sum = small + big;
  |                     ^ no implementation for `i32 + i64`
  |
  = help: the trait `Add<i64>` is not implemented for `i32`
```

(The compiler prints a long list of notes after this; they are not needed now. Read the first lines of an error first.)

The fix is to convert one of the numbers with the keyword `as`:

```rust
fn main() {
    let small: i32 = 10;
    let big: i64 = 20;
    let sum = small as i64 + big;
    println!("sum = {sum}");

    println!("300 as u8   = {}", 300_i32 as u8);
    println!("-1 as u32   = {}", -1_i32 as u32);
    println!("3.99 as i32 = {}", 3.99_f64 as i32);
    println!("7 as f64    = {}", 7_i32 as f64);
    println!("'a' as u8   = {}", 'a' as u8);
    println!("97 as char  = {}", 97_u8 as char);
}
```

```text
sum = 30
300 as u8   = 44
-1 as u32   = 4294967295
3.99 as i32 = 3
7 as f64    = 7
'a' as u8   = 97
97 as char  = a
```

What happens:

1. `small as i64` makes a new `i64` with the same number. Now both sides of `+` are `i64`.
2. `300 as u8` does **not** fail. It keeps only the lowest 8 bits: `300 − 256 = 44`.
3. `-1 as u32` reinterprets the bits of `-1`: all 32 bits are ones, which is `4294967295` as an unsigned number.
4. `3.99 as i32` cuts off the fraction (it does not round): `3`.
5. `as f64` turns an integer into a float. Note that `{}` prints the float `7.0` as `7`.
6. Between `char` and `u8`, `as` converts a character to its code and back (only `u8` can become a `char` this way).

`as` is powerful but it can **silently lose information** (case 2, 3, 4). Widening conversions (`i32` to `i64`, `u8` to `u32`) are always safe. For narrowing ones, prefer checked conversions, which you will meet in Phase 05 (`try_from`). Lesson 02.3 shows the `as` operator again together with the other operators.

## 8. Tuples and the unit type

A **tuple** groups a few values into one. The values may have different types, and the size is fixed when you write it.

```rust
fn main() {
    let person = ("Aziza", 29, 1.68);
    println!("{:?}", person);
    println!("name = {}", person.0);
    println!("age = {}", person.1);

    let (name, age, height) = person;
    println!("{name} is {age} years old and {height} m tall");

    let nothing = ();
    println!("{:?}", nothing);
}
```

What happens:

1. `("Aziza", 29, 1.68)` is a tuple of three values: a `&str`, an `i32` and an `f64`. Its type is `(&str, i32, f64)`.
2. `{:?}` prints a whole tuple (the plain `{}` cannot print tuples, because there is no single obvious way to show them).
3. `person.0` and `person.1` read the first and second items. Counting starts at **0**.
4. `let (name, age, height) = person;` is **destructuring**: it breaks the tuple into three separate variables in one line.
5. `()` is the **unit** type: an empty tuple, a value that carries no information. A function that returns nothing actually returns `()`. It takes 0 bytes.

```text
("Aziza", 29, 1.68)
name = Aziza
age = 29
Aziza is 29 years old and 1.68 m tall
()
```

## 9. Arrays

An **array** holds several values of the **same type**, and its length is fixed. The type is written `[Type; length]`.

```rust
fn main() {
    let days = [31, 28, 31, 30];
    println!("{:?}", days);
    println!("first = {}, last = {}", days[0], days[3]);
    println!("count = {}", days.len());

    let zeros = [0; 5];
    println!("{:?}", zeros);

    let names: [&str; 3] = ["Ann", "Bob", "Cy"];
    println!("{:?}", names);

    let mut scores = [10, 20, 30];
    scores[1] = 99;
    println!("{:?}", scores);
}
```

What happens:

1. `[31, 28, 31, 30]` is an array of four `i32` numbers: its type is `[i32; 4]`.
2. `days[0]` reads the item at **index** 0 (the first). `days[3]` is the last one. The index is a `usize`.
3. `days.len()` is the number of items: `4`.
4. `[0; 5]` is a shortcut: the value `0` repeated 5 times.
5. `let names: [&str; 3]` shows an annotation: three items of type `&str`.
6. `scores[1] = 99;` changes one item. This is allowed only because `scores` is declared with `mut`.

```text
[31, 28, 31, 30]
first = 31, last = 30
count = 4
[0, 0, 0, 0, 0]
["Ann", "Bob", "Cy"]
[10, 99, 30]
```

An array lives in one continuous block of memory, one item after another:

```text
index:   0     1     2     3
       +-----+-----+-----+-----+
days   |  31 |  28 |  31 |  30 |     4 items x 4 bytes = 16 bytes
       +-----+-----+-----+-----+
```

### Out-of-bounds indexes

What if you ask for an item that does not exist? Rust checks every index and panics, instead of reading random memory. In many older languages the same mistake silently reads or writes memory that does not belong to the array — a famous source of security holes.

```rust
fn main() {
    let days = [31, 28, 31, 30];
    let index = days.len() + 2;
    println!("trying index {index}");
    let value = days[index];
    println!("value = {value}");
}
```

```text
trying index 6

thread 'main' (3888423) panicked at src/main.rs:5:17:
index out of bounds: the len is 4 but the index is 6
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

If the compiler can already see the mistake, it stops you earlier. With `days[7]` written directly, it refuses to build:

```text
error: this operation will panic at runtime
 --> src/main.rs:3:17
  |
3 |     let value = days[7];
  |                 ^^^^^^^ index out of bounds: the length is 4 but the index is 7
  |
  = note: `#[deny(unconditional_panic)]` on by default
```

Arrays have a fixed size. When you need a list that can grow, you will use a *vector* (`Vec`), taught in Phase 04, Lesson 7.

## Common mistakes

**1. Mixing number types.** `i32 + i64` gives `E0308` and `E0277`. Convert with `as` (or choose the same type from the start).

**2. Expecting `/` to give decimals for integers.** `7 / 2` is `3`, because both are integers. Lesson 02.3 explains this. Use `7.0 / 2.0` for `3.5`.

**3. A type that is too small.** A `u8` counter that grows above 255 panics in debug mode and wraps in release mode. Choose a bigger type, or use `checked_add`.

**4. Using `==` on floats.** Compare with a small tolerance, as in section 4.

**5. Confusing `'a'` and `"a"`.** One is a `char`, the other is text (`&str`). `let c: char = "x";` gives `E0308 mismatched types: expected char, found &str`.

**6. Wrong index.** An index equal to the length is already out of bounds. The last item of a 4-item array has index `3`.

> 🧪 **Testing:** Limits and conversions are easy to check with assertions. Write down what you expect, and let the computer confirm:
>
> ```rust
> use std::mem::size_of;
>
> fn main() {
>     assert_eq!(size_of::<u16>(), 2);
>     assert_eq!(size_of::<i128>(), 16);
>     assert_eq!(u16::MAX, 65535);
>     assert_eq!(i16::MIN, -32768);
>     assert_eq!(200_u8.checked_add(100), None);
>     assert_eq!(200_u8.wrapping_add(100), 44);
>     assert_eq!(200_u8.saturating_add(100), 255);
>     println!("all checks passed");
> }
> ```
>
> ```text
> all checks passed
> ```
>
> Always test the **edges**: the smallest value, the biggest value, and one step beyond them. Bugs live at the edges.

> ⚡ **Performance:** A smaller type uses less memory, and an array of a million `u8` takes 1 MB (megabyte) while a million `i64` takes 8 MB. But do not shrink types without a reason: correctness first, and measure before you optimize. In an optimized build (`--release`), integer arithmetic becomes single CPU instructions; a debug build only adds the overflow check you saw in section 3.

## Try it yourself

Work in `~/projects/rust/code/p02-basics`.

1. Create `src/bin/02_types.rs`. Copy the program from section 1 (the type names) and run it with `cargo run --bin 02_types`.
2. Add `let g: i64 = 5;` and `let h = 5u16;` and print their types too.
3. Add a program `src/bin/02_limits.rs` that prints `MIN` and `MAX` for `i8`, `u16` and `i64`, and compare with the table in section 2.
4. In a new file `src/bin/02_overflow.rs`, run `add_one(255)` from section 3 with `cargo run --bin 02_overflow`, then with `cargo run --release --bin 02_overflow`. Describe the difference in your own words.
5. Create an array of your own five numbers. Print the first, the last and the length. Then deliberately read index 5 using a variable (as in section 9) and read the panic message.
6. Run `cargo fmt` and `cargo clippy`, then commit:

```bash
cd ~/projects/rust
git add code/p02-basics
git commit -m "feat(p02): add data type examples"
```

## Exercises

### Exercise 1 — Which lines compile? (easy)

For each line, say whether it compiles. Then check by putting each line alone inside `fn main() { … }`.

```rust
let a: u8 = 255;
let b: u8 = 256;
let c: i8 = -129;
let d: u32 = -1;
let e: f32 = 1;
let f: f32 = 1.0;
let g: char = "x";
let h: bool = 1;
```

<details>
<summary>Hint</summary>

Check the range of the type, the sign, and whether the literal has a fractional part. Single quotes are `char`.

</details>

<details>
<summary>Answer</summary>

Only `a` and `f` compile.

- `b`: error `literal out of range for u8` (256 > 255).
- `c`: error `literal out of range for i8` (−129 < −128).
- `d`: error `E0600 cannot apply unary operator - to type u32`.
- `e`: error `E0308 mismatched types` — `1` is an integer literal, but `f32` needs a float literal such as `1.0`.
- `g`: error `E0308` — `"x"` is a `&str`, not a `char`.
- `h`: error `E0308` — a number is not a `bool`.

Why it works: the compiler checks every literal against the declared type.

</details>

### Exercise 2 — Coordinates and colours (easy)

Create a tuple `point = (3, -4)` and destructure it into `x` and `y`. Create an array `rgb: [u8; 3] = [255, 128, 0]` and print each colour. Then add the three colours, as `u32` numbers, and print the sum. Finally print `36.6` converted to an `i32`.

<details>
<summary>Hint</summary>

Adding three `u8` values can overflow (`255 + 128`). Convert each item with `as u32` before adding.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let point = (3, -4);
    let (x, y) = point;
    println!("x = {x}, y = {y}");
    let rgb: [u8; 3] = [255, 128, 0];
    println!("red = {}, green = {}, blue = {}", rgb[0], rgb[1], rgb[2]);
    println!("sum = {}", rgb[0] as u32 + rgb[1] as u32 + rgb[2] as u32);
    let temp: f64 = 36.6;
    let rounded = temp as i32;
    println!("rounded down: {rounded}");
}
```

Output:

```text
x = 3, y = -4
red = 255, green = 128, blue = 0
sum = 383
rounded down: 36
```

Why it works: each item is widened to `u32` before the addition, so the sum 383 fits. `as i32` cuts off `.6`; it does not round.

</details>

### Exercise 3 — Average (medium)

You have the array `grades = [90, 75, 82, 68, 100]`. Compute `total` (the sum) and `average` (as an `f64`) without a loop: add the items by index. Use `grades.len()` for the count. Print both.

<details>
<summary>Hint</summary>

Integer division would lose the fraction. Convert both `total` and `grades.len()` with `as f64` before dividing.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let grades = [90, 75, 82, 68, 100];
    let total = grades[0] + grades[1] + grades[2] + grades[3] + grades[4];
    let average = total as f64 / grades.len() as f64;
    println!("total = {total}");
    println!("average = {average}");
}
```

```text
total = 415
average = 83
```

Why it works: `415 as f64 / 5 as f64` is the float division `415.0 / 5.0 = 83.0`, and `{}` prints a whole float without `.0`.

</details>

### Exercise 4 — Write checks for overflow (medium)

Write a program with `assert_eq!` checks that prove these facts, and prints `all checks passed` if they are true: `u16::MAX` is `65535`; `i16::MIN` is `-32768`; a `u16` takes 2 bytes; `200_u8.checked_add(100)` is `None`; `200_u8.wrapping_add(100)` is `44`; `200_u8.saturating_add(100)` is `255`. Then change one expected number on purpose and read the panic message.

<details>
<summary>Hint</summary>

Use `std::mem::size_of::<u16>()` for the size. Write `use std::mem::size_of;` at the top of the file.

</details>

<details>
<summary>Answer</summary>

```rust
use std::mem::size_of;

fn main() {
    assert_eq!(size_of::<u16>(), 2);
    assert_eq!(u16::MAX, 65535);
    assert_eq!(i16::MIN, -32768);
    assert_eq!(200_u8.checked_add(100), None);
    assert_eq!(200_u8.wrapping_add(100), 44);
    assert_eq!(200_u8.saturating_add(100), 255);
    println!("all checks passed");
}
```

Why it works: `200 + 100 = 300`; `300 − 256 = 44` when wrapping, and the saturating version stops at the maximum 255. The output is `all checks passed`.

</details>

### Exercise 5 — Choose the right type (hard)

Choose the best type for each value, then prove with assertions that your choice can hold the expected values: (a) a person's age, from 0 to 150; (b) the population of the Earth, about 8 billion; (c) the brightness of a pixel, from 0 to 255; (d) a bank balance that can be negative. Print `all type choices fit` at the end.

<details>
<summary>Hint</summary>

`u32::MAX` is about 4.29 billion, so it is too small for 8 billion. To compare a `u32` with a bigger number, first convert it: `u32::MAX as u64`. `assert!(condition)` panics when the condition is false.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    // age 0..=150 fits in u8 (0..=255)
    assert!(u8::MAX >= 150);
    // world population (about 8 billion) does not fit in u32, but fits in u64
    assert!((u32::MAX as u64) < 8_000_000_000);
    assert!(u64::MAX > 8_000_000_000);
    // pixel brightness 0..=255 is exactly u8
    assert_eq!(u8::MAX, 255);
    // a balance that can be negative needs a signed type
    let balance: i64 = -250;
    assert!(balance < 0);
    println!("all type choices fit");
}
```

Why it works: (a) `u8`, (b) `u64`, (c) `u8`, (d) `i64` (signed, and large enough for amounts in cents). The output is `all type choices fit`.

</details>

## Quiz

1. What type does `let x = 5;` have, and what type does `let y = 5.0;` have?

<details>
<summary>Answer</summary>

`x` is `i32` and `y` is `f64`. These are the defaults.

</details>

2. What is the range of `u8`? Of `i8`?

<details>
<summary>Answer</summary>

`u8`: 0 to 255. `i8`: −128 to 127. Both have 8 bits, so 256 values.

</details>

3. What happens when `255_u8 + 1` is calculated at run time in a debug build? In a release build?

<details>
<summary>Answer</summary>

Debug: the program panics with `attempt to add with overflow`. Release: the result wraps around to `0`.

</details>

4. Which method returns `None` on overflow, and which one stops at the maximum?

<details>
<summary>Answer</summary>

`checked_add` returns `None` on overflow. `saturating_add` stops at the maximum (or minimum) value.

</details>

5. What is the difference between `'a'` and `"a"`?

<details>
<summary>Answer</summary>

`'a'` is a `char` (one Unicode character, 4 bytes). `"a"` is text of type `&str`.

</details>

6. How do you read the second item of an array `a`, and what is its type?

<details>
<summary>Answer</summary>

`a[1]`. Indexes start at 0, and the index itself is a `usize`.

</details>

7. What does `3.99 as i32` give, and why can `as` be dangerous?

<details>
<summary>Answer</summary>

It gives `3` (the fraction is cut off). `as` may silently lose information: `300 as u8` becomes `44`, and `-1 as u32` becomes `4294967295`.

</details>

8. Why is `0.1 + 0.2 == 0.3` false?

<details>
<summary>Answer</summary>

Floats are stored in binary, and `0.1` and `0.2` cannot be stored exactly. The sum is `0.30000000000000004`. Compare floats with a small tolerance instead.

</details>

## Summary

- Rust is statically typed; the compiler infers most types. Defaults: `i32` and `f64`.
- Integer types are `i8…i128`, `u8…u128`, `isize` and `usize`. Use `usize` for indexes and sizes.
- Overflow panics in debug builds and wraps in release builds. Use `checked_*`, `wrapping_*` or `saturating_*` when it can happen.
- Floats (`f32`, `f64`) are approximate. Do not compare them with `==`.
- `bool` is `true`/`false`; `char` is one Unicode character in single quotes.
- Tuples group values of different types; arrays group values of one type, with a fixed length.
- Array indexes start at 0 and are checked; a bad index panics.
- `as` converts numbers but can lose information; Rust never converts numbers silently.

## Sources and further reading

- [The Rust Programming Language — 3.2 Data Types](https://doc.rust-lang.org/book/ch03-02-data-types.html) — the official book chapter on this topic.
- [Rust by Example — Primitives](https://doc.rust-lang.org/rust-by-example/primitives.html) — runnable examples of scalar types, tuples and arrays.
- [Rust by Example — Casting](https://doc.rust-lang.org/rust-by-example/types/cast.html) — what `as` does in each case.
- [The Rust Reference — Numeric types](https://doc.rust-lang.org/reference/types/numeric.html) — the exact list and sizes of the numeric types.
- [Standard library — Primitive Type u8](https://doc.rust-lang.org/std/primitive.u8.html) — all methods of an integer type, including `checked_add`.
- [Unicode — What is Unicode?](https://www.unicode.org/standard/WhatIsUnicode.html) — a short official introduction to Unicode.

---

[🏠 Start](../START-HERE.md) · [← 02.1 Variables and mutability](01-variables-and-mutability.md) · [📋 Phase 02](README.md) · [02.3 Operators and expressions →](03-operators-and-expressions.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
