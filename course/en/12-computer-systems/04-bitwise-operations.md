<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 12.3 Floating-point numbers](03-floating-point-numbers.md) · [📋 Phase 12](README.md) · [12.5 Text encoding: ASCII, Unicode and UTF-8 →](05-text-encoding-ascii-and-utf8.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/04-bitwise-operations.md) <!-- nav -->

# Lesson 12.4 — Bitwise operations and bit tricks

> **Phase 12 — Computer systems** · Lesson 4 of 15 · ⏱ about 135 minutes

## What you will learn

- How to set, clear, toggle and test single bits with masks.
- How to store several flags or small values in one integer, and get them out again (bit fields).
- The difference between logical and arithmetic shifts, and what happens when you shift too far.
- The built-in bit methods (`count_ones`, `trailing_zeros`, `is_power_of_two`, …) and classic tricks such as `x & (x - 1)` and rounding up to a multiple of a power of two.

## Before you start

- You finished [Lesson 12.3 — Floating-point numbers](03-floating-point-numbers.md).
- [Lesson 02.3 — Operators and expressions](../02-programming-basics/03-operators-and-expressions.md) introduced the operators `&`, `|`, `^`, `!`, `<<` and `>>`, the `{:08b}` format and how to test one bit. This lesson turns them into tools.
- Work in `code/p12-bits`.

## New words

| Word | Meaning |
|---|---|
| **mask** | An integer whose `1` bits mark the bits you are interested in. `x & mask` keeps only those bits. |
| **set / clear / toggle a bit** | Make a bit `1` / make it `0` / flip it. |
| **flag** | A bit that means "yes" or "no" for one option, for example "the owner may write". Several flags fit into one integer. |
| **bit field** | A group of neighbouring bits inside an integer that stores one small value, for example the red part of a colour. |
| **logical shift** | A right shift that fills the new high bits with zeros. Rust uses it for unsigned types. |
| **arithmetic shift** | A right shift that fills the new high bits with copies of the sign bit. Rust uses it for signed types. |
| **rotation** | A shift where the bits that fall out on one side come back in on the other side. |
| **power of two** | A number with exactly one `1` bit: 1, 2, 4, 8, 16 … |
| **alignment** | The rule that an address must be a multiple of some power of two, for example 8. Lesson 12.11 explains why types need it. |
| **bit set** | A set of small numbers stored as the bits of an integer: bit *n* is `1` if *n* is in the set. |

## 1. Masks: working with single bits

A **mask** is a number that has `1` bits exactly where you want to act. The bit with number *n* (counting from 0 on the right) is the mask `1 << n`. Remember the bitwise operators from Lesson 02.3: `&` (AND) gives a `1` bit where both bits are 1, `|` (OR) where at least one is 1, `^` (XOR, *exclusive OR*) where exactly one is 1, and `!` (NOT) flips every bit. With a mask, four operations cover almost everything:

| Goal | Code | Why it works |
|---|---|---|
| set bit *n* | `x \| (1 << n)` | OR with 1 makes the bit 1; OR with 0 changes nothing |
| clear bit *n* | `x & !(1 << n)` | `!` makes a mask with a 0 only at *n*; AND with 0 makes it 0 |
| toggle bit *n* | `x ^ (1 << n)` | XOR with 1 flips the bit; XOR with 0 changes nothing |
| test bit *n* | `x & (1 << n) != 0` | AND keeps only that bit; the result is 0 or not |

```rust
fn main() {
    let mut x: u8 = 0b0000_0101;
    println!("start      {x:08b}");
    x |= 1 << 3;
    println!("set bit 3  {x:08b}");
    x &= !(1 << 0);
    println!("clear 0    {x:08b}");
    x ^= 1 << 7;
    println!("toggle 7   {x:08b}");
    x ^= 1 << 7;
    println!("toggle 7   {x:08b}");
    println!("bit 2 set? {}", x & (1 << 2) != 0);
    println!("bit 1 set? {}", x & (1 << 1) != 0);
    let low_nibble = x & 0x0f;
    let high_nibble = x >> 4;
    println!("nibbles    {high_nibble:04b} {low_nibble:04b}");
}
```

