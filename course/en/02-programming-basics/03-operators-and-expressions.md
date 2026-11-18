<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 02.2 Data types](02-data-types.md) · [📋 Phase 02](README.md) · [02.4 Functions →](04-functions.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/02-programming-basics/03-operators-and-expressions.md) <!-- nav -->

# Lesson 02.3 — Operators and expressions

> **Phase 02 — Programming basics in Rust** · Lesson 3 of 11 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How to calculate with arithmetic operators, and why `7 / 2` is `3`.
- How to compare values and combine conditions with `&&`, `||` and `!`.
- What compound assignment (`+=`) and bitwise operators (`&`, `|`, `<<`) do.
- The order in which operators are evaluated (precedence).
- The difference between an **expression** and a **statement**, and how a block `{ }` gives a value.

## Before you start

- You finished [Lesson 02.2 — Data types](02-data-types.md). You know integers, floats, `bool` and `as`.
- Put the examples of this lesson into `src/bin/` of `~/projects/rust/code/p02-basics`.

## New words

| Word | Meaning |
|---|---|
| **operator** | A symbol that does a calculation or a comparison: `+`, `==`, `&&`. |
| **operand** | A value that an operator works on. In `3 + 4`, the operands are `3` and `4`. |
| **expression** | A piece of code that produces a value: `3 + 4`, `x`, `{ 5 }`. |
| **statement** | An instruction that does something but produces no value. It usually ends with `;`. |
| **precedence** | The rules that say which operator is evaluated first (`*` before `+`). |
| **associativity** | The direction in which operators of the same precedence are evaluated (`10 - 4 - 3` goes left to right). |
| **remainder** | What is left after integer division: `7 % 2` is `1`. |
| **truncate** | Cut off the fractional part, without rounding: `3.9` becomes `3`. |
| **comparison operator** | An operator that compares two values and gives `bool`: `==`, `<`. |
| **logical operator** | An operator that combines `bool` values: `&&` (and), `||` (or), `!` (not). |
| **short-circuit** | `&&` and `||` do not evaluate the right side if the left side already decides the answer. |
| **compound assignment** | An operator like `+=` that changes a variable using its own value. |
| **bitwise operator** | An operator that works on the individual bits of an integer: `&`, `|`, `^`, `!`, `<<`, `>>`. |
| **method** | A function attached to a value with a dot: `x.abs()`. |
| **block expression** | Code in `{ }` whose last expression, written without `;`, is the value of the whole block. |
| **unit** | The value `()` — "nothing". A statement-like block returns it. |

## 1. Arithmetic

Rust has the usual five arithmetic operators:

| Operator | Meaning | Example |
|---|---|---|
| `+` | add | `7 + 2` is `9` |
| `-` | subtract | `7 - 2` is `5` |
| `*` | multiply | `7 * 2` is `14` |
| `/` | divide | `7 / 2` is `3` |
| `%` | remainder | `7 % 2` is `1` |

The parts of `7 + 2` are two **operands** (`7` and `2`) and one **operator** (`+`).

```rust
fn main() {
    println!("7 + 2  = {}", 7 + 2);
    println!("7 - 2  = {}", 7 - 2);
    println!("7 * 2  = {}", 7 * 2);
    println!("7 / 2  = {}", 7 / 2);
    println!("7 % 2  = {}", 7 % 2);
    println!("-7 / 2 = {}", -7 / 2);
    println!("-7 % 2 = {}", -7 % 2);
    println!("7.0 / 2.0 = {}", 7.0 / 2.0);
    println!("7 as f64 / 2 as f64 = {}", 7 as f64 / 2 as f64);
}
```

What happens:

1. Lines 1–3 are ordinary school arithmetic.
2. `7 / 2` is **integer division**: when both numbers are integers, the result is an integer too. The fractional part `.5` is cut off (**truncated**), so the answer is `3`, not `3.5`.
3. `7 % 2` is the **remainder** of that division: `7 = 3 * 2 + 1`, so the remainder is `1`. The remainder is very useful: `n % 2 == 0` tests whether `n` is even, and `n % 10` is the last digit of `n`.
4. With negative numbers, the result of `/` is truncated towards zero (`-7 / 2` is `-3`), and the remainder takes the sign of the left number (`-7 % 2` is `-1`).
5. `7.0 / 2.0` divides two floats and gives `3.5`.
6. `7 as f64 / 2 as f64` converts the integers to floats first (`as`, Lesson 02.2), so the result is `3.5`.

