<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 12.1 Bits, bytes and number systems](01-bits-bytes-and-number-systems.md) · [📋 Phase 12](README.md) · [12.3 Floating-point numbers →](03-floating-point-numbers.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/02-integers-in-memory.md) <!-- nav -->

# Lesson 12.2 — Integers in memory: two's complement and overflow

> **Phase 12 — Computer systems** · Lesson 2 of 15 · ⏱ about 135 minutes

## What you will learn

- Why unsigned arithmetic in a computer is "arithmetic on a clock" (modulo 2ⁿ).
- How negative numbers are stored with two's complement, and how to negate a number by hand.
- What happens to the bits when you convert between widths: sign extension, zero extension and truncation.
- The difference between a carry and a signed overflow, and the special trouble with `i32::MIN`.
- How to choose overflow behaviour for a whole program: `overflow-checks`, `Wrapping<T>`, `checked_*` and `try_from`.

## Before you start

- You finished [Lesson 12.1 — Bits, bytes and number systems](01-bits-bytes-and-number-systems.md).
- You know the integer types, the overflow panic in debug builds, wrapping in release builds and the methods `checked_add`, `wrapping_add`, `saturating_add` and `overflowing_add` from [Lesson 02.2 — Data types](../02-programming-basics/02-data-types.md). Here you learn **why** they behave like that, at the level of bits.
- Work in `code/p12-bits` from Lesson 12.1.

## New words

| Word | Meaning |
|---|---|
| **modular arithmetic** | Arithmetic "on a clock": after the largest value you continue from zero. Unsigned integers with *n* bits work modulo 2ⁿ. |
| **sign-magnitude** | An old idea for negative numbers: one bit for the sign, the other bits for the size. It is not used for integers today. |
| **two's complement** | The way every modern CPU stores signed integers: the top bit has a *negative* place value. |
| **sign bit** | The top bit of a signed integer. In two's complement it is `1` for every negative number. |
| **sign extension** | Making a signed number wider by copying its sign bit into the new high bits, so the value stays the same. |
| **zero extension** | Making an unsigned number wider by filling the new high bits with zeros. |
| **truncation** | Making a number narrower by throwing away the high bits. |
| **carry** | The bit that "falls out" on the left of an addition. For unsigned numbers, a carry means the result did not fit. |
| **signed overflow** | The true result of a signed operation is outside the range of the type, for example two positive numbers that add up to a "negative" bit pattern. |
| **flag** | A one-bit note that the CPU sets after an instruction, for example "there was a carry". On arm64 the flags are called N, Z, C and V. |
| **overflow check** | Extra code that the compiler adds to test for overflow and panic. On in debug builds, off in release builds by default. |

## 1. Unsigned numbers: arithmetic on a clock

A `u8` has 8 bits, so it has 256 patterns: 0 to 255. When an addition goes past 255, the ninth bit has no place to go and is lost. What stays is the remainder after division by 256. That is **modular arithmetic**, the same arithmetic as a clock: 10 o'clock plus 5 hours is 3 o'clock, not 15.

```rust
fn main() {
    let a: u8 = 250;
    println!("250 + 10 wraps to {}", a.wrapping_add(10));
    println!("(250 + 10) % 256 = {}", (250u32 + 10) % 256);
    println!("3 - 5 wraps to {}", 3u8.wrapping_sub(5));
    println!("(3 - 5 + 256) % 256 = {}", (3i32 - 5 + 256) % 256);
}
```

**What happens:**

1. `wrapping_add` does exactly what the hardware does: add, keep the lowest 8 bits.
2. The second line does the same in a wider type with `%` (remainder): the results agree.
3. Subtraction goes around the clock backwards: 3 − 5 gives 254, which is −2 + 256.

```text
250 + 10 wraps to 4
(250 + 10) % 256 = 4
3 - 5 wraps to 254
(3 - 5 + 256) % 256 = 254
```

Look at the last line again: "−2" and "254" are the **same bit pattern** in 8 bits. This is the key to negative numbers.

## 2. Two's complement: how negative numbers are stored