**What happens:**

1. `x |= 1 << 3` is the short form of `x = x | (1 << 3)`. The compound operators `|=`, `&=` and `^=` work like `+=`.
2. Toggling bit 7 twice brings it back: XOR is its own undo.
3. `x & 0x0f` is a mask for the four lowest bits (`0x0f` = `0000_1111`); `x >> 4` moves the four high bits down. Together they split a byte into its two nibbles.
4. In Rust, `&` binds **more strongly** than `!=`, so `x & (1 << 2) != 0` means `(x & (1 << 2)) != 0`. Many people still add parentheses to make it obvious.

```text
start      00000101
set bit 3  00001101
clear 0    00001100
toggle 7   10001100
toggle 7   00001100
bit 2 set? true
bit 1 set? false
nibbles    0000 1100
```

### Flags: many yes/no answers in one number

You know Unix permissions from Phase 11: `rwxr-xr-x` is the octal number `755`. Each octal digit is three **flags**: read (4 = `100`), write (2 = `010`) and execute (1 = `001`). The three groups (owner, group, others) sit next to each other, 3 bits each:

```text
mode 0o755:    owner  group  others
bits:           111    101    101
meaning:        rwx    r-x    r-x
bit numbers:   8 7 6  5 4 3  2 1 0
```

```rust
const READ: u32 = 0b100;
const WRITE: u32 = 0b010;
const EXECUTE: u32 = 0b001;

fn describe(mode: u32) {
    let owner = (mode >> 6) & 0b111;
    let group = (mode >> 3) & 0b111;
    let others = mode & 0b111;
    println!(
        "{mode:o}: owner {owner:03b}, group {group:03b}, others {others:03b}; owner can write: {}",
        owner & WRITE != 0
    );
}

fn main() {
    describe(0o755);
    describe(0o640);
    let mode = (READ | WRITE) << 6 | READ << 3;
    println!("built mode = {mode:o}");
    let without_group_read = mode & !(READ << 3);
    println!("without group read = {without_group_read:o}");
    println!("everyone may execute 0o755? {}", 0o755 & (EXECUTE << 6 | EXECUTE << 3 | EXECUTE) == 0o111);
}
```

**What happens:**

1. To read one group, shift it down to the lowest 3 bits and mask with `0b111`. This "shift, then mask" pattern is how you read any bit field.
2. To build a mode, combine flags with `|` and shift each group into its place. `<<` binds more strongly than `|`, so the expression means `((READ | WRITE) << 6) | (READ << 3)` = `rw-r-----` = `640`.
3. To remove one permission, AND with the inverted mask of that flag.
4. To check that **all** of several flags are set, mask with all of them and compare with the full mask. (`!= 0` would only say that **at least one** is set.)

```text
755: owner 111, group 101, others 101; owner can write: true
640: owner 110, group 100, others 000; owner can write: true
built mode = 640
without group read = 600
everyone may execute 0o755? true
```

> 📏 **Convention:** Give every flag a named constant (`const WRITE: u32 = 0b010;`) instead of writing magic numbers like `2` in the code. For many flags, the popular crate `bitflags` generates a type-safe flags type for you.

## 2. Bit fields: packing small values

The same "shift and mask" works for values that are wider than one bit. A colour on a web page, like `#ff8800`, is three bytes red, green and blue packed into one 24-bit number:

```rust
fn pack(r: u8, g: u8, b: u8) -> u32 {
    (r as u32) << 16 | (g as u32) << 8 | b as u32
}

fn unpack(color: u32) -> (u8, u8, u8) {
    let r = (color >> 16) & 0xff;
    let g = (color >> 8) & 0xff;
    let b = color & 0xff;
    (r as u8, g as u8, b as u8)
}

fn main() {
    let orange = pack(255, 136, 0);
    println!("packed:   {orange:#08x}");
    println!("unpacked: {:?}", unpack(orange));
    println!("unpacked: {:?}", unpack(0x1e90ff));
}
```

**What happens:**

