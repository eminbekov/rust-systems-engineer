<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 12.14 Project: your own memory allocator](14-project-your-own-allocator.md) · [📋 Phase 12](README.md) · [Phase 13 — Operating systems from the inside →](../13-operating-systems/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/15-checkpoint.md) <!-- nav -->

# Lesson 12.15 — Checkpoint

> **Phase 12 — Computer systems** · Lesson 15 of 15 · ⏱ about 105 minutes

## What you will learn

This checkpoint closes Phase 12 and teaches nothing new. It checks that you can use what the phase taught, and it shows you where to go back if something is still weak. You will:

- Answer 17 questions about numbers, floats, bits, text, the CPU (Central Processing Unit) and its assembly, caches, memory layout, linking and allocators.
- Solve 5 practical tasks; each one has a test or an output that tells you whether you are right.
- Find the lessons to read again in the table at the end.

## Before you start

- You finished [Lesson 12.14 — Project: your own memory allocator](14-project-your-own-allocator.md), or at least its milestones M1 to M3.
- Answer the questions without looking back first, then open the answers.
- Create a package for the practical tasks. Tasks 1 to 3 go into `src/lib.rs`, tasks 4 and 5 into `src/bin/`:

```bash
cd ~/projects/rust
cargo new --lib code/p12-checkpoint
mkdir code/p12-checkpoint/src/bin
```

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A pause at the end of a phase to test yourself before you go on. |
| **exhaustive test** | A test that checks every possible input, for example every `char` (Task 3). It is possible only when the inputs are few enough. |

## Self-test quiz

1. Write `0x2A` in binary and in decimal.

<details>
<summary>Answer</summary>

`0x2A` = `0b0010_1010` = 42. Each hex digit is four bits: `2` = `0010`, `A` = `1010`; 2 × 16 + 10 = 42. (Lesson 12.1)

</details>

2. In which order are the bytes of the `u32` value `0x1234_5678` stored on your Mac?

<details>
<summary>Answer</summary>

`78 56 34 12`: the Mac (arm64) is little-endian, so the least significant byte comes first, at the lowest address. `0x1234_5678_u32.to_le_bytes()` gives `[78, 56, 34, 12]` in hex. Network protocols usually use big-endian, `12 34 56 78`. (Lesson 12.1)

</details>

3. What does `-1i8 as u8` give, and why?

<details>
<summary>Answer</summary>

255. In two's complement, −1 is all ones (`1111_1111`). `as` between types of the same width keeps the bits and only changes their meaning, and as an unsigned number all ones is 255. (Lesson 12.2)

</details>

4. What are `200u8.checked_add(100)` and `200u8.wrapping_add(100)`? And if a `u8` variable `x` holds 200 at run time, what does `x + 100` do in debug and in release builds?

<details>
<summary>Answer</summary>

`checked_add` gives `None`; `wrapping_add` gives 44 (300 − 256). `x + 100` panics with "attempt to add with overflow" in debug builds, where overflow checks are on, and wraps silently to 44 in release builds. (Written with two constants, `200u8 + 100` does not even compile: the compiler sees the overflow.) (Lessons 12.2 and 12.8)

</details>

5. Why does `0.1 + 0.2` print `0.30000000000000004`, and how should a test compare such results?

<details>
<summary>Answer</summary>

0.1 and 0.2 cannot be written exactly as sums of powers of two, so both are rounded, and the sum is rounded again, to a value next to 0.3 but not the one closest to it. Compare with a tolerance, such as `(a - b).abs() <= 1e-12 * a.abs().max(b.abs())`, never with `==`. (Lesson 12.3)

</details>

6. What is `0b1011_0100 & (0b1011_0100 - 1)`, and what does the trick `x & (x - 1)` do in general?

<details>
<summary>Answer</summary>

`0b1011_0000` (176). Subtracting 1 turns the lowest `1` into `0` and the zeros below it into ones; the AND then keeps everything except that lowest `1`. So `x & (x - 1)` clears the lowest set bit, and it is 0 exactly when `x` is 0 or a power of two. (Lesson 12.4)

</details>

7. How many bytes and how many `char`s does the string `"é"` have? How many bytes does `'🦀'` take in UTF-8 (Unicode Transformation Format, 8-bit)?

<details>
<summary>Answer</summary>

