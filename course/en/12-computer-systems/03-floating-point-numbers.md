<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 12.2 Integers in memory: two's complement and overflow](02-integers-in-memory.md) · [📋 Phase 12](README.md) · [12.4 Bitwise operations and bit tricks →](04-bitwise-operations.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/03-floating-point-numbers.md) <!-- nav -->

# Lesson 12.3 — Floating-point numbers

> **Phase 12 — Computer systems** · Lesson 3 of 15 · ⏱ about 120 minutes

## What you will learn

- How an `f32` or `f64` is built from three parts: a sign, an exponent and a fraction (the IEEE 754 standard).
- How to decode a float by hand and in code with `to_bits` and `from_bits`.
- Why `0.1` cannot be stored exactly, and why the gaps between floats grow with their size.
- The special values: positive and negative zero, infinity, NaN and subnormal numbers.
- How to compare, sort and sum floats correctly: tolerances, `total_cmp`, the order of additions and Kahan summation.

## Before you start

- You finished [Lesson 12.2 — Integers in memory](02-integers-in-memory.md).
- From [Lesson 02.2 — Data types](../02-programming-basics/02-data-types.md) you know `f32` and `f64`, that `0.1 + 0.2 != 0.3`, that dividing by zero gives `inf` or `NaN`, and that money must not be stored in floats. This lesson explains the reasons.
- Work in `code/p12-bits`.

## New words

| Word | Meaning |
|---|---|
| **floating point** | A way to store real numbers as "digits × 2 to some power". The point "floats": the same bits can store 0.000001 or 1,000,000. |
| **IEEE 754** | The standard (from the IEEE, Institute of Electrical and Electronics Engineers) that defines float formats and rules. `f32` is its *binary32* format, `f64` its *binary64* format. |
| **sign / exponent / fraction** | The three fields of a float: one bit for the sign, a power of two (the exponent), and the digits after the binary point (the fraction, also called *mantissa* or *significand*). |
| **bias** | A number added to the real exponent so that it can be stored without a sign: 127 for `f32`, 1023 for `f64`. |
| **hidden bit** | The leading `1.` of a normal float. It is always there, so it is not stored. |
| **ULP** (Unit in the Last Place) | The gap between a float and the next larger float. It grows with the size of the number. |
| **machine epsilon** | The ULP of 1.0: the smallest step you can add to 1.0. `f32::EPSILON` ≈ 1.19 × 10⁻⁷, `f64::EPSILON` ≈ 2.22 × 10⁻¹⁶. |
| **NaN** (Not a Number) | A special value for results that have no meaning, like 0/0. It is not equal to anything, not even itself. |
| **subnormal number** | A tiny float below the normal range, without the hidden bit. It lets numbers fade to zero gradually. |
| **rounding error** | The small difference between the exact result of an operation and the nearest float that the computer stores. |
| **tolerance** | How big a difference you accept when you compare two floats ("equal within 10⁻⁹"). |

## 1. Scientific notation, in binary

You may know scientific notation from school: the speed of light is 2.998 × 10⁸ m/s. A number is written as **a few digits** times **a power of ten**. This is how you can write both huge and tiny numbers with the same few digits.

A float does exactly this, but in base 2:

```text
value = (-1)^sign × 1.fraction × 2^exponent
```

First you need binary *fractions*. After the binary point, the place values are halves: ½, ¼, ⅛, 1/16, and so on.

```text
place value:   4   2   1  .  1/2  1/4  1/8
5.75 =         1   0   1  .   1    1    0     (4 + 1 + 0.5 + 0.25)
```

So 5.75 is `101.11` in binary. Move the point so that exactly one `1` is in front of it, like in scientific notation: `101.11` = `1.0111 × 2²`. This is the **normal form**: the part before the point is always `1`, so a float does not even store it (the **hidden bit**). It stores only the sign, the exponent 2 and the fraction bits `0111…`.

### The layout of an `f32`

```text
 bit 31   bits 30..23        bits 22..0
┌──────┬──────────────┬───────────────────────────┐
│ sign │ exponent (8) │       fraction (23)       │
└──────┴──────────────┴───────────────────────────┘
value = (-1)^sign × 1.fraction × 2^(exponent - 127)
```