```text
7 + 2  = 9
7 - 2  = 5
7 * 2  = 14
7 / 2  = 3
7 % 2  = 1
-7 / 2 = -3
-7 % 2 = -1
7.0 / 2.0 = 3.5
7 as f64 / 2 as f64 = 3.5
```

There is a useful rule that always holds for integers: `(a / b) * b + (a % b) == a`.

Both operands must have **the same type**. You cannot write `5 + 2.5` (integer plus float); convert one with `as` or write `5.0 + 2.5`.

### Division by zero and overflow

Dividing an integer by zero is a mistake that Rust catches. If the compiler can see it, the program does not build; otherwise it panics at run time:

```rust
fn divide(a: i32, b: i32) -> i32 {
    a / b
}

fn main() {
    println!("10 / 2 = {}", divide(10, 2));
    println!("10 / 0 = {}", divide(10, 0));
}
```

(`divide` is a function; Lesson 02.4 teaches them.)

```text
10 / 2 = 5

thread 'main' (3928807) panicked at src/main.rs:2:5:
attempt to divide by zero
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Unsigned numbers cannot go below zero. Subtracting a bigger number from a smaller one gives an overflow error (some people call it *underflow*, but Rust uses the word "overflow" in the message):

```rust
fn subtract(a: u32, b: u32) -> u32 {
    a - b
}

fn main() {
    println!("6 - 5 = {}", subtract(6, 5));
    println!("5 - 6 = {}", subtract(5, 6));
}
```

```text
6 - 5 = 1

thread 'main' (3929655) panicked at src/main.rs:2:5:
attempt to subtract with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Remember from Lesson 02.2: this panic happens in debug builds; release builds wrap around. If a subtraction can go below zero, use `checked_sub`, or `abs_diff` when you only need the distance between two numbers.

> 🔒 **Security:** `a - b` on unsigned numbers is a classic bug when `b` comes from a user: a "length minus header size" that wraps to a huge number. Check `b <= a` first, or use `checked_sub`.

### Useful methods on numbers

Many calculations are **methods**: functions that you call with a dot after the value.

```rust
fn main() {
    println!("2^10          = {}", 2_i32.pow(10));
    println!("2.0^10        = {}", 2.0_f64.powi(10));
    println!("sqrt(2)       = {}", 2.0_f64.sqrt());
    println!("abs(-3.5)     = {}", (-3.5_f64).abs());
    println!("abs(-4)       = {}", (-4_i32).abs());
    println!("round(2.5)    = {}", 2.5_f64.round());
    println!("floor(2.7)    = {}", 2.7_f64.floor());
    println!("ceil(2.1)     = {}", 2.1_f64.ceil());
    println!("min(3, 8)     = {}", 3.min(8));
    println!("max(3, 8)     = {}", 3.max(8));
    println!("clamp(15,0,10) = {}", 15.clamp(0, 10));
    println!("abs_diff(3,10) = {}", 3_u32.abs_diff(10));
}
```

```text
2^10          = 1024
2.0^10        = 1024
sqrt(2)       = 1.4142135623730951
abs(-3.5)     = 3.5
abs(-4)       = 4
round(2.5)    = 3
floor(2.7)    = 2
ceil(2.1)     = 3
min(3, 8)     = 3
max(3, 8)     = 8
clamp(15,0,10) = 10
abs_diff(3,10) = 7
```

What happens: `pow` raises an integer to a power; `powi` does the same for floats; `sqrt` is the square root; `abs` removes the sign; `round`, `floor` and `ceil` round to the nearest, down and up; `min` and `max` choose the smaller and the bigger; `clamp(low, high)` forces a value into a range; `abs_diff` gives the distance between two numbers, with no underflow even for unsigned types. You do not need to remember them all — the standard library documentation lists every method of `i32`, `f64` and so on, and RustRover shows them when you type a dot.