1. `pack` first widens each byte to `u32` (a `u8` shifted by 16 would overflow), then moves it to its place: red to bits 16–23, green to bits 8–15, blue stays at bits 0–7. The fields do not overlap, so `|` just places them side by side.
2. `unpack` shifts each field down and masks with `0xff` to cut away everything above it.

```text
packed:   0xff8800
unpacked: (255, 136, 0)
unpacked: (30, 144, 255)
```

Bit fields are everywhere in systems work: network packet headers (Phase 15), the instruction encodings of a CPU (Lesson 12.7), and the parts of a float that you cut out in Lesson 12.3.

## 3. Shifts in depth

### Logical and arithmetic right shift

Shifting left always fills with zeros. Shifting **right** has two versions, and Rust picks one by the type:

```rust
fn main() {
    let u: u8 = 0b1001_0110;
    let i: i8 = u as i8;
    println!("u8 {u:08b} >> 2 = {:08b}", u >> 2);
    println!("i8 {:08b} >> 2 = {:08b} ({} >> 2 = {})", i, i >> 2, i, i >> 2);
    println!("-7 >> 1 = {}, -7 / 2 = {}", -7i32 >> 1, -7i32 / 2);
    println!("13 << 3 = {} (13 * 8)", 13u32 << 3);
    println!("rotate_left(3):  {:08b}", u.rotate_left(3));
    println!("checked_shl(8):  {:?}", 1u8.checked_shl(8));
    println!("wrapping_shl(9): {}", 1u8.wrapping_shl(9));
}
```

**What happens:**

1. For the unsigned `u8`, `>>` is a **logical shift**: zeros come in on the left.
2. For the signed `i8` with the same bits, `>>` is an **arithmetic shift**: copies of the sign bit come in, so a negative number stays negative. −106 >> 2 is −27.
3. Shifting right by *n* divides by 2ⁿ, but for negative numbers it rounds **down** (towards minus infinity): −7 >> 1 is −4, while the division −7 / 2 rounds towards zero and gives −3.
4. Shifting left by *n* multiplies by 2ⁿ (as long as no bits fall out).
5. `rotate_left(3)` moves the three top bits `100` around to the right end instead of losing them.
6. Shifting a `u8` by 8 or more is an overflow. `checked_shl` returns `None`; `wrapping_shl(9)` takes the amount modulo 8, so it shifts by 1.

```text
u8 10010110 >> 2 = 00100101
i8 10010110 >> 2 = 11100101 (-106 >> 2 = -27)
-7 >> 1 = -4, -7 / 2 = -3
13 << 3 = 104 (13 * 8)
rotate_left(3):  10110100
checked_shl(8):  None
wrapping_shl(9): 2
```

> ⚡ **Performance:** You do not need to write `x << 3` instead of `x * 8` to make code fast: the compiler does that change for you (Lesson 12.8 shows it in assembly). Write `* 8` when you mean multiplication and `<< 3` when you mean moving bits.

## 4. Built-in bit methods and classic tricks

Every integer type has methods that count and find bits. Most of them compile to a single CPU instruction.

```rust
fn main() {
    let x: u32 = 0b0010_1100;
    println!("x                   = {x:032b}");
    println!("count_ones          = {}", x.count_ones());
    println!("leading_zeros       = {}", x.leading_zeros());
    println!("trailing_zeros      = {}", x.trailing_zeros());
    println!("x & (x - 1)         = {:08b}  (lowest 1 cleared)", x & (x - 1));
    println!("x & x.wrapping_neg()= {:08b}  (lowest 1 kept)", x & x.wrapping_neg());
    for n in [1u32, 6, 64, 100] {
        println!("{n:>3}: power of two? {:<5} next power of two: {}", n.is_power_of_two(), n.next_power_of_two());
    }
    println!("swap_bytes(0x12345678) = {:#x}", 0x1234_5678u32.swap_bytes());
}
```

**What happens:**