`"é".len()` is 2 (U+00E9 needs two bytes in UTF-8) and `"é".chars().count()` is 1. The crab, U+1F980, is above U+FFFF, so it takes 4 bytes. (Lesson 12.5)

</details>

8. A loop adds numbers into one variable `total`. Why can splitting it into four independent sums make it faster, although the work is the same?

<details>
<summary>Answer</summary>

Each `total += x` must wait for the previous addition: a dependency chain, limited by the **latency** of one addition. Four independent sums let the out-of-order core keep several additions in flight, so the loop runs at the **throughput** of the adder instead. (Lesson 12.6)

</details>

9. On arm64, in which registers does a function receive its first two integer arguments, and where does it return its result? And on x86-64 Linux?

<details>
<summary>Answer</summary>

arm64: arguments in `x0` and `x1`, result in `x0`. x86-64 (System V calling convention): arguments in `rdi` and `rsi`, result in `rax`. Only `extern "C"` functions are guaranteed to follow these rules. (Lesson 12.7)

</details>

10. Name two things that make a release build a different program from a debug build.

<details>
<summary>Answer</summary>

Optimisation (`opt-level = 3`: inlining, constant folding, vectorisation), and checks that are switched off: overflow checks and `debug_assert!`. Bounds checks stay in both, unless the optimiser can prove they never fail. (Lesson 12.8)

</details>

11. An L1 (level 1 cache) hit costs about 1 ns (nanosecond) on your Mac. About how much does a load cost when the data is only in RAM (Random Access Memory), and why does summing a big array in order not pay that price for every element?

<details>
<summary>Answer</summary>

About 100–120 ns. In order, one 128-byte cache line brings 16 `u64` values at once, and the prefetcher sees the regular pattern and loads the next lines before they are needed. Only random, dependent accesses (pointer chasing) pay the full latency every time. (Lesson 12.9)

</details>

12. Two threads each increment their own counter, but the program gets slower when the second thread starts. The counters are fields of one small struct. What is happening, and how do you fix it?

<details>
<summary>Answer</summary>

False sharing: both counters lie in the same cache line, so every write by one core takes the line away from the other. Give each counter its own line, for example with `#[repr(align(128))]` or `crossbeam_utils::CachePadded`. (Lesson 12.10)

</details>

13. What is `size_of::<(u8, u32, u16)>()`? And why are `Option<Box<u8>>` and `Option<u64>` 8 and 16 bytes?

<details>
<summary>Answer</summary>

8: the compiler may reorder a tuple, so it can store `u32`, `u16`, `u8` and one byte of padding. `Box` is never null, so `Option<Box<u8>>` uses address 0 as `None` (a niche) and stays 8 bytes; every bit pattern of a `u64` is a valid number, so `Option<u64>` needs a separate tag, which with padding makes 16. (Lesson 12.11)

</details>

14. Your build fails with "Undefined symbols … referenced from …" on macOS, or "undefined reference to …" on Linux. Which tool reports it, and what are the two usual fixes?

<details>
<summary>Answer</summary>

The linker (`ld`), after compilation succeeded. A symbol that your code uses is not defined in any object file or library it was given. Fix: compile the code that defines it (for example with the `cc` crate) or link the library that contains it. (Lesson 12.12)

</details>

15. A program links fine but fails at start with "error while loading shared libraries: libfoo.so: cannot open shared object file". Who reports this, and which commands help you investigate?

<details>
<summary>Answer</summary>

The loader (`ld.so` on Linux), when the program starts: the executable names `libfoo.so` in its `NEEDED` list, but the loader cannot find the file in its search path. `readelf -d` shows what the program needs, `ldd` shows where each library is (or "not found"); `LD_LIBRARY_PATH` or installing the library fixes it. On macOS, `otool -L` lists the needed libraries. (Lesson 12.12)

</details>

16. Give two rules that every `GlobalAlloc` implementation must follow.

<details>
<summary>Answer</summary>

Any two of: return a block with at least the requested size and alignment, or null; never return a block that overlaps a live block; never allocate inside `alloc` or `dealloc` (no `println!`, no `Vec`); be safe when called from many threads at once, because it is shared through `&self`; `dealloc` gets exactly the pointer and layout of an earlier `alloc`. (Lessons 12.13 and 12.14)

</details>

17. In a free-list allocator, 32 bytes are free in total, but a request for 24 bytes fails. What is this called, and which technique reduces it?