Notice the parentheses in `(-3.5_f64).abs()`. They are needed, as section 6 explains.

### The remainder of negative numbers

`%` is the *remainder*, which can be negative. When you need a result that is always between `0` and `n - 1` (for example, to wrap an index around a circle), use `rem_euclid`:

```rust
fn main() {
    println!("-7 % 3             = {}", -7 % 3);
    println!("(-7).rem_euclid(3) = {}", (-7_i32).rem_euclid(3));
    println!("7 % 3              = {}", 7 % 3);
}
```

```text
-7 % 3             = -1
(-7).rem_euclid(3) = 2
7 % 3              = 1
```

## 2. Comparison operators

A comparison compares two values of the **same type** and gives a `bool`.

| Operator | Meaning |
|---|---|
| `==` | equal to |
| `!=` | not equal to |
| `<` | less than |
| `>` | greater than |
| `<=` | less than or equal to |
| `>=` | greater than or equal to |

```rust
fn main() {
    let a = 5;
    let b = 8;
    println!("{a} == {b}: {}", a == b);
    println!("{a} != {b}: {}", a != b);
    println!("{a} <  {b}: {}", a < b);
    println!("{a} >  {b}: {}", a > b);
    println!("{a} <= {b}: {}", a <= b);
    println!("{a} >= {b}: {}", a >= b);
    println!("\"cat\" < \"dog\": {}", "cat" < "dog");
    println!("'a' < 'b': {}", 'a' < 'b');
}
```

```text
5 == 8: false
5 != 8: true
5 <  8: true
5 >  8: false
5 <= 8: true
5 >= 8: false
"cat" < "dog": true
'a' < 'b': true
```

Text and characters compare in dictionary order (more exactly, by their Unicode codes). Be careful: **one `=` assigns, two `==` compare**. Writing `a = b` where you wanted `a == b` is a famous bug in other languages; in Rust it is usually a type error, because an assignment has the value `()`.

## 3. Logical operators

Logical operators combine `bool` values.

| Operator | Name | Result |
|---|---|---|
| `a && b` | and | `true` only if both are `true` |
| `a \|\| b` | or | `true` if at least one is `true` |
| `!a` | not | the opposite of `a` |

```rust
fn main() {
    let t = true;
    let f = false;
    println!("t && t = {}", t && t);
    println!("t && f = {}", t && f);
    println!("t || f = {}", t || f);
    println!("f || f = {}", f || f);
    println!("!t      = {}", !t);

    let a = 10;
    let b = 0;
    let safe = b != 0 && a / b > 2;
    println!("safe = {safe}");
}
```

```text
t && t = true
t && f = false
t || f = true
f || f = false
!t      = false
safe = false
```

The last two lines show **short-circuit evaluation**. The expression `b != 0 && a / b > 2` reads "b is not zero **and** a divided by b is bigger than 2". Because `b != 0` is `false`, the answer is already known and Rust **does not even evaluate** the right side. So `a / b` (a division by zero) never runs. This is a standard trick for writing safe checks: put the check first, the risky part second. `||` works the same way: if the left side is `true`, the right side is skipped.

## 4. Assignment and compound assignment

You already know `=`: it puts a value into a variable. For the common pattern "change a variable using its own value", Rust has **compound assignment** operators: `+=`, `-=`, `*=`, `/=`, `%=`.

```rust
fn main() {
    let mut n = 10;
    n += 5;
    println!("after += 5:  {n}");
    n -= 3;
    println!("after -= 3:  {n}");
    n *= 2;
    println!("after *= 2:  {n}");
    n /= 5;
    println!("after /= 5:  {n}");
    n %= 3;
    println!("after %= 3:  {n}");
}
```

```text
after += 5:  15
after -= 3:  12
after *= 2:  24
after /= 5:  4
after %= 3:  1
```

`n += 5` means exactly `n = n + 5`. The variable must be `mut`. Rust has **no `++` and `--`**: write `n += 1`.