A first idea for negative numbers is **sign-magnitude**: use the top bit as a minus sign. Then `0000_0101` is 5 and `1000_0101` is −5. It has two problems: there are two zeros (`0000_0000` and `1000_0000`), and the normal addition circuit gives wrong answers, so the CPU would need a second circuit for signed numbers.

**Two's complement** solves both. The rule is short: **the top bit has a negative place value.** For `i8` the place values are:

```text
place value:  -128   64   32   16    8    4    2    1
-1:              1    1    1    1    1    1    1    1   = -128 + 127 = -1
-128:            1    0    0    0    0    0    0    0   = -128
127:             0    1    1    1    1    1    1    1   = 127
-5:              1    1    1    1    1    0    1    1   = -128 + 123 = -5
```

So the top bit is also the **sign bit**: it is `1` exactly when the number is negative. Let us see real bits:

```rust
fn main() {
    for x in [0i8, 1, 2, 127, -1, -2, -127, -128] {
        println!("{x:>4} = {:08b} (as u8: {:>3})", x, x as u8);
    }
    let five: i8 = 5;
    let inverted = !five;
    let negated = inverted.wrapping_add(1);
    println!("5        = {five:08b}");
    println!("!5       = {inverted:08b} = {inverted}");
    println!("!5 + 1   = {negated:08b} = {negated}");
}
```

**What happens:**

1. `{:08b}` on a signed number prints its real bits in two's complement. `x as u8` keeps the same bits and reads them as unsigned: −1 becomes 255, −128 becomes 128. These are the clock pairs from Section 1: a negative `x` has the same bits as `x + 256`.
2. `!five` flips every bit (Lesson 02.3). Flipping gives −6, not −5.
3. Adding 1 gives −5. This is the famous rule: **to negate a two's complement number, invert all bits and add 1.** (Why: `x + !x` is all ones, which is −1, so `!x = −x − 1`, and `!x + 1 = −x`.)

```text
   0 = 00000000 (as u8:   0)
   1 = 00000001 (as u8:   1)
   2 = 00000010 (as u8:   2)
 127 = 01111111 (as u8: 127)
  -1 = 11111111 (as u8: 255)
  -2 = 11111110 (as u8: 254)
-127 = 10000001 (as u8: 129)
-128 = 10000000 (as u8: 128)
5        = 00000101
!5       = 11111010 = -6
!5 + 1   = 11111011 = -5
```

### One adder for everything

The great advantage: signed and unsigned numbers are added by **the same circuit**, and subtraction is just addition of the negated number. The CPU does not need to know whether you think of the bits as `i8` or `u8`.

```rust
fn main() {
    let a: u8 = 0b0000_1100; // 12
    let b: u8 = 0b0000_0101; // 5
    let minus_b = (!b).wrapping_add(1);
    let sum = a.wrapping_add(minus_b);
    println!("a        = {a:08b} = {a}");
    println!("-b       = {minus_b:08b} (as i8: {})", minus_b as i8);
    println!("a + (-b) = {sum:08b} = {sum}");
    println!("a - b    = {}", a - b);
}
```

**What happens:**

1. `minus_b` is "invert and add 1" applied to 5. As an `i8`, those bits mean −5.
2. Adding `a` and `minus_b` with the ordinary (wrapping) addition gives 7, the same as `a - b`. The carry out of the top bit is simply dropped.

```text
a        = 00001100 = 12
-b       = 11111011 (as i8: -5)
a + (-b) = 00000111 = 7
a - b    = 7
```

The range of an *n*-bit signed type follows from the place values: from −2ⁿ⁻¹ (only the top bit set) to 2ⁿ⁻¹ − 1 (all bits except the top one). For `i8` that is −128 to 127. The range is **not symmetric**: there is one more negative number than positive numbers. Remember this; Section 4 shows why it matters.

## 3. Changing the width: extension and truncation

Lesson 02.2 showed that `as` never fails. Now you can see what it does with the bits:

```rust
fn main() {
    let small: i8 = -3;
    let byte: u8 = 0b1111_1101;
    println!("i8  -3       = {:08b}", small);
    println!("as i32       = {:032b} = {}", small as i32, small as i32);
    println!("u8 253       = {:08b}", byte);
    println!("as u32       = {:032b} = {}", byte as u32, byte as u32);
    println!("i8 as u8     = {}", small as u8);
    println!("u8 as i8     = {}", byte as i8);
    let big: i32 = 1000;
    println!("1000 as i8   = {} (1000 = {:016b})", big as i8, big);
    println!("i8::try_from(1000) = {:?}", i8::try_from(big));
    println!("i8::try_from(-3)   = {:?}", i8::try_from(-3i32));
}
```

**What happens:**

1. `small as i32` is **sign extension**: the sign bit `1` is copied into all 24 new bits, so the value stays −3.
2. `byte as u32` is **zero extension**: the new bits are zeros, so the value stays 253. The *source* type decides: the same 8 bits `11111101` grow into two different 32-bit patterns.
3. Between `i8` and `u8` (same width) nothing changes in the bits; only the meaning changes.
4. `1000 as i8` is **truncation**: only the lowest 8 bits `11101000` stay, and as an `i8` they mean −24. No error, no warning.
5. `i8::try_from` checks the range and returns a `Result`: `Ok` if the value fits, an error otherwise. (`TryFrom` is a standard trait; you can use it like this without knowing more.)

```text
i8  -3       = 11111101
as i32       = 11111111111111111111111111111101 = -3
u8 253       = 11111101
as u32       = 00000000000000000000000011111101 = 253
i8 as u8     = 253
u8 as i8     = -3
1000 as i8   = -24 (1000 = 0000001111101000)
i8::try_from(1000) = Err(TryFromIntError(PosOverflow))
i8::try_from(-3)   = Ok(-3)
```

> 📏 **Convention:** Use `as` for widening (it cannot lose information) and when you *want* the bits reinterpreted. For narrowing, use `try_from` (or `try_into`) and handle the error. Clippy has lints for risky casts, for example `clippy::cast_possible_truncation` (in the `pedantic` group), that you can turn on for systems code.

## 4. Carry and overflow: two different questions

After an addition, the CPU answers two questions at once and stores the answers in **flags** (on arm64 they are named C for carry and V for overflow; Lesson 12.7 shows them in assembly):

- **Carry:** did a bit fall out on the left? If you meant the numbers as *unsigned*, this means the result is wrong.
- **Overflow:** if you meant the numbers as *signed*, is the result wrong? This happens when two numbers of the same sign give a result of the other sign.

The same 8-bit addition can trigger one, both or none:

```rust
fn main() {
    let cases: [(i8, i8); 4] = [(100, 27), (100, 28), (-100, -29), (-1, 1)];
    for (a, b) in cases {
        let (signed, s_over) = a.overflowing_add(b);
        let (unsigned, u_over) = (a as u8).overflowing_add(b as u8);
        println!(
            "{a:>4} + {b:>4}: bits {:08b} + {:08b} = {:08b} | signed {signed:>4} overflow={s_over:<5} | unsigned {unsigned:>3} carry={u_over}",
            a as u8, b as u8, unsigned
        );
    }
}
```

**What happens:**

1. For each pair we add once as `i8` and once as `u8` (same bits). `overflowing_add` returns the wrapped result and a `bool`: for the signed add, that is the overflow flag; for the unsigned add, that is the carry.
2. `100 + 27 = 127` fits both ways.
3. `100 + 28`: two positive numbers give the bit pattern `10000000`, which as a signed number is −128. Signed overflow, but no carry (as unsigned, 128 is correct).
4. `-100 + -29`: two negative numbers give a positive pattern. Signed overflow **and** carry.
5. `-1 + 1 = 0` is correct as signed, but as unsigned it is 255 + 1, which carries. Carry without overflow.

```text
 100 +   27: bits 01100100 + 00011011 = 01111111 | signed  127 overflow=false | unsigned 127 carry=false
 100 +   28: bits 01100100 + 00011100 = 10000000 | signed -128 overflow=true  | unsigned 128 carry=false
-100 +  -29: bits 10011100 + 11100011 = 01111111 | signed  127 overflow=true  | unsigned 127 carry=true
  -1 +    1: bits 11111111 + 00000001 = 00000000 | signed    0 overflow=false | unsigned   0 carry=true
```