<details>
<summary>Answer</summary>

External fragmentation: the free memory is split into blocks that are each smaller than 24 bytes. Coalescing neighbouring free blocks when they are freed (easy when the list is sorted by address) reduces it. (Lesson 12.13)

</details>

## Practical tasks

### Task 1 — Bytes in memory order

In `src/lib.rs`, write `pub fn to_hex_le(x: u32) -> String` that returns the four bytes of `x` in little-endian order as two-digit hex numbers separated by spaces: `to_hex_le(0x1234_5678)` is `"78 56 34 12"`. Add a test for that value and for 1.

<details>
<summary>Hint</summary>

`x.to_le_bytes()` gives `[u8; 4]`. `format!("{b:02x}")` prints one byte as two hex digits; collect into a `Vec<String>` and `join(" ")`.

</details>

<details>
<summary>Answer</summary>

```rust
/// The bytes of `x` in memory order on a little-endian machine, as hex.
pub fn to_hex_le(x: u32) -> String {
    x.to_le_bytes()
        .iter()
        .map(|b| format!("{b:02x}"))
        .collect::<Vec<_>>()
        .join(" ")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn hex_le_puts_the_lowest_byte_first() {
        assert_eq!(to_hex_le(0x1234_5678), "78 56 34 12");
        assert_eq!(to_hex_le(1), "01 00 00 00");
    }
}
```

`to_le_bytes` makes the result the same on every computer; reading the raw memory of `x` would only be little-endian on a little-endian CPU. (Lessons 12.1 and 12.4)

</details>

### Task 2 — An average that cannot overflow

Write `pub fn midpoint(a: u32, b: u32) -> u32`: the average of `a` and `b`, rounded down. `(a + b) / 2` overflows for big numbers; your version must not, and it must not use a wider type. Test it with `u32::MAX` and compare it with the standard library's `u32::midpoint` for a few pairs.

<details>
<summary>Hint</summary>

The bits that `a` and `b` share count fully (`a & b`); the bits where they differ count half (`(a ^ b) >> 1`). Neither part can overflow.

</details>

<details>
<summary>Answer</summary>

Add to `src/lib.rs` (the tests go into the same `mod tests`):

```rust
/// The average of `a` and `b`, rounded down, without overflow.
pub fn midpoint(a: u32, b: u32) -> u32 {
    (a & b) + ((a ^ b) >> 1)
}
```

```rust
    #[test]
    fn midpoint_never_overflows() {
        assert_eq!(midpoint(u32::MAX, u32::MAX), u32::MAX);
        assert_eq!(midpoint(u32::MAX, u32::MAX - 1), u32::MAX - 1);
        assert_eq!(midpoint(3, 4), 3);
        assert_eq!(midpoint(0, 10), 5);
    }

    #[test]
    fn midpoint_matches_the_standard_library() {
        for (a, b) in [(1, 2), (7, 100), (u32::MAX, 0), (12345, 67890)] {
            assert_eq!(midpoint(a, b), a.midpoint(b), "a = {a}, b = {b}");
        }
    }
```

Why it works: `a + b = 2 × (a & b) + (a ^ b)`, because shared bits are counted twice and differing bits once. Dividing by two gives `(a & b) + (a ^ b) / 2`, and each part is at most `u32::MAX`, as is their sum. (Lessons 12.2 and 12.4)

</details>

### Task 3 — UTF-8 length from the code point

Write `pub fn utf8_len(c: char) -> usize` that returns how many bytes `c` takes in UTF-8, using only its code point (`u32::from(c)`) and the ranges of UTF-8. Test it against `char::len_utf8` for **every** `char`.

<details>
<summary>Hint</summary>

Up to `0x7F`: 1 byte; up to `0x7FF`: 2; up to `0xFFFF`: 3; above: 4. `char::MIN..=char::MAX` is a range you can loop over; it skips the surrogates by itself.

</details>

<details>
<summary>Answer</summary>

```rust
/// How many bytes `c` takes in UTF-8, from its code point alone.
pub fn utf8_len(c: char) -> usize {
    match u32::from(c) {
        0..=0x7F => 1,
        0x80..=0x7FF => 2,
        0x800..=0xFFFF => 3,
        _ => 4,
    }
}
```

```rust
    #[test]
    fn utf8_len_matches_len_utf8_for_every_char() {
        for c in char::MIN..=char::MAX {
            assert_eq!(utf8_len(c), c.len_utf8(), "char U+{:04X}", u32::from(c));
        }
    }
```