## 5. Bitwise operators

Integers are stored as bits. **Bitwise operators** work on each bit separately. Systems engineers use them every day: flags, permissions, hardware registers, network headers. Here is the summary; Phase 12 goes deeper.

| Operator | Name | Rule for each bit |
|---|---|---|
| `a & b` | AND | `1` only if both bits are `1` |
| `a \| b` | OR | `1` if at least one bit is `1` |
| `a ^ b` | XOR (exclusive OR) | `1` if the bits are different |
| `!a` | NOT | flips every bit |
| `a << n` | shift left | moves bits left by `n` places (fills with zeros) |
| `a >> n` | shift right | moves bits right by `n` places |

```rust
fn main() {
    let a: u8 = 0b1100_1010;
    let b: u8 = 0b1010_0110;
    println!("a       = {:08b}", a);
    println!("b       = {:08b}", b);
    println!("a & b   = {:08b}", a & b);
    println!("a | b   = {:08b}", a | b);
    println!("a ^ b   = {:08b}", a ^ b);
    println!("!a      = {:08b}", !a);
    println!("a << 1  = {:08b}", a << 1);
    println!("a >> 2  = {:08b}", a >> 2);
    let flags: u8 = 0b0000_0100;
    println!("bit 2 set? {}", flags & (1 << 2) != 0);
    println!("bit 3 set? {}", flags & (1 << 3) != 0);
}
```

What happens:

1. `{:08b}` is a printing format: `b` means "in binary", `8` is the width, `0` means "fill with zeros". Lesson 02.6 teaches formats.
2. Every operation is shown with its 8 bits, so you can compare them column by column. For example, in `a & b` a bit is `1` only where both `a` and `b` have `1`.
3. `a << 1` moves all bits one place to the left; the highest bit is lost, and a `0` appears on the right. Shifting left by 1 multiplies by 2 (when nothing is lost); shifting right by 1 divides by 2.
4. The last two lines test a single bit: `1 << 2` is `0b100`, a number with only bit 2 set (counting from 0 on the right). `flags & (1 << 2)` keeps only that bit. If the result is not zero, the bit was set.

```text
a       = 11001010
b       = 10100110
a & b   = 10000010
a | b   = 11101110
a ^ b   = 01101100
!a      = 00110101
a << 1  = 10010100
a >> 2  = 00110010
bit 2 set? true
bit 3 set? false
```

Note that `!` means "not" for `bool` and "flip all bits" for integers.

## 6. Precedence: what is evaluated first

When an expression has several operators, **precedence** decides the order, as in school maths: `*` and `/` before `+` and `-`. When operators have the same precedence, **associativity** decides: arithmetic goes from left to right.

From the strongest to the weakest (the most useful part of the full table):

| Level | Operators |
|---|---|
| 1 (strongest) | method calls and field access: `x.pow(2)`, `t.0` |
| 2 | unary `-` and `!` |
| 3 | `as` |
| 4 | `*` `/` `%` |
| 5 | `+` `-` |
| 6 | `<<` `>>` |
| 7 | `&` |
| 8 | `^` |
| 9 | `\|` |
| 10 | `==` `!=` `<` `>` `<=` `>=` |
| 11 | `&&` |
| 12 | `\|\|` |
| 13 (weakest) | `=` `+=` `-=` … |

```rust
fn main() {
    println!("2 + 3 * 4   = {}", 2 + 3 * 4);
    println!("(2 + 3) * 4 = {}", (2 + 3) * 4);
    println!("10 - 4 - 3  = {}", 10 - 4 - 3);
    println!("100 / 10 / 2 = {}", 100 / 10 / 2);
    println!("-2.pow(2)   = {}", -2_i32.pow(2));
    println!("(-2).pow(2) = {}", (-2_i32).pow(2));
    println!("1 + 2 == 3 && 2 * 2 == 4: {}", 1 + 2 == 3 && 2 * 2 == 4);
}
```

```text
2 + 3 * 4   = 14
(2 + 3) * 4 = 20
10 - 4 - 3  = 3
100 / 10 / 2 = 5
-2.pow(2)   = -4
(-2).pow(2) = 4
1 + 2 == 3 && 2 * 2 == 4: true
```