### The trouble with the minimum value

Because the signed range is not symmetric, the smallest value has no positive partner. `-i8::MIN` would be 128, which does not fit. Invert-and-add-one gives back `10000000`, the same number:

```rust
fn main() {
    let min = i8::MIN;
    println!("i8::MIN = {min}, bits {:08b}", min);
    println!("checked_abs:   {:?}", min.checked_abs());
    println!("wrapping_abs:  {}", min.wrapping_abs());
    println!("wrapping_neg:  {}", min.wrapping_neg());
    println!("unsigned_abs:  {}", min.unsigned_abs());
    println!("checked_div(-1): {:?}", min.checked_div(-1));
    println!("i32::MIN.checked_div(-1): {:?}", i32::MIN.checked_div(-1));
}
```

**What happens:**

1. `checked_abs` honestly says "no answer" (`None`).
2. `wrapping_abs` and `wrapping_neg` return −128 itself: the absolute value of a number is negative!
3. `unsigned_abs` returns a `u8`, which has room for 128. This is the correct way to get the size of any signed number.
4. Division can overflow too: `MIN / -1` would be `-MIN`. Rust panics on it even in release builds, because the CPU instruction for it fails on some processors; `checked_div` returns `None` instead.

```text
i8::MIN = -128, bits 10000000
checked_abs:   None
wrapping_abs:  -128
wrapping_neg:  -128
unsigned_abs:  128
checked_div(-1): None
i32::MIN.checked_div(-1): None
```

## 5. Choosing the overflow behaviour of a program

You have four tools. Choose on purpose:

| You want | Use |
|---|---|
| Overflow is a bug and must never go unnoticed | the default in debug; add `overflow-checks = true` to release builds |
| Overflow is expected and correct (hashes, checksums, counters that wrap) | `wrapping_*` or the type `std::num::Wrapping<T>` |
| Overflow is possible because input comes from outside | `checked_*` and return an error |
| Clamp at the limit (volume, scores) | `saturating_*` |

Here is the first row. A function that adds 1, called with a value that the compiler cannot know in advance (it depends on the number of command-line arguments, which is 1 when you give none):

```rust
fn add_one(x: u8) -> u8 {
    x + 1
}

fn main() {
    let input: u8 = std::env::args().count() as u8 + 254;
    println!("{}", add_one(input));
}
```

`cargo run` (debug) panics, `cargo run --release` silently prints the wrapped value:

```text

thread 'main' (5361232) panicked at src/main.rs:2:5:
attempt to add with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

```text
0
```

Add a release profile to `Cargo.toml` to keep the checks in release builds too:

```toml
[profile.release]
overflow-checks = true
```

Now `cargo run --release` panics like the debug build (`attempt to add with overflow` at `src/main.rs:2:5`). The checks cost a little speed, usually a few percent; many projects decide that correctness is worth it.

For the second row, `Wrapping<T>` makes wrapping the normal behaviour of `+`, `-` and `*`, so you cannot forget it. Here is a simple string hash called djb2 ("multiply by 33 and add the next byte"), where wrapping is part of the algorithm:

```rust
use std::num::Wrapping;

fn main() {
    let mut hash = Wrapping(5381u32);
    for byte in "hello".bytes() {
        hash = hash * Wrapping(33) + Wrapping(byte as u32);
    }
    println!("hash = {}", hash.0);
}
```

**What happens:**

1. `Wrapping(5381u32)` wraps the number in a small struct. Its `+` and `*` always wrap, in debug and in release.
2. `hash.0` takes the plain `u32` out again (`.0` is the first field of a tuple struct).

```text
hash = 261238937
```

> 🔒 **Security:** Overflow in a size calculation is a classic vulnerability. If `count * item_size` wraps to a small number, a program may reserve a small buffer and then write `count` items into it. In safe Rust the write is caught by bounds checks, but the program still panics (a crash that an attacker can trigger on purpose). Use `checked_mul` for every size that depends on input.

> 🧪 **Testing:** Test the borders: `0`, `1`, `MAX`, `MIN`, `MIN + 1`, `-1`. For `i8` and `u8` functions you can even test **every** possible input: there are only 256 values (or 65,536 pairs), and the loop takes milliseconds. Exercise 2 does this.

## Common mistakes

> ⚠️ **Common mistake: `abs()` of the minimum.** This function looks harmless:

```rust
fn distance_from_zero(x: i32) -> i32 {
    x.abs()
}

