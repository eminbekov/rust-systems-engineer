<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 12 — Computer systems](README.md) · [📋 Phase 12](README.md) · [12.2 Integers in memory: two's complement and overflow →](02-integers-in-memory.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/01-bits-bytes-and-number-systems.md) <!-- nav -->

# Lesson 12.1 — Bits, bytes and number systems

> **Phase 12 — Computer systems** · Lesson 1 of 15 · ⏱ about 120 minutes

## What you will learn

- How a positional number system works, and how to convert between decimal, binary, hexadecimal and octal by hand.
- How to print and parse numbers in any of these bases in Rust.
- Powers of two, and why 1 KiB (1024 bytes) is not the same as 1 KB (1000 bytes).
- What byte order (endianness) is, and how to turn a number into bytes and back with `to_le_bytes`, `to_be_bytes` and `from_be_bytes`.

## Before you start

- You finished [Phase 11 — Linux, the shell and containers](../11-linux-shell-and-containers/README.md).
- You remember the first look at bits, bytes and hexadecimal from [Lesson 00.3 — How a computer runs a program](../00-getting-started/03-how-a-computer-runs-a-program.md) and the integer types and number literals (`0xff`, `0b1010`) from [Lesson 02.2 — Data types](../02-programming-basics/02-data-types.md). This lesson goes deeper; it does not repeat them.
- Create the practice package for this phase:

```bash
cd ~/projects/rust
cargo new code/p12-bits
```

**How the code of Phase 12 is organised.** Each topic gets its own package `code/p12-<topic>`: `p12-bits` for Lessons 12.1 to 12.4, `p12-text` for Lesson 12.5, `p12-cpu` for Lesson 12.6, `p12-asm` for Lessons 12.7 and 12.8, and so on. An example goes into `src/main.rs` (replace the old one) and runs with `cargo run`. If you want to keep an example, copy it to `src/bin/<name>.rs` and run it with `cargo run --bin <name>`. (As soon as `src/bin/` has files, `cargo run` alone asks which program you mean; then run `src/main.rs` with `cargo run --bin p12-bits`, the package name.) Exercise answers with tests go into `src/lib.rs` and run with `cargo test`.

## New words

| Word | Meaning |
|---|---|
| **base** (radix) | How many different digits a number system uses. Decimal has base 10, binary base 2, hexadecimal base 16, octal base 8. |
| **positional number system** | A system where the value of a digit depends on its place. In 352, the 3 means 3 hundreds. |
| **hexadecimal** (hex) | Base 16. Digits `0`–`9` and `a`–`f` (a = 10 … f = 15). One hex digit is exactly 4 bits. |
| **octal** | Base 8. Digits `0`–`7`. One octal digit is exactly 3 bits. Used for Unix file permissions. |
| **nibble** | Half a byte: 4 bits, one hex digit. |
| **most / least significant bit** | The bit with the biggest place value (on the left) / the smallest (on the right, place value 1). Bits are counted from 0 on the right. |
| **word** | The natural size of a number for a CPU (Central Processing Unit). On a 64-bit CPU a word is 64 bits; `usize` has this size. |
| **KiB, MiB, GiB** | Binary size units from the IEC (International Electrotechnical Commission): 1 KiB = 1024 bytes, 1 MiB = 1024 KiB, 1 GiB = 1024 MiB. |
| **KB, MB, GB** | Decimal size units from the SI (International System of Units): 1 KB = 1000 bytes, 1 MB = 1000 KB. |
| **byte order** (endianness) | The order in which the bytes of a multi-byte number are stored in memory or in a file. |
| **little-endian** | The least significant byte comes first (at the lowest address). Your Mac and x86-64 computers work like this. |
| **big-endian** | The most significant byte comes first. Network protocols and many file formats use it, so it is also called *network byte order*. |
| **magic number** | A few fixed bytes at the start of a file that say which format it is. |

## 1. One idea, many bases

You already read binary in Lesson 00.3 by adding place values. Here is the general rule behind it.

In a **positional number system** with base *b*, each place is worth *b* times the place to its right. The rightmost place is worth 1. So a number is a sum of "digit × place value":

```text
base 10:   352      = 3×100 + 5×10 + 2×1                    = 352
base 2:    11001010 = 1×128 + 1×64 + 0×32 + 0×16
                    + 1×8   + 0×4  + 1×2  + 0×1             = 202
base 16:   ca       = 12×16 + 10×1                          = 202
base 8:    312      = 3×64  + 1×8  + 2×1                    = 202
```

The same number, 202, has four different spellings. The number does not change; only the way we write it does. The computer always stores bits. "Decimal" or "hex" is only a question of how a person reads or writes them.

### Decimal to any base: divide and keep the remainders

Lesson 00.3 showed the "take the biggest place value that fits" method. A second method works for every base and is easy to turn into code: **divide by the base again and again, and write down the remainders**. The remainders, read from the last to the first, are the digits.

```text
202 in binary:                    202 in hex:
202 / 2 = 101  remainder 0        202 / 16 = 12  remainder 10 → a
101 / 2 =  50  remainder 1         12 / 16 =  0  remainder 12 → c
 50 / 2 =  25  remainder 0        read upwards:  ca
 25 / 2 =  12  remainder 1
 12 / 2 =   6  remainder 0
  6 / 2 =   3  remainder 0
  3 / 2 =   1  remainder 1
  1 / 2 =   0  remainder 1
read upwards: 11001010
```

Why does it work? The remainder of a division by 2 is the value of the last binary digit (is the number odd?). Dividing by 2 throws that digit away and shifts the others one place to the right. Then you repeat.

### Binary to hex and back: groups of four

Because 16 = 2⁴, one hex digit is exactly four bits (a **nibble**). To convert, cut the binary number into groups of four bits from the right, and replace each group:

```text
binary:  1100 1010        hex digit table (memorise the first and last rows):
hex:        c    a        0000=0 0001=1 0010=2 0011=3 0100=4 0101=5 0110=6 0111=7
                          1000=8 1001=9 1010=a 1011=b 1100=c 1101=d 1110=e 1111=f
```

The same trick works for octal with groups of **three** bits, because 8 = 2³: `011 001 010` is `3 1 2`, octal 312.

That is why systems engineers love hex: a byte is always exactly two hex digits (`00` to `ff`), a 32-bit number is eight hex digits, and you can still see every bit if you need it.

Octal is rare today, with one important exception you met in Phase 11: Unix file permissions. In `chmod 755`, each octal digit is three bits `rwx` (read, write, execute): `7` = `111` = `rwx`, `5` = `101` = `r-x`.

## 2. Number systems in Rust

You know how to *write* literals in other bases (`0b…`, `0o…`, `0xff`). Now let us *print* and *parse* them. Put this into `src/main.rs` of `code/p12-bits`:

```rust
fn main() {
    let n: u32 = 202;
    println!("decimal: {n}");
    println!("binary:  {n:b}");
    println!("octal:   {n:o}");
    println!("hex:     {n:x}  {n:X}");
    println!("with prefixes: {n:#b} {n:#o} {n:#x}");
    println!("padded:  {n:#010b} {n:08b} {n:04x}");

    let from_bin = u32::from_str_radix("11001010", 2).unwrap();
    let from_hex = u32::from_str_radix("ca", 16).unwrap();
    let from_oct = u32::from_str_radix("312", 8).unwrap();
    println!("parsed:  {from_bin} {from_hex} {from_oct}");

    println!("digit 'c' in base 16 = {:?}", 'c'.to_digit(16));
    println!("digit 'c' in base 10 = {:?}", 'c'.to_digit(10));
    println!("number 11 as a base-16 digit = {:?}", char::from_digit(11, 16));
}
```

**What happens:**

1. `{n:b}`, `{n:o}`, `{n:x}` and `{n:X}` print the number in binary, octal and hex (small or capital letters). These work for every integer type.
2. `#` adds the prefix `0b`, `0o` or `0x`, the same prefixes that Rust literals use.
3. `{n:#010b}` means: prefix, width 10 **including the prefix**, fill with zeros. `{n:08b}` is 8 digits without a prefix, the usual way to show one byte. `{n:04x}` is 4 hex digits.
4. `u32::from_str_radix(text, base)` parses text written in the given base (2 to 36). It returns a `Result`, because the text may be wrong; here we know it is right, so `unwrap` is fine.
5. `char::to_digit(base)` turns one digit character into its value, or `None` if it is not a digit of that base. `char::from_digit(value, base)` goes the other way.

```text
decimal: 202
binary:  11001010
octal:   312
hex:     ca  CA
with prefixes: 0b11001010 0o312 0xca
padded:  0b11001010 11001010 00ca
parsed:  202 202 202
digit 'c' in base 16 = Some(12)
digit 'c' in base 10 = None
number 11 as a base-16 digit = Some('b')
```

> 📏 **Convention:** Print bytes as two hex digits (`{:02x}`), and a whole slice of bytes with `{:02x?}`, which you will use below. Use `_` in long binary literals to group bits by four: `0b1100_1010`.

## 3. Powers of two and sizes

With *n* bits you can make 2ⁿ different patterns. Some powers of two appear so often that it is worth knowing them by heart:

```rust
fn main() {
    for p in [8, 10, 16, 20, 30, 32, 40] {
        println!("2^{p:<2} = {}", 1u64 << p);
    }
    println!("u32 has {} bits, u64 has {} bits", u32::BITS, u64::BITS);
    println!("usize has {} bits on this computer", usize::BITS);
    for n in [1u32, 255, 256, 1000, 1_000_000] {
        let bits_needed = n.ilog2() + 1;
        println!("{n:>9} needs {bits_needed:>2} bits: {n:b}");
    }
}
```

**What happens:**

1. `1u64 << p` shifts the bit `1` to the left by `p` places. Each place to the left doubles the value, so this is 2ᵖ (Lesson 02.3 introduced shifts; Lesson 12.4 uses them a lot).
2. `u32::BITS` is a constant: the number of bits in the type. `usize::BITS` is 64 on your Mac, because `usize` has the size of a **word** of a 64-bit CPU.
3. `n.ilog2()` is the integer base-2 logarithm: the place of the highest `1` bit, counted from 0. Adding 1 gives the number of bits needed to write `n`. (It panics for 0, because 0 has no `1` bit.)

```text
2^8  = 256
2^10 = 1024
2^16 = 65536
2^20 = 1048576
2^30 = 1073741824
2^32 = 4294967296
2^40 = 1099511627776
u32 has 32 bits, u64 has 64 bits
usize has 64 bits on this computer
        1 needs  1 bits: 1
      255 needs  8 bits: 11111111
      256 needs  9 bits: 100000000
     1000 needs 10 bits: 1111101000
  1000000 needs 20 bits: 11110100001001000000
```

### KiB or KB?

2¹⁰ = 1024 is very close to 1000. For a long time people used "kilobyte" for both, which caused confusion. Today there are two sets of units:

| Binary unit (IEC) | Bytes | Decimal unit (SI) | Bytes |
|---|---|---|---|
| 1 KiB (kibibyte) | 2¹⁰ = 1,024 | 1 KB (kilobyte) | 10³ = 1,000 |
| 1 MiB (mebibyte) | 2²⁰ = 1,048,576 | 1 MB (megabyte) | 10⁶ = 1,000,000 |
| 1 GiB (gibibyte) | 2³⁰ = 1,073,741,824 | 1 GB (gigabyte) | 10⁹ = 1,000,000,000 |
| 1 TiB (tebibyte) | 2⁴⁰ ≈ 1.10 × 10¹² | 1 TB (terabyte) | 10¹² |

The difference grows with size: 1 TiB is about 10% more than 1 TB. Memory sizes are almost always powers of two (a page of memory, a cache), so in this phase we use KiB and MiB. For example, `sysctl hw.l1dcachesize` on an Apple M4 Pro prints `65536`, which is exactly 64 KiB (the level-1 data cache of one efficiency core; `sysctl hw.perflevel0.l1dcachesize` shows the performance cores' `131072`, 128 KiB — Lesson 12.9 explains these caches). Disk makers usually use decimal units, which is why a "1 TB" disk shows up as about 931 GiB.

> 💡 **Tip:** Some tools write "K" or "KB" but mean 1024. Kubernetes is careful: `128Mi` means 128 MiB, `128M` means 128 MB. When a number matters, check which unit a tool uses.

## 4. Byte order: which byte comes first?

Memory is a row of numbered bytes (Lesson 03.1). A `u32` needs 4 bytes, so the question is: **in which order are those 4 bytes put into the 4 boxes?** There are two answers, and both are used.

```text
the number 0x12345678 (four bytes: 12 34 56 78)

address:        100  101  102  103
little-endian:   78   56   34   12     ← least significant byte first (your Mac, x86-64)
big-endian:      12   34   56   78     ← most significant byte first (networks, many file formats)
```

Inside one running program you never notice the byte order: the CPU reads its own numbers correctly. It matters at the **borders**: when you write numbers into a file or a network packet, or read bytes made by another program. Then both sides must agree on the order. Rust makes the order explicit:

```rust
fn main() {
    let n: u32 = 0x1234_5678;
    println!("number:        {n:#010x}");
    println!("little-endian: {:02x?}", n.to_le_bytes());
    println!("big-endian:    {:02x?}", n.to_be_bytes());
    println!("native:        {:02x?}", n.to_ne_bytes());
    println!("this computer is little-endian: {}", cfg!(target_endian = "little"));

    let bytes = [0x00, 0x00, 0x01, 0x2c];
    let as_big = u32::from_be_bytes(bytes);
    let as_little = u32::from_le_bytes(bytes);
    println!("{bytes:02x?} read as big-endian    = {as_big}");
    println!("{bytes:02x?} read as little-endian = {as_little}");
}
```

**What happens:**

1. `to_le_bytes` returns an array `[u8; 4]` in little-endian order, `to_be_bytes` in big-endian order. `to_ne_bytes` uses the **native** order of the computer the program runs on.
2. `cfg!(target_endian = "little")` is `true` when the program is compiled for a little-endian target. It is known at compile time.
3. `from_be_bytes` and `from_le_bytes` build a number from 4 bytes. The **same four bytes** give 300 or 738,263,040, depending on the order you assume. Choosing the wrong order does not crash; it silently gives a wrong number.

```text
number:        0x12345678
little-endian: [78, 56, 34, 12]
big-endian:    [12, 34, 56, 78]
native:        [78, 56, 34, 12]
this computer is little-endian: true
[00, 00, 01, 2c] read as big-endian    = 300
[00, 00, 01, 2c] read as little-endian = 738263040
```

This is the detail that Lesson 00.3 postponed: the instruction `add w0, w1, w0` is the 32-bit number `0x0b000020`, and in the program file it is stored as the bytes `20 00 00 0b`. That is little-endian.

> 🔒 **Security:** Bytes that come from a file or a network are input from outside. Check the length before you read a number from them: `u32::from_be_bytes` needs exactly 4 bytes, and turning a slice into `[u8; 4]` with `try_into()` returns an error if it is shorter. Never assume that a length field in a packet is honest.

> 🧪 **Testing:** For code that reads or writes bytes, write a **round-trip test**: turn a value into bytes and back, and check that you get the same value (`assert_eq!(u32::from_be_bytes(n.to_be_bytes()), n)`). Add one test with fixed bytes written by hand, so that a wrong byte order fails the test.

## Common mistakes

> ⚠️ **Common mistake: a prefix in `from_str_radix`.** The text must contain only digits. A `0x` prefix is not a hex digit:

```rust
fn main() {
    println!("{:?}", u32::from_str_radix("0xff", 16));
    println!("{:?}", u32::from_str_radix("ff", 16));
    println!("{:?}", u8::from_str_radix("100000000", 2));
}
```

```text
Err(ParseIntError { kind: InvalidDigit })
Ok(255)
Err(ParseIntError { kind: PosOverflow })
```

Remove the prefix first, for example with `text.trim_start_matches("0x")`. The third line shows the other common error: nine bits do not fit into a `u8`, so the result is `PosOverflow` (positive overflow).

> ⚠️ **Common mistake: native order in files.** `to_ne_bytes` is fine for data that stays inside one program, but a file written with it on your Mac cannot be read correctly by a big-endian computer. For files and network data, always choose `to_le_bytes` or `to_be_bytes` on purpose, and write the choice in the format description.

> ⚠️ **Common mistake: mixing KB and KiB.** "The cache is 64 KB" may mean 64,000 or 65,536 bytes. In code, write sizes as `64 * 1024` or `1 << 16`, never as `64000`.

## Try it yourself

Build a small hex dump, like the `xxd` tool from Lesson 00.3. Put this into `src/main.rs`:

```rust
fn hexdump(data: &[u8]) {
    for (line, chunk) in data.chunks(8).enumerate() {
        let offset = line * 8;
        print!("{offset:08x}: ");
        for byte in chunk {
            print!("{byte:02x} ");
        }
        for _ in chunk.len()..8 {
            print!("   ");
        }
        let text: String = chunk
            .iter()
            .map(|&b| if b.is_ascii_graphic() || b == b' ' { b as char } else { '.' })
            .collect();
        println!(" {text}");
    }
}

fn main() {
    hexdump("Hi, bytes!\n".as_bytes());
    hexdump(&0x1234_5678_u32.to_le_bytes());
}
```

**What happens:**

1. `data.chunks(8)` cuts the bytes into pieces of at most 8; `enumerate` numbers the pieces, so the offset of a line is `line * 8`, printed as 8 hex digits.
2. Each byte is printed as two hex digits. A short last line is padded with spaces, so that the text column stays aligned.
3. The text column shows a byte as a character if it is a visible ASCII character or a space, and as `.` otherwise (for example the newline `0a`). ASCII (American Standard Code for Information Interchange) is the old table that gives each English letter, digit and punctuation mark a number from 0 to 127; Lesson 12.5 explains it in detail.

Run `cargo run`:

```text
00000000: 48 69 2c 20 62 79 74 65  Hi, byte
00000008: 73 21 0a                 s!.
00000000: 78 56 34 12              xV4.
```

Now change `to_le_bytes` to `to_be_bytes` and predict the last line before you run it.

## Exercises

### Exercise 1 — Convert by hand (easy)

Without a computer, convert: 45 to binary and to hex; `0x3f` to decimal; `0b1011_0110` to hex; `0b111_101_101` to octal. Then check your answers with a small program that prints them.

<details>
<summary>Hint</summary>

For 45 use the "divide by 2" method. For binary to hex, cut into groups of four bits from the right; for octal, into groups of three.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    println!("{:b} {:x}", 45, 45);
    println!("{}", 0x3f);
    println!("{:x}", 0b1011_0110);
    println!("{:o}", 0b111_101_101);
}
```

```text
101101 2d
63
b6
755
```

Why it works: 45 = 32 + 8 + 4 + 1 = `101101`; in groups of four `0010 1101` = `2d`. `0x3f` = 3×16 + 15 = 63. `1011 0110` = `b6`. `111 101 101` = `7 5 5`, the permission `rwxr-xr-x`.

</details>

### Exercise 2 — Your own base converter (medium)

In `src/lib.rs`, write `pub fn to_base(n: u64, base: u32) -> String` for bases 2 to 36 (digits `0`–`9`, then `a`–`z`). Write a test that compares your function with Rust's own formatting (`{:b}`, `{:o}`, `{:x}`, `to_string`) for several numbers, including `0` and `u64::MAX`, and a test for base 36.

<details>
<summary>Hint</summary>

Use the "divide and keep the remainders" method from Section 1. `char::from_digit(rest, base)` gives the digit character. The digits come out in reverse order; collect them and reverse. Treat `0` as a special case.

</details>

<details>
<summary>Answer</summary>

```rust
/// Writes `n` in the given base (2 to 36), using digits 0-9 and then a-z.
pub fn to_base(mut n: u64, base: u32) -> String {
    assert!((2..=36).contains(&base), "base must be from 2 to 36");
    if n == 0 {
        return "0".to_string();
    }
    let mut digits = Vec::new();
    while n > 0 {
        let rest = (n % base as u64) as u32;
        digits.push(char::from_digit(rest, base).unwrap());
        n /= base as u64;
    }
    digits.iter().rev().collect()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn matches_rust_formatting() {
        for n in [0u64, 1, 2, 7, 255, 256, 1000, u64::MAX] {
            assert_eq!(to_base(n, 2), format!("{n:b}"));
            assert_eq!(to_base(n, 8), format!("{n:o}"));
            assert_eq!(to_base(n, 16), format!("{n:x}"));
            assert_eq!(to_base(n, 10), n.to_string());
        }
    }

    #[test]
    fn base_36_round_trip() {
        let text = to_base(1_000_000, 36);
        assert_eq!(text, "lfls");
        assert_eq!(u64::from_str_radix(&text, 36), Ok(1_000_000));
    }
}
```

```text
running 2 tests
test tests::base_36_round_trip ... ok
test tests::matches_rust_formatting ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: each `n % base` is the lowest digit and each `n /= base` removes it, exactly like the division table in Section 1. The test uses the standard library as a trusted "oracle" (a known-correct answer to compare with), and `from_str_radix` checks the round trip.

</details>

### Exercise 3 — Read a file header (medium)

A made-up file format starts with 6 bytes: a big-endian `u16` version, then a big-endian `u32` length. Write `pub fn parse_header(bytes: &[u8; 6]) -> (u16, u32)` and a test with the bytes `00 02 00 00 01 2c`, which must give version 2 and length 300.

<details>
<summary>Hint</summary>

Build small arrays from the right bytes: `[bytes[0], bytes[1]]` for the version. Then use `u16::from_be_bytes` and `u32::from_be_bytes`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Reads a 6-byte header: a big-endian u16 version, then a big-endian u32 length.
pub fn parse_header(bytes: &[u8; 6]) -> (u16, u32) {
    let version = u16::from_be_bytes([bytes[0], bytes[1]]);
    let length = u32::from_be_bytes([bytes[2], bytes[3], bytes[4], bytes[5]]);
    (version, length)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn header_is_big_endian() {
        let bytes = [0x00, 0x02, 0x00, 0x00, 0x01, 0x2c];
        assert_eq!(parse_header(&bytes), (2, 300));
    }
}
```

Why it works: the type `&[u8; 6]` guarantees exactly six bytes, so the indexes cannot be out of range. `0x012c` = 1×256 + 2×16 + 12 = 300. If you used `from_le_bytes` by mistake, the test would fail with a huge length, which is exactly what the test is for.

</details>

### Exercise 4 — Human-readable sizes (hard)

Write `pub fn human_size(bytes: u64) -> String` that prints sizes with binary units and one decimal: `0 B`, `1023 B`, `1.0 KiB`, `1.5 KiB`, `16.0 MiB`, `5.0 TiB`. TiB is the largest unit (`3000.0 TiB` is fine). Write tests for all these cases.

<details>
<summary>Hint</summary>

Keep the units in an array `["B", "KiB", "MiB", "GiB", "TiB"]`. Below 1024, print the whole number with `B`. Otherwise, divide an `f64` by 1024 while it is at least 1024 and there is a larger unit. Format with `{value:.1}`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Formats a byte count with binary units: B, KiB, MiB, GiB, TiB.
pub fn human_size(bytes: u64) -> String {
    const UNITS: [&str; 5] = ["B", "KiB", "MiB", "GiB", "TiB"];
    if bytes < 1024 {
        return format!("{bytes} B");
    }
    let mut value = bytes as f64;
    let mut unit = 0;
    while value >= 1024.0 && unit < UNITS.len() - 1 {
        value /= 1024.0;
        unit += 1;
    }
    format!("{value:.1} {}", UNITS[unit])
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sizes() {
        assert_eq!(human_size(0), "0 B");
        assert_eq!(human_size(1023), "1023 B");
        assert_eq!(human_size(1024), "1.0 KiB");
        assert_eq!(human_size(1536), "1.5 KiB");
        assert_eq!(human_size(16 * 1024 * 1024), "16.0 MiB");
        assert_eq!(human_size(5 * 1024u64.pow(4)), "5.0 TiB");
        assert_eq!(human_size(3000 * 1024u64.pow(4)), "3000.0 TiB");
    }
}
```

Why it works: each division by 1024 moves one unit up. The condition `unit < UNITS.len() - 1` stops at TiB, so a very large size stays in TiB instead of running past the end of the array. The tests cover the border cases 1023 and 1024, where off-by-one bugs live.

</details>

## Quiz

1. How many different values can 12 bits hold?

<details>
<summary>Answer</summary>

2¹² = 4096 (from 0 to 4095 if unsigned).

</details>

2. Why is one hex digit exactly 4 bits?

<details>
<summary>Answer</summary>

Because 16 = 2⁴: four bits make 16 patterns, one for each hex digit `0`–`f`. So two hex digits are exactly one byte.

</details>

3. How many bytes are 1 MiB and 1 MB?

<details>
<summary>Answer</summary>

1 MiB = 2²⁰ = 1,048,576 bytes; 1 MB = 10⁶ = 1,000,000 bytes.

</details>

4. The `u32` value `0xAABBCCDD` is stored on your Mac. Which byte is at the lowest address?

<details>
<summary>Answer</summary>

`0xDD`. Your Mac is little-endian: the least significant byte comes first (`to_le_bytes()` gives `[dd, cc, bb, aa]`).

</details>

5. Why does `u32::from_str_radix("0x1f", 16)` return an error?

<details>
<summary>Answer</summary>

`from_str_radix` expects only digits of the base. `x` is not a hex digit, so it returns `InvalidDigit`. Remove the prefix first.

</details>

6. Which byte order do network protocols usually use, and what is it also called?

<details>
<summary>Answer</summary>

Big-endian, also called network byte order.

</details>

7. What does `println!("{:#06x}", 255)` print?

<details>
<summary>Answer</summary>

`0x00ff`. The width 6 includes the two characters of the prefix, so four hex digits are left.

</details>

## Summary

- A number is the same no matter how it is written; base 2, 8, 10 and 16 are only different spellings of the same bits.
- Convert from decimal by dividing by the base and reading the remainders backwards; convert between binary and hex with groups of four bits, binary and octal with groups of three.
- In Rust, print with `{:b}`, `{:o}`, `{:x}` (add `#` for a prefix, `0N` for zero padding) and parse with `from_str_radix`.
- 2¹⁰ = 1024: binary units (KiB, MiB, GiB) are powers of 1024, decimal units (KB, MB, GB) are powers of 1000.
- Byte order decides which byte of a number comes first in memory. Your Mac and x86-64 are little-endian; networks and many file formats are big-endian.
- At the borders of a program (files, network), choose the byte order on purpose with `to_le_bytes`/`to_be_bytes` and `from_le_bytes`/`from_be_bytes`, and test it with fixed bytes.

## Sources and further reading

- [Rust standard library: `u32`](https://doc.rust-lang.org/std/primitive.u32.html) — `from_str_radix`, `to_be_bytes`, `from_le_bytes`, `ilog2`, `BITS` and every other method used here (the same methods exist for all integer types).
- [Rust standard library: `std::fmt`](https://doc.rust-lang.org/std/fmt/index.html) — the full formatting syntax: `b`, `o`, `x`, `X`, `#`, width and fill.
- [NIST: Prefixes for binary multiples](https://physics.nist.gov/cuu/Units/binary.html) — NIST (National Institute of Standards and Technology, the US standards office) gives the official explanation of KiB, MiB and GiB.
- [Computer Systems: A Programmer's Perspective (CS:APP)](https://csapp.cs.cmu.edu/) — the classic textbook for this phase; chapter 2 covers number systems and byte order (the book uses C).

---

[🏠 Start](../START-HERE.md) · [← Phase 12 — Computer systems](README.md) · [📋 Phase 12](README.md) · [12.2 Integers in memory: two's complement and overflow →](02-integers-in-memory.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