The surprise is `-2_i32.pow(2)`: the method call (level 1) is stronger than the unary minus (level 2). So Rust computes `2.pow(2) = 4` first and then negates it: `-4`. If you want `(-2)²`, write the parentheses.

> 📏 **Convention:** When in doubt, add parentheses. A reader should never need the precedence table to understand your code. `(a + b) * c` is clearer than relying on rules, and `(x % 4 == 0) && (y > 0)` is friendlier than the bare version.

## 7. Expressions and statements

This is one of the most important ideas in Rust.

- An **expression** produces a value. `5`, `x + 1`, `a > b`, and `foo(3)` (a function call) are all expressions.
- A **statement** does something but produces no value. `let x = 5;` is a statement. A statement usually ends with `;`.

A semicolon turns an expression into a statement: it throws the value away.

In Rust, a **block** `{ ... }` is an expression too. The value of the block is its **last expression**, if that line has **no semicolon**:

```rust
fn main() {
    let y = {
        let x = 3;
        x + 1
    };
    println!("y = {y}");

    let z = {
        let x = 3;
        x + 1;
    };
    println!("z = {:?}", z);
}
```

What happens:

1. The first block creates `x = 3`, then the last line `x + 1` has **no semicolon**, so it is the value of the block: `4`. That value goes into `y`.
2. In the second block the last line is `x + 1;` **with** a semicolon. The value `4` is thrown away, and the block has no value. A block without a value has the value `()` (unit).
3. So `z` is `()`. `{:?}` prints it as `()`.
4. The compiler also warns that `x + 1;` is an "unused arithmetic operation", because it computes a value and drops it.

```text
y = 4
z = ()
```

(The warning appears before this output: `warning: unused arithmetic operation that must be used`.)

If you *expect* a number but the semicolon turns it into `()`, you get a type error with a very friendly hint:

```rust
fn main() {
    let y: i32 = {
        let x = 3;
        x + 1;
    };
    println!("y = {y}");
}
```

```text
error[E0308]: mismatched types
 --> src/main.rs:2:18
  |
2 |       let y: i32 = {
  |  __________________^
3 | |         let x = 3;
4 | |         x + 1;
  | |              - help: remove this semicolon to return this value
5 | |     };
  | |_____^ expected `i32`, found `()`
```

The message says what is wrong (`expected i32, found ()`) and the fix (`remove this semicolon`). This exact mistake is the most common one with functions, which you meet in the next lesson. Another consequence: even an assignment is an expression, and its value is `()`:

```rust
fn main() {
    let mut a = 1;
    let b = (a = 5);
    println!("a = {a}, b = {:?}", b);
}
```

```text
a = 5, b = ()
```

(The compiler also gives two warnings here, because this code is odd. You will almost never write it.) The important message: **in Rust almost everything is an expression**. In Lesson 02.5 you will see that `if` and `match` are expressions too, so you can write `let x = if … { … } else { … };`.

## Common mistakes

**1. Expecting decimals from integer division.** `7 / 2` is `3`. If you want `3.5`, use floats: `7.0 / 2.0`, or convert with `as f64`. A common bug is `let average = sum / count;` with integers.

**2. Mixing integers and floats.** `5 == 5.0` does not compile:

```text
error[E0277]: can't compare `{integer}` with `{float}`
 --> src/main.rs:4:22
  |
4 |     println!("{}", a == b);
  |                      ^^ no implementation for `{integer} == {float}`
```

Write `5.0 == 5.0` or `a as f64 == b`.

**3. Calling a method on a number whose type is not known yet.**

```rust
fn main() {
    let x = 2.0;
    println!("{}", x.sqrt());
}
```

```text
error[E0689]: can't call method `sqrt` on ambiguous numeric type `{float}`
 --> src/main.rs:3:22
  |
3 |     println!("{}", x.sqrt());
  |                      ^^^^
  |
help: you must specify a type for this binding, like `f32`
  |
2 |     let x: f32 = 2.0;
  |          +++
```