fn main() {
    println!("{}", distance_from_zero(-7));
    println!("{}", distance_from_zero(i32::MIN));
}
```

```text
7

thread 'main' (5362207) panicked at /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/core/src/num/mod.rs:433:5:
attempt to negate with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

The panic points into the standard library (where `abs` lives), not into your code; run with `RUST_BACKTRACE=1` to see the caller. In a release build it would silently return a negative number. Fix: return `u32` and use `x.unsigned_abs()`.

> ⚠️ **Common mistake: the average of two numbers.** `(lo + hi) / 2` overflows when both are large, even though the average itself fits:

```rust
fn average(lo: u32, hi: u32) -> u32 {
    (lo + hi) / 2
}

fn main() {
    let lo: u32 = 4_000_000_000;
    let hi: u32 = 4_100_000_000;
    println!("safe:     {}", lo + (hi - lo) / 2);
    println!("midpoint: {}", lo.midpoint(hi));
    println!("average:  {}", average(lo, hi));
}
```

```text
safe:     4050000000
midpoint: 4050000000

thread 'main' (5362431) panicked at src/main.rs:2:5:
attempt to add with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

This bug lived in many binary search implementations for years ([Lesson 07.10](../07-algorithms-and-data-structures/10-binary-search.md) used `lo + (hi - lo) / 2` for this reason). The standard library also has `midpoint`.

> ⚠️ **Common mistake: narrowing with `as`.** `len as u16` turns a length of 70,000 into 4,464 without a word. Use `u16::try_from(len)` and decide what to do with the error.

## Try it yourself

1. Predict the bits of `-1i16`, `-256i16` and `i16::MIN`, then print them with `{:016b}`.
2. Print `-3i8 as u16` and `-3i8 as u8 as u16`. They are different. Explain why with the rules of Section 3. (The first sign-extends to 16 bits, because the source is signed; the second first reinterprets as `u8`, then zero-extends.)
3. In `Cargo.toml`, add `overflow-checks = true` under `[profile.release]` and run the `add_one` example with `--release`. Then remove it again.

## Exercises

### Exercise 1 — Two's complement by hand (easy)

Write −5 as an `i8` in binary, using "invert and add 1". Then say which `i8` value the bits `1111_0110` mean. Check both with one `println!`.

<details>
<summary>Hint</summary>

5 is `0000_0101`. For the second part, either use the negative place value −128, or invert and add 1 to find the positive partner.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    println!("{:08b} {}", -5i8, 0b1111_0110u8 as i8);
}
```

```text
11111011 -10
```

Why it works: inverting `0000_0101` gives `1111_1010`, plus 1 gives `1111_1011`. For `1111_0110`: −128 + 64 + 32 + 16 + 4 + 2 = −10 (or: invert to `0000_1001` = 9, plus 1 = 10, so the value is −10).

</details>

### Exercise 2 — Detect overflow yourself (medium)

In `src/lib.rs`, write `pub fn add_with_flag(a: i8, b: i8) -> (i8, bool)` that returns the wrapped sum and `true` on signed overflow. Do not use `overflowing_add` or `checked_add` inside; use the rule from Section 4. Write a test that compares your function with `overflowing_add` for **all** 65,536 pairs.

<details>
<summary>Hint</summary>

Compute `a.wrapping_add(b)`. Overflow happens exactly when `a` and `b` have the same sign and the sum has a different sign. `x < 0` tells you the sign bit. Two nested `for` loops over `i8::MIN..=i8::MAX` give all pairs.