1. `count_ones` counts the `1` bits (also called the *population count*). `leading_zeros` counts zeros from the top, `trailing_zeros` from the bottom. `trailing_zeros` is the position of the lowest `1`.
2. `x & (x - 1)` **clears the lowest `1` bit**. Subtracting 1 turns the lowest `1` into `0` and all zeros below it into ones; the AND then removes all of them. A power of two has only one `1`, so for it the result is 0: that is how `is_power_of_two` can work.
3. `x & x.wrapping_neg()` keeps **only** the lowest `1` bit (because −x = !x + 1, Lesson 12.2).
4. `next_power_of_two` rounds up to a power of two. `Vec` and hash tables grow their memory in powers of two.
5. `swap_bytes` reverses the byte order: this is how a number is converted between little-endian and big-endian (Lesson 12.1).

```text
x                   = 00000000000000000000000000101100
count_ones          = 3
leading_zeros       = 26
trailing_zeros      = 2
x & (x - 1)         = 00101000  (lowest 1 cleared)
x & x.wrapping_neg()= 00000100  (lowest 1 kept)
  1: power of two? true  next power of two: 1
  6: power of two? false next power of two: 8
 64: power of two? true  next power of two: 64
100: power of two? false next power of two: 128
swap_bytes(0x12345678) = 0x78563412
```

### Rounding up to a multiple of a power of two

Memory allocators (Lesson 12.13) must often round an address up to the next multiple of 8, 16 or 4096. When `align` is a power of two, `align - 1` is a mask of all the low bits (for 8: `0b111`). An address is a multiple of `align` exactly when those low bits are 0. To round up:

```text
align_up(addr, align) = (addr + align - 1) & !(align - 1)

example: addr = 13, align = 8
  13 + 7       = 20  = 0b1_0100
  !(8 - 1)     =       ...1_1000   (all bits except the lowest three)
  20 & !7      = 16  = 0b1_0000
```

Adding `align - 1` pushes the address past the next boundary (unless it was already on one), and the mask cuts it back down to the boundary. You will write this function in Exercise 4.

> 🧪 **Testing:** Bit tricks are easy to get almost right. Test them against a slow, obvious version: for `align_up`, check for thousands of inputs that the result is aligned, not smaller than the input, and less than `align` away from it. Testing such *properties* is the idea behind property-based testing (Phase 09).

## Common mistakes

> ⚠️ **Common mistake: shifting by the full width.** A mask with the lowest `width` bits looks easy:

```rust
fn mask_of(width: u32) -> u32 {
    (1 << width) - 1
}

fn main() {
    println!("{:b}", mask_of(4));
    println!("{:b}", mask_of(32));
}
```

```text
1111

thread 'main' (5578658) panicked at src/main.rs:2:5:
attempt to shift left with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

In a debug build, shifting a `u32` by 32 panics. In a release build it prints `1111` and then `0`: the shift amount is taken modulo 32, so `1 << 32` becomes `1 << 0`, and the "all ones" mask silently becomes zero. Fix: handle the full width separately (`if width == 32 { u32::MAX } else { (1 << width) - 1 }`), or use `u32::MAX.checked_shr(32 - width).unwrap_or(0)`. If the shift amount is a constant, the compiler even stops you: `1u32 << 32` gives `error: this arithmetic operation will overflow`.

> ⚠️ **Common mistake: shifting before widening.** `(r << 16) as u32` with `r: u8` shifts the `u8` first, which overflows. Widen first: `(r as u32) << 16`.

> ⚠️ **Common mistake: `!` versus `-`.** `!mask` flips the bits; `-mask` is the negative number. For unsigned types, `-` does not even compile, and for signed types they differ by one (`!x == -x - 1`).

## Try it yourself

Print the positions of all `1` bits of a number with the two tricks from Section 4: `trailing_zeros` finds the lowest `1`, and `x &= x - 1` removes it.

```rust
fn main() {
    let mut x: u64 = 0b1010_0110;
    print!("set bits of {x:#b}:");
    while x != 0 {
        print!(" {}", x.trailing_zeros());
        x &= x - 1;
    }
    println!();
}
```

**What happens:**

1. As long as there is a `1` left, print the position of the lowest one.
2. Remove it with `x &= x - 1`. The loop runs once per `1` bit, not once per bit position, so it is fast for numbers with few ones.

```text
set bits of 0b10100110: 1 2 5 7
```

Then change the number and predict the output before you run it.

## Exercises

### Exercise 1 — Predict the bits (easy)

With `let a: u8 = 0b1100;`, predict `a | 0b0011`, `a & 0b1010`, `a ^ a`, `!0u8` and `0b1u8 << 4 >> 2` as decimal numbers. Then check with one `println!`.

<details>
<summary>Hint</summary>

Write the bits under each other and apply the rule of each operator column by column. Shifts are evaluated from left to right.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    let a: u8 = 0b1100;
    println!("{} {} {} {} {}", a | 0b0011, a & 0b1010, a ^ a, !0u8, 0b1u8 << 4 >> 2);
}
```