The compiler does not know whether `2.0` is an `f32` or an `f64`, so it cannot know which `sqrt`. Write `let x: f64 = 2.0;` or `2.0_f64`.

**4. A forgotten or an extra semicolon at the end of a block.** See section 7.

**5. `-2.pow(2)` is not `4`.** Add parentheses: `(-2_i32).pow(2)`.

**6. Using `=` instead of `==`.** Compare with `==`.

**7. Subtracting from an unsigned number.** `5_u32 - 6` panics (or wraps in release). Check first, or use `checked_sub`.

> 🧪 **Testing:** Arithmetic is the perfect thing for assertions. State the rule, and check it for several values:
>
> ```rust
> fn main() {
>     let a = 7;
>     let b = 3;
>     assert_eq!(a / b, 2);
>     assert_eq!(a % b, 1);
>     assert_eq!(a / b * b + a % b, a);
>     let x = 7.0 / 2.0;
>     assert_eq!(x, 3.5);
>     assert!(a > b && b > 0);
>     println!("arithmetic checks passed");
> }
> ```
>
> ```text
> arithmetic checks passed
> ```
>
> `assert!(condition)` panics if the condition is `false`. (Comparing floats with `assert_eq!` works here because `7.0 / 2.0` is exactly `3.5`. In general, compare floats with a tolerance, as in Lesson 02.2.)

> 📏 **Convention:** Spaces around binary operators: `a + b`, not `a+b`. `cargo fmt` does this for you.

## Try it yourself

1. Create `src/bin/03_operators.rs` with the program from section 1 (arithmetic). Run `cargo run --bin 03_operators`.
2. Add lines that print `17 / 5`, `17 % 5`, `-17 / 5`, `-17 % 5` and `(-17_i32).rem_euclid(5)`. Predict each answer before you run.
3. Add a check of whether a number is even: `let n = 42; println!("{}", n % 2 == 0);`. Try 7.
4. Print the binary form of `0b0101` shifted left by 1, 2 and 3 (use `{:08b}`). What arithmetic operation is a left shift by 1?
5. Write `let r = { let a = 6; a * 7 };` and print `r`. Then add a semicolon after `a * 7` and read the compiler's complaint.
6. Run `cargo fmt` and `cargo clippy`, then commit:

```bash
cd ~/projects/rust
git add code/p02-basics
git commit -m "feat(p02): add operators and expressions examples"
```

## Exercises

### Exercise 1 — Predict the output (easy)

What does this program print? Write down your answers before you run it.

```rust
fn main() {
    println!("{}", 17 / 5);
    println!("{}", 17 % 5);
    println!("{}", 2 + 3 * 2 - 1);
    println!("{}", (2 + 3) * (2 - 1));
    println!("{}", 10 > 3 && 4 < 2);
    println!("{}", 10 > 3 || 4 < 2);
    println!("{}", !(5 == 5));
    println!("{}", 1.0 / 4.0);
}
```

<details>
<summary>Hint</summary>

Integer division cuts off the fraction. `&&` needs both sides true; `||` needs one.

</details>

<details>
<summary>Answer</summary>

```text
3
2
7
5
false
true
false
0.25
```

Why it works: `17 = 3 * 5 + 2`; `2 + 6 - 1 = 7`; `5 * 1 = 5`; `4 < 2` is false, so `&&` gives false and `||` gives true; `!(true)` is false; and `1.0 / 4.0` is a float division.

</details>

### Exercise 2 — Temperature converter with a check (easy)

Convert `36.6` degrees Celsius to Fahrenheit with the formula `F = C * 9 / 5 + 32`. Print the result. Then add `assert!` checks: the result is within `1e-9` of `97.88`, and boiling water (`100` °C) is exactly `212` °F.

<details>
<summary>Hint</summary>