</details>

<details>
<summary>Answer</summary>

```rust
/// Adds two i8 values with wrapping and reports signed overflow,
/// using only the sign rule (no overflowing_add inside).
pub fn add_with_flag(a: i8, b: i8) -> (i8, bool) {
    let sum = a.wrapping_add(b);
    let same_sign_inputs = (a < 0) == (b < 0);
    let sign_changed = (sum < 0) != (a < 0);
    (sum, same_sign_inputs && sign_changed)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn matches_overflowing_add_for_every_pair() {
        for a in i8::MIN..=i8::MAX {
            for b in i8::MIN..=i8::MAX {
                assert_eq!(add_with_flag(a, b), a.overflowing_add(b), "a = {a}, b = {b}");
            }
        }
    }
}
```

Why it works: if the signs differ, the true sum lies between `a` and `b`, so it always fits. If the signs are equal and the result's sign differs, the true sum went past the end of the range and wrapped. The exhaustive test proves it for every input, not only for a few examples.

</details>

### Exercise 3 — A safe total (medium)

Write `pub fn checked_total(prices: &[u32]) -> Option<u32>` that adds all prices and returns `None` if the total does not fit in a `u32`. Write two tests: one where the total fits (including an empty slice and a total of exactly `u32::MAX`) and one where it overflows.

<details>
<summary>Hint</summary>

Use `checked_add` in a loop. The `?` operator works on `Option` too: on `None` it returns `None` from the function at once.

</details>

<details>
<summary>Answer</summary>

```rust
/// Sums all prices, or returns None if the total does not fit in a u32.
pub fn checked_total(prices: &[u32]) -> Option<u32> {
    let mut total: u32 = 0;
    for &price in prices {
        total = total.checked_add(price)?;
    }
    Some(total)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn total_fits() {
        assert_eq!(checked_total(&[]), Some(0));
        assert_eq!(checked_total(&[100, 250, 50]), Some(400));
        assert_eq!(checked_total(&[u32::MAX, 0]), Some(u32::MAX));
    }

    #[test]
    fn total_overflows() {
        assert_eq!(checked_total(&[u32::MAX, 1]), None);
        assert_eq!(checked_total(&[3_000_000_000, 2_000_000_000]), None);
    }
}
```

Why it works: `checked_add` returns `None` the moment a sum would not fit, and `?` passes it on. The same result is possible with an iterator: `prices.iter().try_fold(0u32, |acc, &p| acc.checked_add(p))`.

</details>

### Exercise 4 — Sign extension of an n-bit field (hard)

Machine instructions often contain small signed numbers, for example a 12-bit offset. Write `pub fn sign_extend(value: u32, bits: u32) -> i32` that treats the lowest `bits` bits of `value` as a two's complement number and ignores the rest. Tests: `(0x7ff, 12)` → 2047, `(0x800, 12)` → −2048, `(0xfff, 12)` → −1, `(0xabc_0001, 4)` → 1, `(0xffff_ffff, 32)` → −1.

<details>
<summary>Hint</summary>

Shift the field to the top of the 32 bits with `<<`, so its sign bit becomes bit 31. Convert to `i32` with `as` (same bits). Then shift right by the same amount. On a **signed** type, `>>` copies the sign bit into the new high bits (an *arithmetic shift*; Lesson 12.4 explains it), which is exactly sign extension.

</details>

<details>
<summary>Answer</summary>

```rust
/// Treats the lowest `bits` bits of `value` as a two's complement number.
pub fn sign_extend(value: u32, bits: u32) -> i32 {
    assert!((1..=32).contains(&bits), "bits must be from 1 to 32");
    let unused = 32 - bits;
    ((value << unused) as i32) >> unused
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn twelve_bit_values() {
        assert_eq!(sign_extend(0x7ff, 12), 2047);
        assert_eq!(sign_extend(0x800, 12), -2048);
        assert_eq!(sign_extend(0xfff, 12), -1);
        assert_eq!(sign_extend(0x0ff, 12), 255);
    }

    #[test]
    fn ignores_higher_bits_and_full_width() {
        assert_eq!(sign_extend(0xabc_0001, 4), 1);
        assert_eq!(sign_extend(0xffff_ffff, 32), -1);
        assert_eq!(sign_extend(0b1, 1), -1);
    }
}
```