```text
15 8 0 255 4
```

Why it works: `1100 | 0011 = 1111` (15); `1100 & 1010 = 1000` (8); any value XOR itself is 0; `!0u8` is eight ones (255); `1 << 4` is 16 and `>> 2` gives 4.

</details>

### Exercise 2 — Permission text (medium)

In `src/lib.rs`, write `pub fn permissions_string(mode: u32) -> String` that turns the lowest 9 bits of a Unix mode into text like `rwxr-xr-x`. Ignore higher bits (a full mode from `stat` can be `0o100644`, where `100` means "regular file"). Write a test with `0o755`, `0o640`, `0o000`, `0o777` and `0o100644`.

<details>
<summary>Hint</summary>

Loop over the shifts `[6, 3, 0]`. For each group, get its 3 bits with "shift, then mask with `0b111`", and push `'r'` or `'-'`, `'w'` or `'-'`, `'x'` or `'-'` depending on the bits `0b100`, `0b010`, `0b001`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Turns the lowest 9 bits of a Unix mode into text like "rwxr-xr-x".
pub fn permissions_string(mode: u32) -> String {
    let mut text = String::new();
    for shift in [6, 3, 0] {
        let part = (mode >> shift) & 0b111;
        text.push(if part & 0b100 != 0 { 'r' } else { '-' });
        text.push(if part & 0b010 != 0 { 'w' } else { '-' });
        text.push(if part & 0b001 != 0 { 'x' } else { '-' });
    }
    text
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn known_modes() {
        assert_eq!(permissions_string(0o755), "rwxr-xr-x");
        assert_eq!(permissions_string(0o640), "rw-r-----");
        assert_eq!(permissions_string(0o000), "---------");
        assert_eq!(permissions_string(0o777), "rwxrwxrwx");
        assert_eq!(permissions_string(0o100644), "rw-r--r--");
    }
}
```

Why it works: each shift brings one group to the lowest 3 bits, and the mask `0b111` cuts off everything above, including the file-type bits of `0o100644`.

</details>

### Exercise 3 — A bit set (medium)

Write a type `pub struct BitSet64(u64)` that stores a set of the numbers 0 to 63. Give it `new`, `insert`, `remove`, `contains`, `len`, `is_empty`, `union` and `intersection`. `insert(64)` must panic with a clear message. Write tests for insert/contains/remove (including inserting the same number twice), for union and intersection, and a `#[should_panic]` test.

<details>
<summary>Hint</summary>

`insert` sets bit *n*, `remove` clears it, `contains` tests it, `len` is `count_ones`. Union is `|` of the two numbers, intersection is `&`. Let `contains` return `false` for `n >= 64` instead of panicking (or shifting too far).

</details>

<details>
<summary>Answer</summary>