The exponent field is stored with a **bias** of 127: the stored number is the real exponent + 127, so that negative exponents need no sign. For 5.75 the stored exponent is 2 + 127 = 129. An `f64` has the same design with 1 sign bit, 11 exponent bits (bias 1023) and 52 fraction bits.

Let us check this on real values. `to_bits()` returns the 32 bits of an `f32` as a `u32`. To cut the fields out we use a shift (`>>`) and a **mask** with `&`: `x & 0xff` keeps only the lowest 8 bits. Lesson 12.4 teaches masks in detail; here it is enough to read them as "cut out these bits".

```rust
fn show(x: f32) {
    let bits = x.to_bits();
    let sign = bits >> 31;
    let exponent = (bits >> 23) & 0xff;
    let fraction = bits & 0x7f_ffff;
    println!(
        "{x:>10}: {sign} {exponent:08b} {fraction:023b}  (exponent {exponent} - 127 = {})",
        exponent as i32 - 127
    );
}

fn main() {
    for x in [1.0f32, 2.0, 5.75, -5.75, 0.1, 0.5] {
        show(x);
    }
}
```

**What happens:**

1. `bits >> 31` moves the top bit to the bottom: the sign.
2. `(bits >> 23) & 0xff` moves the exponent field to the bottom and keeps its 8 bits.
3. `bits & 0x7f_ffff` keeps the lowest 23 bits (`0x7fffff` is 23 ones): the fraction.

```text
         1: 0 01111111 00000000000000000000000  (exponent 127 - 127 = 0)
         2: 0 10000000 00000000000000000000000  (exponent 128 - 127 = 1)
      5.75: 0 10000001 01110000000000000000000  (exponent 129 - 127 = 2)
     -5.75: 1 10000001 01110000000000000000000  (exponent 129 - 127 = 2)
       0.1: 0 01111011 10011001100110011001101  (exponent 123 - 127 = -4)
       0.5: 0 01111110 00000000000000000000000  (exponent 126 - 127 = -1)
```

Read the output like this:

- 1.0 is `1.0 × 2⁰`: fraction all zeros, exponent 0. 2.0 differs only in the exponent: `1.0 × 2¹`.
- 5.75 has exactly the fraction `0111` that we found by hand. −5.75 differs only in the sign bit. (Negative floats do **not** use two's complement; the sign is a separate bit.)
- 0.1 has a fraction that repeats `1001 1001 1001 …` and ends in `…1101`. That last part is rounded. The next section explains why.

## 2. Why 0.1 is not exact

In decimal, 1/3 = 0.3333… never ends. In binary, the same happens with 1/10: 0.1 is `0.000110011001100110011…` forever, because 1/10 is not a sum of a few halves, quarters and eighths. A float has only 23 (or 52) fraction bits, so the computer stores the **nearest** number it can. Print more digits than usual and you see it:

```rust
fn main() {
    println!("0.1f32 is really {:.30}", 0.1f32);
    println!("0.1f64 is really {:.30}", 0.1f64);
    println!("0.5f64 is really {:.30}", 0.5f64);
    let mut sum = 0.0f64;
    for _ in 0..10 {
        sum += 0.1;
    }
    println!("0.1 added ten times = {sum}");
    println!("{:.20}", 0.1 + 0.2);
}
```

**What happens:**

1. `{:.30}` prints 30 digits after the point. The `f32` value is wrong in the 9th digit, the `f64` value in the 18th.
2. 0.5 is ½, a single binary digit, so it is stored exactly. Numbers made of halves, quarters, eighths… (0.25, 0.375, 5.75) are exact; most "nice" decimal numbers are not.
3. Each addition rounds a little; ten small errors add up to a visible difference.

```text
0.1f32 is really 0.100000001490116119384765625000
0.1f64 is really 0.100000000000000005551115123126
0.5f64 is really 0.500000000000000000000000000000
0.1 added ten times = 0.9999999999999999
0.30000000000000004441
```

> ⚠️ **Common mistake:** Thinking that Rust prints the "real" value. By default, `{}` prints the **shortest** decimal text that turns back into the same float. That is why `0.1` prints as `0.1`, even though the stored value is not exactly 0.1.

### The gaps grow with the size

A float has a fixed number of fraction bits. For numbers between 1 and 2 they describe steps of 2⁻²³; between 2 and 4 the same bits describe steps twice as big, and so on. So the **gap between neighbouring floats (the ULP) grows with the number**:

```rust
fn main() {
    let big = 16_777_216.0f32; // 2^24
    println!("{} + 1 = {}", big, big + 1.0);
    println!("{} + 2 = {}", big, big + 2.0);
    println!("16777217 as f32 = {}", 16_777_217u32 as f32);
    let huge = 9_007_199_254_740_992.0f64; // 2^53
    println!("{} + 1 = {}", huge, huge + 1.0);

    for x in [1.0f64, 1000.0, 1_000_000.0, 1e15] {
        println!("gap after {x:>16}: {:e}", x.next_up() - x);
    }
    println!("f32::EPSILON = {:e}, f64::EPSILON = {:e}", f32::EPSILON, f64::EPSILON);
}
```

**What happens:**

1. From 2²⁴ = 16,777,216 on, an `f32` can only store **even** whole numbers: the gap is 2. Adding 1 is rounded away. The integer 16,777,217 has no `f32` at all.
2. For `f64` the same happens at 2⁵³: up to there, every whole number is exact; above it, not.
3. `next_up()` returns the next larger float. The gap after 1.0 is the **machine epsilon**; after 10¹⁵ it is already 0.125.

```text
16777216 + 1 = 16777216
16777216 + 2 = 16777218
16777217 as f32 = 16777216
9007199254740992 + 1 = 9007199254740992
gap after                1: 2.220446049250313e-16
gap after             1000: 1.1368683772161603e-13
gap after          1000000: 1.1641532182693481e-10
gap after 1000000000000000: 1.25e-1
f32::EPSILON = 1.1920929e-7, f64::EPSILON = 2.220446049250313e-16
```

> ⚠️ **Common mistake:** Using `f32` for a counter, an ID or a timestamp. Above 16,777,216 an `f32` counter stops counting: `x + 1.0 == x`. Use integers for anything you count.

## 3. Special values

Two exponent patterns are reserved. **All zeros** means zero or a *subnormal* number. **All ones** means infinity or NaN.

```rust
fn main() {
    let values = [0.0f32, -0.0, f32::INFINITY, f32::NEG_INFINITY, f32::NAN, f32::MIN_POSITIVE, f32::from_bits(1), f32::MAX];
    for x in values {
        println!("{x:>14e}  bits {:#010x}  normal? {:<5} nan? {}", x.to_bits(), x.is_normal(), x.is_nan());
    }
    println!("0.0 == -0.0 : {}", 0.0f32 == -0.0);
    let nan = f32::NAN;
    println!("nan == nan  : {}", nan == nan);
    println!("1/-0.0      : {}", 1.0f32 / -0.0);
    println!("f32::MAX * 2: {}", f32::MAX * 2.0);
    println!("nan as i32 = {}, 1e20 as i32 = {}, -1e20 as u8 = {}", nan as i32, 1e20f32 as i32, -1e20f32 as u8);
}
```

**What happens:**

1. **Zero** is all zero bits. **Negative zero** has only the sign bit set (`0x80000000`). They compare as equal, but they are different bits, and `1 / -0.0` gives `-inf`.
2. **Infinity** has the exponent all ones and fraction zero (`0x7f800000`). It appears when a result is too big: `f32::MAX * 2.0`.
3. **NaN** has the exponent all ones and a non-zero fraction. `nan == nan` is `false`: by the standard, NaN is not equal to anything. Use `is_nan()` to test for it.
4. `f32::MIN_POSITIVE` (about 1.18 × 10⁻³⁸) is the smallest **normal** float. Below it come the **subnormal** numbers: exponent field 0, no hidden bit. `from_bits(1)`, the smallest of them, is about 1.4 × 10⁻⁴⁵ (printed as `1e-45`, the shortest text that gives back the same bits).
5. Converting a float to an integer with `as` is defined in Rust: it **saturates** at the integer's limits, and NaN becomes 0. It never wraps and never crashes.

```text
           0e0  bits 0x00000000  normal? false nan? false
          -0e0  bits 0x80000000  normal? false nan? false
           inf  bits 0x7f800000  normal? false nan? false
          -inf  bits 0xff800000  normal? false nan? false
           NaN  bits 0x7fc00000  normal? false nan? true
 1.1754944e-38  bits 0x00800000  normal? true  nan? false
         1e-45  bits 0x00000001  normal? false nan? false
  3.4028235e38  bits 0x7f7fffff  normal? true  nan? false
0.0 == -0.0 : true
nan == nan  : false
1/-0.0      : -inf
f32::MAX * 2: inf
nan as i32 = 0, 1e20 as i32 = 2147483647, -1e20 as u8 = 0
```

Because NaN is not equal to itself, floats break a rule that the trait `Eq` promises ("every value equals itself"). That is why `f32` and `f64` implement only `PartialEq` and `PartialOrd`, not `Eq` and `Ord`. You will feel this when you sort (see "Common mistakes").

> 🔒 **Security:** A NaN from bad input spreads silently: every calculation with NaN gives NaN, and every comparison with it is `false`. A check like `if amount > limit { reject }` lets a NaN through. When you parse floats from outside, reject NaN and infinity with `x.is_finite()`.

## 4. Rounding: order matters

Every float operation rounds its exact result to the nearest float. One rounding error is tiny, but it has a strange effect: **the order of additions changes the result**. In normal mathematics (a + b) + c = a + (b + c). For floats it is not always true:

```rust
fn main() {
    let a = 1e16f64;
    let b = -1e16f64;
    let c = 1.0f64;
    println!("(a + b) + c = {}", (a + b) + c);
    println!("a + (b + c) = {}", a + (b + c));

    let mut v = vec![2.5f64, -1.0, f64::NAN, 0.0, -0.0, 10.0];
    v.sort_by(|x, y| x.total_cmp(y));
    println!("sorted with total_cmp: {v:?}");
    println!("max of 1.0 and NaN: {}", 1.0f64.max(f64::NAN));
}
```

**What happens:**

1. `(a + b) + c`: `a + b` is exactly 0, then + 1 gives 1.
2. `a + (b + c)`: near 10¹⁶ the gap between floats is 2, so `b + c` = −10¹⁶ + 1 rounds back to −10¹⁶. The 1 is lost, and the result is 0.
3. `total_cmp` is a **total order** for floats: it puts every value, even NaN and −0.0, in a fixed place (−0.0 before 0.0, NaN at the end for a positive NaN). Use it for sorting.
4. `max` and `min` ignore a NaN argument and return the other number.

```text
(a + b) + c = 1
a + (b + c) = 0
sorted with total_cmp: [-1.0, -0.0, 0.0, 2.5, 10.0, NaN]
max of 1.0 and NaN: 1
```

Because of rounding, the compiler is **not** allowed to reorder float additions for speed (it would change your results). You will see the effect of this on speed in Lesson 12.6.

> ⚡ **Performance:** `f64` is the default for a good reason: on your Mac it is about as fast as `f32` for single values. `f32` is worth it when you have millions of values (it halves the memory and doubles how many values fit into a cache line and into a SIMD register — SIMD, Single Instruction, Multiple Data, means one instruction that works on several values at once; Lessons 12.9 and 19.7 come back to this), and when the lower precision is acceptable.

> 🧪 **Testing:** Never use `assert_eq!` on computed floats. Compare with a tolerance (Exercise 3). Test the special values explicitly: `0.0`, `-0.0`, `NaN`, `INFINITY`, a subnormal, and a very large number. And for code that turns floats into bits and back, a round-trip test with `to_bits()` compares the exact bits, so it is safe to use `assert_eq!` there.

## Common mistakes

> ⚠️ **Common mistake: sorting floats like integers.** `sort()` needs `Ord`, which floats do not have:

```rust
fn main() {
    let mut v = vec![2.5f64, 1.0];
    v.sort();
    println!("{v:?}");
}
```

```text
error[E0277]: the trait bound `f64: Ord` is not satisfied
 --> src/main.rs:3:7
  |
3 |     v.sort();
  |       ^^^^ the trait `Ord` is not implemented for `f64`
```

A common "fix" is `sort_by(|a, b| a.partial_cmp(b).unwrap())`. It compiles, but it panics as soon as the data contains a NaN, because `partial_cmp` returns `None` for NaN:

```rust
fn main() {
    let mut v = vec![2.5f64, f64::NAN, 1.0];
    v.sort_by(|a, b| a.partial_cmp(b).unwrap());
    println!("{v:?}");
}
```

```text

thread 'main' (5505498) panicked at src/main.rs:3:39:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

The real fix: `v.sort_by(|a, b| a.total_cmp(b))`.

> ⚠️ **Common mistake: an absolute tolerance for all sizes.** `(a - b).abs() < 1e-9` works for numbers near 1, but for numbers near 10²⁰ the gap between floats is far bigger than 10⁻⁹, so two "equal" results never pass. For numbers of any size, use a **relative** tolerance (Exercise 3).

> ⚠️ **Common mistake: money in floats.** Not even 0.10 dollars can be stored exactly. Keep money as an integer number of the smallest unit (cents, `i64`).

## Try it yourself

1. Use the `show` function from Section 1 on `0.25`, `0.75`, `3.0` and `-0.0`. Predict the three fields first.
2. Print `f64::MAX`, `f64::MIN_POSITIVE` and `f64::EPSILON` with `{:e}` (with `{}` the first two print hundreds of digits). Compare them with the `f32` values above.
3. Find the first whole number that an `f32` cannot store, with a loop: start at `0u32`, and stop at the first `n` where `(n as f32) as u32 != n`. (You should get 16,777,217.)

## Exercises

### Exercise 1 — Decode by hand (easy)

Which number do the `f32` bits `0x40a00000` mean? And which bits does −2.5 have? Work it out on paper, then check with `f32::from_bits` and `to_bits`.

<details>
<summary>Hint</summary>

`0x40a00000` in binary is `0100 0000 1010 0000 …`. Split it into 1 + 8 + 23 bits. For −2.5: 2.5 = `10.1` in binary = `1.01 × 2¹`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    println!("{}", f32::from_bits(0x40a0_0000));
    println!("{:#010x}", (-2.5f32).to_bits());
}
```

```text
5
0xc0200000
```

Why it works: `0x40a00000` = sign `0`, exponent `10000001` = 129 → 2, fraction `0100…` → `1.01₂ × 2²` = 1.25 × 4 = 5. For −2.5: sign `1`, exponent 1 + 127 = 128 = `10000000`, fraction `0100…`; the bits `1100 0000 0010 0000 …` are `0xc0200000`.

</details>

### Exercise 2 — Decode and encode in code (medium)

In `src/lib.rs`, write `pub fn decode_f32(x: f32) -> (u32, i32, u32)` that returns the sign bit, the **unbiased** exponent and the 23 fraction bits. Write `pub fn encode_f32(sign: u32, exponent: i32, fraction: u32) -> f32` that does the reverse for normal numbers. Test known values (1.0, 5.75, −0.5) and a round trip for several values.

<details>
<summary>Hint</summary>

Use the shifts and masks of Section 1 and subtract the bias 127. To encode, add the bias, shift each part back into place with `<<`, combine them with `|` (bitwise OR) and call `f32::from_bits`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Splits an f32 into its sign bit, unbiased exponent and 23 fraction bits.
pub fn decode_f32(x: f32) -> (u32, i32, u32) {
    let bits = x.to_bits();
    let sign = bits >> 31;
    let exponent = ((bits >> 23) & 0xff) as i32 - 127;
    let fraction = bits & 0x7f_ffff;
    (sign, exponent, fraction)
}

/// Builds a normal f32 back from its parts: (-1)^sign * 1.fraction * 2^exponent.
pub fn encode_f32(sign: u32, exponent: i32, fraction: u32) -> f32 {
    let biased = (exponent + 127) as u32;
    f32::from_bits((sign << 31) | (biased << 23) | fraction)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn decodes_known_values() {
        assert_eq!(decode_f32(1.0), (0, 0, 0));
        assert_eq!(decode_f32(5.75), (0, 2, 0b0111 << 19));
        assert_eq!(decode_f32(-0.5), (1, -1, 0));
    }

    #[test]
    fn round_trip_for_many_values() {
        for x in [1.0f32, -2.5, 0.1, 3.14159, 1e30, -1e-30, f32::MAX] {
            let (s, e, f) = decode_f32(x);
            assert_eq!(encode_f32(s, e, f), x, "value {x}");
        }
    }
}
```

Why it works: the fields do not overlap, so `|` simply puts them next to each other. For 5.75 the fraction `0111` sits at the top of the 23 bits, which is `0b0111 << 19`. Comparing round-tripped floats with `assert_eq!` is fine here, because no arithmetic happened: the bits are identical.

</details>

### Exercise 3 — Compare with a relative tolerance (medium)

Write `pub fn approx_eq(a: f64, b: f64, rel: f64) -> bool` that returns `true` when the difference is at most `rel` times the larger of `|a|` and `|b|`. Tests: `0.1 + 0.2` ≈ `0.3`; `1e20 + 1e5` ≈ `1e20`; `1.0` is not ≈ `1.001`; `0.0` ≈ `-0.0`; NaN is not ≈ NaN (all with `rel = 1e-12`).

<details>
<summary>Hint</summary>

Handle `a == b` first (it covers `0.0` and `-0.0`, and infinities of the same sign). Then compare `(a - b).abs()` with `a.abs().max(b.abs()) * rel`. Any comparison with NaN is `false`, so the NaN case works by itself.

</details>

<details>
<summary>Answer</summary>

```rust
/// True if a and b differ by at most `rel` times the larger of their sizes.
pub fn approx_eq(a: f64, b: f64, rel: f64) -> bool {
    if a == b {
        return true;
    }
    let diff = (a - b).abs();
    let largest = a.abs().max(b.abs());
    diff <= largest * rel
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn approx_eq_cases() {
        assert!(approx_eq(0.1 + 0.2, 0.3, 1e-12));
        assert!(approx_eq(1e20 + 1e5, 1e20, 1e-12));
        assert!(!approx_eq(1.0, 1.001, 1e-12));
        assert!(approx_eq(0.0, -0.0, 1e-12));
        assert!(!approx_eq(f64::NAN, f64::NAN, 1e-12));
    }
}
```

Why it works: a relative tolerance scales with the size of the numbers, just like the gaps between floats do. Note one limit: comparing with exactly `0.0` (`approx_eq(1e-300, 0.0, …)`) is `false`, because the relative size of the difference is 100%. Near zero you need an extra absolute tolerance.

</details>

### Exercise 4 — Kahan summation (hard)

Adding 0.1 ten million times should give 1,000,000, but the simple sum is wrong by about 10⁻⁴. Kahan summation keeps the rounding error of each step in a second variable and adds it back. Write `pub fn kahan_sum(values: &[f64]) -> f64` and a test that shows it is more accurate than `values.iter().sum()` and within 10⁻⁶ of the exact answer.

<details>
<summary>Hint</summary>

Keep `sum` and `lost` (the error so far, starting at 0). For each `x`: `y = x - lost` (correct the new value), `t = sum + y` (the rounded new sum), `lost = (t - sum) - y` (what the rounding threw away), `sum = t`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Kahan summation: keeps the rounding error of each addition and adds it back.
pub fn kahan_sum(values: &[f64]) -> f64 {
    let mut sum = 0.0;
    let mut lost = 0.0;
    for &x in values {
        let y = x - lost;
        let t = sum + y;
        lost = (t - sum) - y;
        sum = t;
    }
    sum
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn kahan_is_more_accurate() {
        let values = vec![0.1; 10_000_000];
        let naive: f64 = values.iter().sum();
        let careful = kahan_sum(&values);
        let exact = 1_000_000.0;
        println!("naive error = {:e}, kahan error = {:e}", (naive - exact).abs(), (careful - exact).abs());
        assert!((careful - exact).abs() < (naive - exact).abs());
        assert!((careful - exact).abs() < 1e-6);
    }
}
```

With `cargo test kahan -- --nocapture` you see the printed line:

```text
naive error = 1.610246254131198e-4, kahan error = 0e0
```

Why it works: `(t - sum)` is the part of `y` that really got into the sum; subtracting `y` leaves the part that was rounded away. The next step adds it back. This only works because the compiler keeps your order of float operations exactly as written (Section 4); with reordering, `lost` would always be 0.

</details>

## Quiz

1. What are the three fields of an `f32`, and how many bits does each have?

<details>
<summary>Answer</summary>

Sign (1 bit), exponent (8 bits, bias 127), fraction (23 bits). An `f64` has 1, 11 (bias 1023) and 52.

</details>

2. Why does a normal float not store the `1` before the binary point?

<details>
<summary>Answer</summary>

In normal form it is always `1`, so storing it would waste a bit. This "hidden bit" gives one extra bit of precision for free.

</details>

3. Why is 0.5 stored exactly, but 0.1 is not?

<details>
<summary>Answer</summary>

0.5 is ½, a single binary place. 0.1 is an endless repeating binary fraction (`0.000110011…`), so it must be rounded to the nearest float.

</details>

4. What is `16_777_216.0f32 + 1.0`, and why?

<details>
<summary>Answer</summary>

`16777216`. At 2²⁴ the gap between `f32` values is 2, so the exact result 16,777,217 is rounded to a neighbour.

</details>

5. Why do `f64` values not implement `Ord`?

<details>
<summary>Answer</summary>

Because of NaN: it is not equal to itself and not ordered with any value, so a total order with `==` and `<` is impossible. Use `total_cmp` for sorting.

</details>

6. What does `f64::NAN as i32` give in Rust, and what does `1e20 as i32` give?

<details>
<summary>Answer</summary>

0 and `i32::MAX` (2147483647). Float-to-integer `as` saturates at the limits and turns NaN into 0.

</details>

7. Why can `(a + b) + c` and `a + (b + c)` differ for floats?

<details>
<summary>Answer</summary>

Each addition rounds its result. In a different order, different intermediate results are rounded, so a small value can be lost in one order and kept in the other.

</details>

## Summary

- A float is `(-1)^sign × 1.fraction × 2^(exponent − bias)`: `f32` has 1 + 8 + 23 bits (bias 127), `f64` has 1 + 11 + 52 bits (bias 1023).
- Only sums of powers of two are exact; 0.1 is rounded. `{}` prints the shortest text that gives back the same float, not the exact stored value.
- The gap between floats (ULP) grows with the number: `f32` stores every whole number only up to 2²⁴, `f64` up to 2⁵³.
- Special values: ±0, ±infinity, NaN (not equal to itself) and subnormal numbers. Float-to-integer `as` saturates, NaN becomes 0.
- Every operation rounds, so the order of operations matters; the compiler keeps your order.
- Compare with a relative tolerance, sort with `total_cmp`, reject NaN from input with `is_finite`, and keep money and counters in integers.

## Sources and further reading

- [Rust standard library: `f32`](https://doc.rust-lang.org/std/primitive.f32.html) — `to_bits`, `from_bits`, `next_up`, `total_cmp`, `is_finite`, `EPSILON`, `MIN_POSITIVE` and the other constants.
- [The Rust Reference: Numeric cast](https://doc.rust-lang.org/reference/expressions/operator-expr.html) — the exact rules for `as` between floats and integers (section "Type cast expressions").
- [The Floating-Point Guide](https://floating-point-gui.de/) — a short, friendly site about float problems and how to compare floats.
- [IEEE 754 (IEEE Standards Association)](https://standards.ieee.org/ieee/754/6210/) — the page of the standard itself (the full text is paid; you do not need it).
- [Computer Systems: A Programmer's Perspective (CS:APP)](https://csapp.cs.cmu.edu/) — chapter 2.4 explains floating point with many examples.

---

[🏠 Start](../START-HERE.md) · [← 12.2 Integers in memory: two's complement and overflow](02-integers-in-memory.md) · [📋 Phase 12](README.md) · [12.4 Bitwise operations and bit tricks →](04-bitwise-operations.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