Why it works: the left shift throws away the unwanted high bits and puts the field's sign bit at bit 31. The arithmetic right shift moves the field back and fills the space with copies of that sign bit. The `assert!` keeps `bits` in a range where the shifts are valid (shifting a `u32` by 32 or more is an overflow).

</details>

## Quiz

1. What is `200u8.wrapping_add(100)`, and why?

<details>
<summary>Answer</summary>

44, because 300 does not fit and 300 mod 256 = 44.

</details>

2. In two's complement, what is the place value of the top bit of an `i16`?

<details>
<summary>Answer</summary>

−2¹⁵ = −32,768.

</details>

3. How do you negate a two's complement number by hand?

<details>
<summary>Answer</summary>

Invert all bits and add 1.

</details>

4. Why is `i8::MIN.wrapping_abs()` equal to −128?

<details>
<summary>Answer</summary>

The positive value 128 does not fit into `i8` (maximum 127). Inverting `10000000` and adding 1 gives `10000000` again. Use `unsigned_abs` to get 128 as a `u8`.

</details>

5. `-1i8 as i32` and `255u8 as i32` start from the same 8 bits. Why are the results different?

<details>
<summary>Answer</summary>

The source type decides the extension: `i8` is sign-extended (new bits are copies of the sign bit `1`, value −1), `u8` is zero-extended (new bits are `0`, value 255).

</details>

6. An addition of two `i8` values set the carry but not the overflow flag. Is the signed result correct?

<details>
<summary>Answer</summary>

Yes. Carry only matters for unsigned numbers. For example `-1 + 1` carries, but 0 is the correct signed result.

</details>

7. What does `overflow-checks = true` under `[profile.release]` do?

<details>
<summary>Answer</summary>

It keeps the overflow checks in release builds, so overflow panics instead of wrapping silently.

</details>

## Summary

- Unsigned integers with *n* bits do arithmetic modulo 2ⁿ: the carry out of the top bit is lost.
- Signed integers use two's complement: the top bit has the place value −2ⁿ⁻¹, so it is also the sign bit. Negate by inverting and adding 1.
- The same addition circuit works for signed and unsigned numbers; only the interpretation of the result differs.
- Widening sign-extends signed values and zero-extends unsigned values; narrowing with `as` truncates silently, `try_from` checks.
- Carry means "wrong as unsigned", overflow means "wrong as signed". `MIN` has no positive partner, so `abs`, negation and `MIN / -1` can overflow.
- Choose the policy on purpose: `overflow-checks`, `Wrapping<T>`, `checked_*`, `saturating_*`; test the borders, and test small types exhaustively.

## Sources and further reading

- [Rust standard library: `i32`](https://doc.rust-lang.org/std/primitive.i32.html) — every `checked_*`, `wrapping_*`, `overflowing_*`, `unsigned_abs` and `midpoint` method.
- [Rust standard library: `std::num::Wrapping`](https://doc.rust-lang.org/std/num/struct.Wrapping.html) — the wrapping integer type.
- [The Cargo Book: Profiles](https://doc.rust-lang.org/cargo/reference/profiles.html) — `overflow-checks` and the other settings of debug and release builds.
- [The Rust Reference: Integer overflow](https://doc.rust-lang.org/reference/behavior-not-considered-unsafe.html) — the official rules: overflow is not undefined behaviour in Rust; it either panics or wraps.
- [Computer Systems: A Programmer's Perspective (CS:APP)](https://csapp.cs.cmu.edu/) — chapter 2.2–2.3 covers two's complement and integer arithmetic in depth.

---

[🏠 Start](../START-HERE.md) · [← 12.1 Bits, bytes and number systems](01-bits-bytes-and-number-systems.md) · [📋 Phase 12](README.md) · [12.3 Floating-point numbers →](03-floating-point-numbers.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