```rust
/// A set of the numbers 0 to 63, stored as the bits of one u64.
#[derive(Debug, Default, Clone, Copy, PartialEq, Eq)]
pub struct BitSet64(u64);

impl BitSet64 {
    pub fn new() -> Self {
        BitSet64(0)
    }

    pub fn insert(&mut self, n: u32) {
        assert!(n < 64, "only 0 to 63 fit");
        self.0 |= 1 << n;
    }

    pub fn remove(&mut self, n: u32) {
        assert!(n < 64, "only 0 to 63 fit");
        self.0 &= !(1 << n);
    }

    pub fn contains(&self, n: u32) -> bool {
        n < 64 && self.0 & (1 << n) != 0
    }

    pub fn len(&self) -> u32 {
        self.0.count_ones()
    }

    pub fn is_empty(&self) -> bool {
        self.0 == 0
    }

    pub fn union(self, other: BitSet64) -> BitSet64 {
        BitSet64(self.0 | other.0)
    }

    pub fn intersection(self, other: BitSet64) -> BitSet64 {
        BitSet64(self.0 & other.0)
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn insert_contains_remove() {
        let mut set = BitSet64::new();
        assert!(set.is_empty());
        set.insert(0);
        set.insert(5);
        set.insert(63);
        set.insert(5);
        assert_eq!(set.len(), 3);
        assert!(set.contains(5) && set.contains(63) && !set.contains(4));
        assert!(!set.contains(100));
        set.remove(5);
        assert!(!set.contains(5));
        assert_eq!(set.len(), 2);
    }

    #[test]
    fn union_and_intersection() {
        let mut a = BitSet64::new();
        let mut b = BitSet64::new();
        for n in [1, 2, 3] {
            a.insert(n);
        }
        for n in [3, 4] {
            b.insert(n);
        }
        assert_eq!(a.union(b).len(), 4);
        let both = a.intersection(b);
        assert_eq!(both.len(), 1);
        assert!(both.contains(3));
    }

    #[test]
    #[should_panic(expected = "only 0 to 63 fit")]
    fn rejects_64() {
        BitSet64::new().insert(64);
    }
}
```

Why it works: one `u64` holds 64 yes/no answers, one per number. Set operations become single instructions: `|` for union, `&` for intersection. In `contains`, `&&` stops before the shift when `n >= 64`, so the shift can never overflow. Clippy asks for `is_empty` whenever a type has `len`, which is why it is there.

</details>

### Exercise 4 — Align up (hard)

Write `pub fn align_up(addr: usize, align: usize) -> usize` and `pub fn is_aligned(addr: usize, align: usize) -> bool` for an `align` that is a power of two (panic otherwise). Tests: `align_up(13, 16)` = 16, `align_up(8, 8)` = 8, `align_up(0, 8)` = 0, `align_up(4097, 4096)` = 8192; a property test over many inputs; a `#[should_panic]` test for `align = 12`.

<details>
<summary>Hint</summary>

Use the formula from Section 4. For `is_aligned`, the low bits `addr & (align - 1)` must be zero. For the property test, loop over several aligns and the addresses `0..10_000` and check three things: the result is aligned, it is not smaller than `addr`, and it is less than `align` away.

</details>

<details>
<summary>Answer</summary>

```rust
/// Rounds `addr` up to the next multiple of `align` (a power of two).
pub fn align_up(addr: usize, align: usize) -> usize {
    assert!(align.is_power_of_two(), "align must be a power of two");
    (addr + align - 1) & !(align - 1)
}

/// True if `addr` is a multiple of `align` (a power of two).
pub fn is_aligned(addr: usize, align: usize) -> bool {
    assert!(align.is_power_of_two(), "align must be a power of two");
    addr & (align - 1) == 0
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn align_examples() {
        assert_eq!(align_up(0, 8), 0);
        assert_eq!(align_up(1, 8), 8);
        assert_eq!(align_up(8, 8), 8);
        assert_eq!(align_up(13, 16), 16);
        assert_eq!(align_up(4097, 4096), 8192);
        assert!(is_aligned(4096, 4096));
        assert!(!is_aligned(4100, 8));
    }

    #[test]
    fn align_up_properties() {
        for align in [1usize, 2, 4, 8, 64, 4096] {
            for addr in 0..10_000 {
                let up = align_up(addr, align);
                assert!(up >= addr && up - addr < align && is_aligned(up, align));
            }
        }
    }

    #[test]
    #[should_panic(expected = "power of two")]
    fn align_must_be_power_of_two() {
        align_up(10, 12);
    }
}
```