All tests of tasks 1 to 3 together (`cargo test -q --lib`):

```text
running 4 tests
....
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.03s
```

The test checks all 1,112,064 `char` values in a few hundredths of a second, which is why an exhaustive test is the right choice here. (Lesson 12.5)

</details>

### Task 4 — A struct that fits the cache better

This `repr(C)` struct describes one measurement sample:

```rust
#[repr(C)]
struct SampleBefore {
    flag: bool,
    time: u64,
    id: u16,
    value: f32,
    kind: u8,
}
```

In `src/bin/task4.rs`, compute its size by hand, then write a `repr(C)` struct `Sample` with the same fields in a better order, add a `const` assertion for its size, and print how many of each fit into one 128-byte cache line.

<details>
<summary>Hint</summary>

Write down offsets: `flag` at 0, then `time` must start at a multiple of 8. Sort the fields by alignment, biggest first.

</details>

<details>
<summary>Answer</summary>

```rust
#[allow(dead_code)]
#[repr(C)]
struct SampleBefore {
    flag: bool,
    time: u64,
    id: u16,
    value: f32,
    kind: u8,
}

#[allow(dead_code)]
#[repr(C)]
struct Sample {
    time: u64,
    value: f32,
    id: u16,
    flag: bool,
    kind: u8,
}

const _: () = assert!(size_of::<Sample>() == 16);

fn main() {
    let line = 128;
    let before = size_of::<SampleBefore>();
    let after = size_of::<Sample>();
    println!("before: {before} bytes, {} per cache line", line / before);
    println!("after:  {after} bytes, {} per cache line", line / after);
}
```

```text
before: 32 bytes, 4 per cache line
after:  16 bytes, 8 per cache line
```

Why it works: in the old order `flag` (0) is followed by 7 padding bytes, `time` is at 8, `id` at 16, 2 padding bytes, `value` at 20, `kind` at 24, and 7 padding bytes round the size up to 32. In the new order the fields fill 8 + 4 + 2 + 1 + 1 = 16 bytes with no padding at all. Twice as many samples per cache line means half the memory traffic for a loop over all samples. (Lessons 12.10 and 12.11)

</details>

### Task 5 — Fewer allocations

In `src/bin/task5.rs`, install a counting global allocator (Lesson 12.13) and count the allocations of two functions that add up the number of digits of `0..1000`: one uses `i.to_string()` in the loop, the other reuses one `String` buffer with `buffer.clear()` and `write!(buffer, "{i}")`. Print both counts.

<details>
<summary>Hint</summary>

`write!` into a `String` needs `use std::fmt::Write;`. Read the counter before and after each function. A `String` created with `String::with_capacity(10)` never has to grow for numbers below 1000.

</details>

<details>
<summary>Answer</summary>

```rust
use std::alloc::{GlobalAlloc, Layout, System};
use std::fmt::Write;
use std::sync::atomic::{AtomicUsize, Ordering};

struct Counting;

static ALLOCATIONS: AtomicUsize = AtomicUsize::new(0);

// SAFETY: every call is passed on unchanged to `System`.
unsafe impl GlobalAlloc for Counting {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        ALLOCATIONS.fetch_add(1, Ordering::Relaxed);
        // SAFETY: the caller upholds the rules of `GlobalAlloc::alloc`.
        unsafe { System.alloc(layout) }
    }

    unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout) {
        // SAFETY: `ptr` was allocated by `System` with this `layout`.
        unsafe { System.dealloc(ptr, layout) }
    }
}

#[global_allocator]
static GLOBAL: Counting = Counting;

fn digits_new_strings(n: u32) -> usize {
    let mut total = 0;
    for i in 0..n {
        total += i.to_string().len();
    }
    total
}

fn digits_one_buffer(n: u32) -> usize {
    let mut total = 0;
    let mut buffer = String::with_capacity(10);
    for i in 0..n {
        buffer.clear();
        write!(buffer, "{i}").expect("writing to a String cannot fail");
        total += buffer.len();
    }
    total
}

fn main() {
    let before = ALLOCATIONS.load(Ordering::Relaxed);
    let a = digits_new_strings(1000);
    let middle = ALLOCATIONS.load(Ordering::Relaxed);
    let b = digits_one_buffer(1000);
    let after = ALLOCATIONS.load(Ordering::Relaxed);
    println!("new strings: {a} digits, {} allocations", middle - before);
    println!("one buffer:  {b} digits, {} allocations", after - middle);
}
```