Give the variable a type: `let celsius: f64 = 36.6;`, otherwise `.abs()` fails with `E0689`. Use `(fahrenheit - 97.88).abs() < 1e-9`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let celsius: f64 = 36.6;
    let fahrenheit = celsius * 9.0 / 5.0 + 32.0;
    println!("{celsius} C = {fahrenheit} F");
    assert!((fahrenheit - 97.88).abs() < 1e-9);

    let boiling = 100.0;
    assert_eq!(boiling * 9.0 / 5.0 + 32.0, 212.0);
    println!("all checks passed");
}
```

```text
36.6 C = 97.88000000000001 F
all checks passed
```

Why it works: the exact value has a tiny floating-point error (`97.88000000000001`), which is why we compare with a tolerance. For 100 °C all numbers are exact, so `assert_eq!` works.

</details>

### Exercise 3 — Seconds to hours, minutes, seconds (medium)

Given `total_seconds = 3725`, compute `hours`, `minutes` and `seconds` with `/` and `%` only. Print `1h 2m 5s`. Add three `assert_eq!` checks.

<details>
<summary>Hint</summary>

`hours = total / 3600`. What remains after the hours is `total % 3600`; divide it by 60 for the minutes. The seconds are `total % 60`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let total_seconds = 3725;
    let hours = total_seconds / 3600;
    let minutes = total_seconds % 3600 / 60;
    let seconds = total_seconds % 60;
    println!("{hours}h {minutes}m {seconds}s");
    assert_eq!(hours, 1);
    assert_eq!(minutes, 2);
    assert_eq!(seconds, 5);
}
```

Why it works: `3725 / 3600 = 1`; `3725 % 3600 = 125`; `125 / 60 = 2`; `3725 % 60 = 5`. `%` and `/` have the same precedence and go left to right, so `total_seconds % 3600 / 60` means `(total_seconds % 3600) / 60`. Output: `1h 2m 5s`.

</details>

### Exercise 4 — Leap year (medium)

A year is a leap year if it is divisible by 4, except years divisible by 100 — unless they are also divisible by 400. Write one boolean expression for it and test it with four years: 2024 (leap), 2023 (not), 1900 (not), 2000 (leap). The function line is given; you write the expression. (A function is a named piece of code that takes inputs and returns a result; Lesson 02.4 explains them. For now, only fill in the line between the braces.)

```rust
fn is_leap_year(year: u32) -> bool {
    // write the expression here
}
```

<details>
<summary>Hint</summary>

Use `%`, `==`, `!=`, `&&`, `||`. The rule is "(divisible by 4 and not by 100) or divisible by 400". Do not add a `;` at the end — the last expression is the result.

</details>

<details>
<summary>Answer</summary>

```rust
fn is_leap_year(year: u32) -> bool {
    (year % 4 == 0 && year % 100 != 0) || year % 400 == 0
}

fn main() {
    assert_eq!(is_leap_year(2024), true);
    assert_eq!(is_leap_year(2023), false);
    assert_eq!(is_leap_year(1900), false);
    assert_eq!(is_leap_year(2000), true);
    println!("all leap-year checks passed");
}
```

Why it works: 1900 is divisible by 4 and by 100, but not by 400, so the left part is false and the right part is false. 2000 is divisible by 400, so the right part is true. The output is `all leap-year checks passed`.

</details>

### Exercise 5 — Digits and bits (hard)

(a) Without a loop, compute the sum of the digits of `n = 4721` using `/` and `%`; it must be `14`. (b) A number `n > 0` is a power of two exactly when `n & (n - 1) == 0`. Print whether `64` and `100` are powers of two, and add assertions.

<details>
<summary>Hint</summary>