```text
running 3 tests
test tests::align_examples ... ok
test tests::align_must_be_power_of_two - should panic ... ok
test tests::align_up_properties ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: for a power of two, `align - 1` is a mask of the low bits and `!(align - 1)` clears them, which rounds down to a multiple; adding `align - 1` first turns "round down" into "round up". With `align = 12` the mask trick would be wrong (12 is `1100`, so `11` is not a mask of low bits), which is why the function refuses it. One limit remains: `addr + align - 1` can overflow for addresses near `usize::MAX`; real allocators use `checked_add` there.

</details>

## Quiz

1. How do you clear bit 5 of `x`?

<details>
<summary>Answer</summary>

`x & !(1 << 5)` (or `x &= !(1 << 5)`).

</details>

2. What does `(x >> 8) & 0xff` give for `x = 0x00ab_cdef`?

<details>
<summary>Answer</summary>

`0xcd`: the shift brings the second byte to the bottom, the mask keeps only that byte.

</details>

3. What is the difference between `0b1000_0000u8 >> 1` and `0b1000_0000u8 as i8 >> 1`?

<details>
<summary>Answer</summary>

The `u8` shift is logical and gives `0b0100_0000` (64). The `i8` shift is arithmetic, copies the sign bit, and gives `0b1100_0000` (−64).

</details>

4. Why is `x & (x - 1) == 0` true for powers of two?

<details>
<summary>Answer</summary>

A power of two has exactly one `1` bit, and `x & (x - 1)` clears the lowest `1` bit, so nothing is left. (It is also true for `x = 0`, which is not a power of two; `is_power_of_two` handles that case.)

</details>

5. What happens with `1u32 << n` when `n` is 32 in a debug build, and in a release build?

<details>
<summary>Answer</summary>

Debug: panic "attempt to shift left with overflow". Release: the shift amount is taken modulo 32, so the result is `1 << 0` = 1, silently.

</details>

6. Why does `align_up` only work when `align` is a power of two?

<details>
<summary>Answer</summary>

Only then is `align - 1` a mask of all the low bits (like `0b111` for 8), so that clearing those bits gives a multiple of `align`.

</details>

## Summary

- A mask marks the bits you want: `| mask` sets, `& !mask` clears, `^ mask` toggles, `& mask != 0` tests.
- Many flags or small values fit into one integer. Read a field with "shift, then mask"; write it with "shift into place, then OR".
- `>>` is logical for unsigned types and arithmetic for signed types. Shifting by the full width or more is an overflow: it panics in debug and silently uses the amount modulo the width in release.
- Built-in methods (`count_ones`, `leading_zeros`, `trailing_zeros`, `is_power_of_two`, `next_power_of_two`, `rotate_left`, `swap_bytes`) are fast and clear; use them instead of hand-written loops.
- `x & (x - 1)` clears the lowest `1`; `(addr + align - 1) & !(align - 1)` rounds up to a power-of-two alignment.
- Test bit tricks against obvious properties over many inputs.

## Sources and further reading

- [Rust standard library: `u64`](https://doc.rust-lang.org/std/primitive.u64.html) — `count_ones`, `leading_zeros`, `trailing_zeros`, `rotate_left`, `swap_bytes`, `checked_shl`, `is_power_of_two` and more.
- [The Rust Reference: Arithmetic and logical binary operators](https://doc.rust-lang.org/reference/expressions/operator-expr.html) — the exact rules for `&`, `|`, `^`, `<<`, `>>` and their precedence.
- [Bit Twiddling Hacks (Stanford)](https://graphics.stanford.edu/~seander/bithacks.html) — a famous collection of bit tricks, written for C; most of them work the same in Rust.
- [The `bitflags` crate](https://docs.rs/bitflags) — a type-safe way to define a set of flags.

---

[🏠 Start](../START-HERE.md) · [← 12.3 Floating-point numbers](03-floating-point-numbers.md) · [📋 Phase 12](README.md) · [12.5 Text encoding: ASCII, Unicode and UTF-8 →](05-text-encoding-ascii-and-utf8.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