```text
new strings: 2890 digits, 1000 allocations
one buffer:  2890 digits, 1 allocations
```

Why it works: every `to_string()` creates a new `String` on the heap, which is freed again at the end of the loop body. `clear()` sets the length to 0 but keeps the capacity, so the buffer is allocated once and reused 1000 times. The result (2890 digits) proves both versions did the same work. (Lesson 12.13)

</details>

## If you got something wrong

| Topic | Lesson to review |
|---|---|
| Number systems, hex, byte order | [12.1 — Bits, bytes and number systems](01-bits-bytes-and-number-systems.md) |
| Two's complement, overflow, `checked_*` / `wrapping_*` | [12.2 — Integers in memory](02-integers-in-memory.md) |
| Rounding, NaN (Not a Number), comparing floats | [12.3 — Floating-point numbers](03-floating-point-numbers.md) |
| Masks, shifts, bit tricks, `align_up` | [12.4 — Bitwise operations and bit tricks](04-bitwise-operations.md) |
| ASCII (American Standard Code for Information Interchange), Unicode, UTF-8 byte patterns | [12.5 — Text encoding](05-text-encoding-ascii-and-utf8.md) |
| Pipeline, latency and throughput, branch prediction | [12.6 — How the CPU runs instructions](06-how-the-cpu-runs-instructions.md) |
| Registers, calling convention, reading `.s` files | [12.7 — Reading assembly](07-reading-assembly.md) |
| MIR (Mid-level Intermediate Representation) and LLVM IR (the intermediate representation of the LLVM compiler back end), debug versus release, bounds checks | [12.8 — From Rust to machine code](08-from-rust-to-machine-code.md) |
| Cache levels, cache lines, prefetching | [12.9 — The memory hierarchy and caches](09-memory-hierarchy-and-caches.md) |
| Loop order, `Vec` versus lists, struct of arrays, false sharing | [12.10 — Writing cache-friendly code](10-cache-friendly-code.md) |
| Size, alignment, padding, `repr`, niches | [12.11 — Memory layout of Rust types](11-memory-layout-of-rust-types.md) |
| Symbols, linker errors, Mach-O and ELF (the executable formats of macOS and Linux), the loader | [12.12 — Linking and loading programs](12-linking-and-loading.md) |
| `Layout`, `GlobalAlloc`, bump and free-list allocators | [12.13 — How memory allocators work](13-how-memory-allocators-work.md) |
| Building and testing an allocator, Miri | [12.14 — Project: your own memory allocator](14-project-your-own-allocator.md) |

## What's next

You can now look below the language: at the bits of a number, the instructions the CPU runs, the caches that decide how fast memory is, the layout of every type, the linker and the loader, and the allocator behind every `Box`. Phase 13, *Operating systems from the inside*, goes one level further down, to the kernel: processes, system calls, file descriptors, virtual memory and `mmap` (the call your allocator's big brothers use to get memory), and finally namespaces and cgroups, the building blocks of containers. Most of it runs on Linux, so keep your VM (virtual machine) `dev` from Phase 11 or a Docker container ready.

Before you continue, commit your work in `code/p12-checkpoint` and `code/p12-allocator`, and add a line to your learning journal: which of the measurements in this phase surprised you most?

## Sources and further reading

- [Computer Systems: A Programmer's Perspective (CS:APP)](https://csapp.cs.cmu.edu/) — the classic book that covers the whole phase: data representation, machine code, caches, linking and memory allocation.
- [Ulrich Drepper — What Every Programmer Should Know About Memory](https://www.akkadia.org/drepper/cpumemory.pdf) — the long reference on memory and caches.
- [The Rust Reference — Type layout](https://doc.rust-lang.org/reference/type-layout.html) — layout rules, to check your answers to questions 13 and task 4.
- [std::alloc::GlobalAlloc](https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html) — the allocator rules behind questions 16 and 17.

---

[🏠 Start](../START-HERE.md) · [← 12.14 Project: your own memory allocator](14-project-your-own-allocator.md) · [📋 Phase 12](README.md) · [Phase 13 — Operating systems from the inside →](../13-operating-systems/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