For (a): the thousands digit is `n / 1000`; the hundreds digit is `n / 100 % 10`; the tens digit is `n / 10 % 10`; the last digit is `n % 10`. For (b), use `u32` numbers; `64` is `0b1000000`, and `63` is `0b0111111`, so `64 & 63` is `0`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let n = 4721;
    let digit_sum = n / 1000 + n / 100 % 10 + n / 10 % 10 + n % 10;
    println!("digit sum of {n} = {digit_sum}");
    assert_eq!(digit_sum, 14);

    let power = 64_u32;
    let not_power = 100_u32;
    println!("64 is a power of two? {}", power & (power - 1) == 0);
    println!("100 is a power of two? {}", not_power & (not_power - 1) == 0);
    assert!(power & (power - 1) == 0);
    assert!(not_power & (not_power - 1) != 0);
}
```

```text
digit sum of 4721 = 14
64 is a power of two? true
100 is a power of two? false
```

Why it works: (a) `4 + 7 + 2 + 1 = 14`. (b) A power of two has exactly one `1` bit. Subtracting 1 turns that bit into `0` and all lower bits into `1`, so the two numbers share no bits and `&` gives `0`. For `100` (`0b1100100`) and `99` (`0b1100011`) the highest bits are shared, so the result is not zero. The operator `&` binds tighter than `==`, so `n & (n - 1) == 0` means `(n & (n - 1)) == 0`. The parentheses around `n - 1` are needed: `-` is stronger than `&`, but they make the intention obvious.

</details>

## Quiz

1. What are `9 / 2` and `9 % 2`?

<details>
<summary>Answer</summary>

`9 / 2 = 4` (integer division cuts off the fraction) and `9 % 2 = 1` (the remainder).

</details>

2. What does `&&` do when its left side is `false`?

<details>
<summary>Answer</summary>

It does not evaluate the right side (short-circuit); the result is `false`.

</details>

3. Is there `x++` in Rust? What do you write instead?

<details>
<summary>Answer</summary>

No. Write `x += 1;`.

</details>

4. Why is `-2_i32.pow(2)` equal to `-4`?

<details>
<summary>Answer</summary>

A method call binds tighter than the unary minus, so `2.pow(2) = 4` is computed first and then negated. Write `(-2_i32).pow(2)` to get `4`.

</details>

5. What is the value of the block `{ let a = 2; a * 3 }`? And of `{ let a = 2; a * 3; }`?

<details>
<summary>Answer</summary>

The first is `6`. The second is `()`, because the semicolon discards the value.

</details>

6. Which operator tests whether a number is even?

<details>
<summary>Answer</summary>

`n % 2 == 0`.

</details>

7. What does `0b0011 << 2` give?

<details>
<summary>Answer</summary>

`0b1100` (that is `12`). Shifting left by 2 multiplies by 4.

</details>

8. Why does `5 == 5.0` not compile?

<details>
<summary>Answer</summary>

Comparisons need two values of the same type. `5` is an integer and `5.0` is a float; Rust never converts between them silently.

</details>

## Summary

- Arithmetic: `+ - * / %`. Integer division truncates; `%` gives the remainder.
- Both operands must have the same type. Convert with `as`.
- Comparisons (`== != < > <= >=`) give `bool`; `&&`, `||` and `!` combine booleans and short-circuit.
- `+=`, `-=`, `*=` and the others change a `mut` variable. There is no `++`.
- Bitwise operators (`& | ^ ! << >>`) work on the bits of integers.
- Precedence: methods, unary, `as`, `* / %`, `+ -`, shifts, `&`, `^`, `|`, comparisons, `&&`, `||`, assignment. Use parentheses for clarity.
- An expression produces a value; a statement does not. A block's value is its last expression without `;`.
- Check your arithmetic with `assert_eq!` and `assert!`.

## Sources and further reading

- [The Rust Programming Language — 3.3 Functions](https://doc.rust-lang.org/book/ch03-03-how-functions-work.html) — the section "Statements and Expressions" explains the block rule from section 7.
- [The Rust Programming Language — Appendix B: Operators and Symbols](https://doc.rust-lang.org/book/appendix-02-operators.html) — a list of all operators.
- [The Rust Reference — Operator expressions](https://doc.rust-lang.org/reference/expressions/operator-expr.html) — the exact rules and the full precedence table.
- [Rust by Example — Expressions](https://doc.rust-lang.org/rust-by-example/expression.html) — short runnable examples.
- [Standard library — Primitive Type f64](https://doc.rust-lang.org/std/primitive.f64.html) — all float methods (`sqrt`, `round`, …).

---

[🏠 Start](../START-HERE.md) · [← 02.2 Data types](02-data-types.md) · [📋 Phase 02](README.md) · [02.4 Functions →](04-functions.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
