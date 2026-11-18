<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 12.10 Writing cache-friendly code](10-cache-friendly-code.md) · [📋 Phase 12](README.md) · [12.12 Linking and loading programs →](12-linking-and-loading.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/11-memory-layout-of-rust-types.md) <!-- nav -->

# Lesson 12.11 — Memory layout of Rust types

> **Phase 12 — Computer systems** · Lesson 11 of 15 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What size and alignment are, and why the CPU cares about alignment.
- How Rust places the fields of a struct, where padding comes from, and why Rust may reorder fields.
- What `#[repr(C)]`, `#[repr(packed)]`, `#[repr(align(N))]` and `#[repr(transparent)]` change.
- How enums are stored, and how the "niche" trick makes `Option<&T>` as small as `&T`.
- What fat pointers and zero-sized types are, and how to check a layout with tests that run at compile time.

## Before you start

- You finished [Lesson 12.10 — Writing cache-friendly code](10-cache-friendly-code.md).
- You know `size_of` and the padding surprise from [Lesson 03.1 — Memory: the stack and the heap](../03-ownership-and-borrowing/01-memory-stack-and-heap.md), trait objects from [Lesson 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md), raw pointers and undefined behaviour from [Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md), and `#[repr(C)]` for FFI (Foreign Function Interface, calling C code) from [Lesson 06.10](../06-rust-advanced/10-ffi-calling-c.md).
- Create the package and, as in Lesson 12.1, put each example into `src/main.rs` (replace the old one) and run it with `cargo run`:

```bash
cd ~/projects/rust
cargo new code/p12-layout
cd code/p12-layout
```

## New words

| Word | Meaning |
|---|---|
| **layout** | Where the bytes of a value go: its size, its alignment, and the position of each field. |
| **size** | How many bytes one value of a type takes, including padding. `size_of::<T>()` returns it. |
| **alignment** | The number that every address of a value of this type must be a multiple of. `align_of::<T>()` returns it. It is always a power of two. |
| **padding** | Unused bytes that the compiler inserts between or after fields, so that every field is aligned. |
| **offset** | The distance in bytes from the start of a struct to one of its fields. `std::mem::offset_of!(Type, field)` returns it. |
| **representation** (`repr`) | The set of layout rules for a type. Rust's default is `repr(Rust)`; the attribute `#[repr(...)]` picks another one. |
| **discriminant** (tag) | The hidden number inside an enum value that says which variant it is. |
| **niche** | A bit pattern that a type never uses, such as the value 0 for a reference. The compiler may store an enum's tag there for free. |
| **thin / fat pointer** | A thin pointer is just an address (8 bytes). A fat pointer is an address plus extra data: a length for slices and `str`, a vtable pointer for `dyn Trait`. |
| **vtable** (virtual method table) | A table of function pointers for one type and one trait. A `&dyn Trait` carries a pointer to it (Lesson 06.1). |
| **ZST** (Zero-Sized Type) | A type whose values take 0 bytes, such as `()` or `struct Marker;`. |
| **UB** (Undefined Behaviour) | Breaking a rule that the compiler relies on; the program may then do anything (Lesson 06.6). |
| **ABI** (Application Binary Interface) | The low-level rules for how compiled code passes and stores values: layouts, registers, calling conventions. |
| **const assertion** | An `assert!` inside a `const` item. It is checked while compiling, so a wrong layout stops the build. |

## 1. Size and alignment

Every type has two numbers. Its **size** is how many bytes one value takes. Its **alignment** says where a value may start: the address must be a multiple of the alignment. A `u32` has alignment 4, so it may start at address 0, 4, 8, 12, … but not at 5.

Why? The CPU reads memory in aligned pieces, and the hardware is built for values that do not cross the borders of those pieces. An aligned `u64` can be loaded by one simple instruction, and it can never be split across two cache lines (Lesson 12.9). Some CPUs refuse misaligned loads entirely; others handle them, but more slowly, and atomic operations (Lesson 05.10) need alignment everywhere. Rust therefore **always** keeps values aligned, and creating a misaligned reference is UB (Undefined Behaviour).

Put this into `src/main.rs`:

```rust
/// Prints the size and the alignment of the type `T`.
fn row<T>(name: &str) {
    println!("{name:<9}{:>4}  {:>5}", size_of::<T>(), align_of::<T>());
}

fn main() {
    println!("type     size  align");
    row::<u8>("u8");
    row::<u16>("u16");
    row::<u32>("u32");
    row::<u64>("u64");
    row::<u128>("u128");
    row::<f64>("f64");
    row::<bool>("bool");
    row::<char>("char");
    row::<usize>("usize");
    row::<[u16; 3]>("[u16; 3]");
}
```

```text
type     size  align
u8          1      1
u16         2      2
u32         4      4
u64         8      8
u128       16     16
f64         8      8
bool        1      1
char        4      4
usize       8      8
[u16; 3]    6      2
```

**What happens:**

1. `row::<T>` is a generic function (Lesson 05.1): `size_of::<T>()` and `align_of::<T>()` are both in the prelude, so no `use` line is needed.
2. For the number types, alignment equals size. That is the usual rule on 64-bit computers, both on arm64 (your Mac) and on x86-64.
3. `usize` is 8 bytes because addresses are 64 bits wide. On a 32-bit target it would be 4.
4. An array has the alignment of its element and the size of all elements together: `[u16; 3]` is 6 bytes, aligned to 2.

## 2. Structs and padding

A struct must keep every field aligned. If a small field is followed by a bigger one, there is a gap: **padding**. The struct as a whole gets the largest alignment of its fields, and its size is rounded up to a multiple of that alignment, so that the elements of an array of structs stay aligned too.

Rust's default representation, `repr(Rust)`, is allowed to **reorder fields** to make the padding smaller. `#[repr(C)]` forbids that: fields stay in the order you wrote, with the same rules a C compiler uses. Compare the two with the same four fields:

```rust
use std::mem::offset_of;

struct Mixed {
    a: u8,
    b: u32,
    c: u16,
    d: u64,
}

#[repr(C)]
struct MixedC {
    a: u8,
    b: u32,
    c: u16,
    d: u64,
}

fn main() {
    println!("Mixed:  size {}, align {}", size_of::<Mixed>(), align_of::<Mixed>());
    println!(
        "        a at {}, b at {}, c at {}, d at {}",
        offset_of!(Mixed, a),
        offset_of!(Mixed, b),
        offset_of!(Mixed, c),
        offset_of!(Mixed, d)
    );
    println!("MixedC: size {}, align {}", size_of::<MixedC>(), align_of::<MixedC>());
    println!(
        "        a at {}, b at {}, c at {}, d at {}",
        offset_of!(MixedC, a),
        offset_of!(MixedC, b),
        offset_of!(MixedC, c),
        offset_of!(MixedC, d)
    );
}
```

```text
Mixed:  size 16, align 8
        a at 14, b at 8, c at 12, d at 0
MixedC: size 24, align 8
        a at 0, b at 4, c at 8, d at 16
```

**What happens:**

1. `offset_of!(Type, field)` is a macro from `std::mem` that returns the byte position of a field inside the struct.
2. `MixedC` keeps the written order. `a` (1 byte) is at 0; `b` needs a multiple of 4, so 3 bytes of padding come first and `b` is at 4; `c` follows at 8; `d` needs a multiple of 8, so 6 bytes of padding push it to 16. Total: 24 bytes, 9 of them wasted.
3. `Mixed` uses the default representation, and the compiler sorted the fields from the biggest alignment to the smallest: `d` at 0, `b` at 8, `c` at 12, `a` at 14, and one padding byte at the end to reach a multiple of 8. Total: 16 bytes.

```text
   MixedC (repr(C), 24 bytes)

   offset: 0    1    2    3    4    5    6    7    8    9   10   11   12 … 15  16 … 23
          [a ][pad][pad][pad][ b                ][ c      ][pad][pad][pad …pad][ d      ]

   Mixed (repr(Rust), 16 bytes)

   offset: 0 … 7          8 … 11         12   13   14   15
          [ d          ][ b          ][ c      ][ a ][pad]
```

> ⚠️ **Common mistake:** Assuming that a struct's fields are in memory in the order you wrote them. With the default representation, the order (and therefore the offsets) is **not guaranteed** and may even change between compiler versions. If anything outside your Rust program depends on the layout (C code, a file format, a network packet, hardware), use `#[repr(C)]`.

> ⚡ **Performance:** Smaller structs mean more values per cache line (Lesson 12.10). With `repr(Rust)` the compiler already sorts fields for you. With `repr(C)`, sort them yourself: biggest alignment first.

## 3. Choosing a representation

The `#[repr(...)]` attribute picks the layout rules. You have already met two of them:

| Attribute | What it does | Use it for |
|---|---|---|
| (none) = `repr(Rust)` | Compiler chooses order and padding; nothing is guaranteed except that every field is aligned | Almost all Rust types |
| `#[repr(C)]` | Fields in written order, C padding rules | FFI (Lesson 06.10), file and network formats |
| `#[repr(align(N))]` | Alignment at least `N` (a power of two); size rounded up to a multiple of `N` | Giving a value its own cache line (Lesson 12.10) |
| `#[repr(packed)]` | Alignment 1: no padding at all; fields may be misaligned | Rare: exact byte formats where no padding is allowed |
| `#[repr(transparent)]` | A struct with one real field gets exactly that field's layout and ABI | Wrapper types passed to C, such as `struct Fd(i32)` |
| `#[repr(u8)]`, `#[repr(i32)]`, … | For enums: the type of the discriminant | Enums shared with C, or stored as one byte |

`repr(packed)` deserves a warning. Without padding, a `u32` field can end up at an odd address, and a reference `&u32` to it would be misaligned, which is UB. So Rust does not let you take such a reference. Formatting macros like `println!` take references to their arguments, so even printing a field fails:

```rust
#[repr(C, packed)]
struct Packed {
    a: u8,
    b: u32,
}

fn main() {
    let p = Packed { a: 1, b: 2 };
    println!("a {}, b {}", p.a, p.b);
}
```

```text
error[E0793]: reference to field of packed struct is unaligned
 --> src/main.rs:9:33
  |
9 |     println!("a {}, b {}", p.a, p.b);
  |                                 ^^^
  |
  = note: this struct is 1-byte aligned, but the type of this field may require higher alignment
  = note: creating a misaligned reference is undefined behavior (even if that reference is never dereferenced)
  = help: copy the field contents to a local variable, or replace the reference with a raw pointer and use `read_unaligned`/`write_unaligned` (loads and stores via `*p` must be properly aligned even when using raw pointers)
```

**What happens:**

1. `#[repr(C, packed)]` combines two rules: written order, and no padding. `Packed` is 5 bytes with alignment 1.
2. `p.a` is fine: a `u8` has alignment 1, so it can never be misaligned.
3. `p.b` is a `u32` at offset 1. `println!` would need `&p.b`, a misaligned reference, and the compiler refuses with E0793.
4. The help line gives the fix: copy the value into an ordinary local variable first. Copying reads the bytes without creating a reference.

With the fix, `let b = p.b;` followed by `println!("a {}, b {}", p.a, b);`, the program prints `a 1, b 2`, and `size_of::<Packed>()` is 5.

> 🔒 **Security:** When you parse bytes from a file or the network, do not cast a pointer to the buffer into a pointer to a `repr(C)` or `repr(packed)` struct: the buffer may be misaligned, too short, or contain values that are invalid for the field types (a `bool` that is 7). Read the fields one by one with `u32::from_le_bytes` and friends, as in Lesson 12.1, and check every value.

## 4. Enums, discriminants and niches

An enum value must remember which variant it is. In general it stores a **discriminant** (tag) next to the data of the largest variant. Run this program (the `#[allow(dead_code)]` lines silence warnings about variants we never create):

```rust
use std::num::NonZeroU32;

#[allow(dead_code)] // we only measure these types
enum Shape {
    Circle(f64),
    Rect(f64, f64),
    Empty,
}

#[allow(dead_code)]
#[repr(u8)]
enum Color {
    Red = 1,
    Green = 2,
    Blue = 4,
}

fn main() {
    println!("Shape:              {:>2} bytes", size_of::<Shape>());
    println!("Color:              {:>2} bytes, Blue = {}", size_of::<Color>(), Color::Blue as u8);
    println!("u32:                {:>2} bytes", size_of::<u32>());
    println!("Option<u32>:        {:>2} bytes", size_of::<Option<u32>>());
    println!("NonZeroU32:         {:>2} bytes", size_of::<NonZeroU32>());
    println!("Option<NonZeroU32>: {:>2} bytes", size_of::<Option<NonZeroU32>>());
    println!("&u32:               {:>2} bytes", size_of::<&u32>());
    println!("Option<&u32>:       {:>2} bytes", size_of::<Option<&u32>>());
    println!("Option<Box<u32>>:   {:>2} bytes", size_of::<Option<Box<u32>>>());
    println!("Option<bool>:       {:>2} bytes", size_of::<Option<bool>>());
    println!("Option<String>:     {:>2} bytes", size_of::<Option<String>>());

    let none: Option<&u32> = None;
    // SAFETY: Option<&u32> is guaranteed to have the same size as a pointer,
    // and None is guaranteed to be stored as the address 0.
    let bits: usize = unsafe { std::mem::transmute(none) };
    println!("None as Option<&u32> is stored as {bits}");
}
```

```text
Shape:              24 bytes
Color:               1 bytes, Blue = 4
u32:                 4 bytes
Option<u32>:         8 bytes
NonZeroU32:          4 bytes
Option<NonZeroU32>:  4 bytes
&u32:                8 bytes
Option<&u32>:        8 bytes
Option<Box<u32>>:    8 bytes
Option<bool>:        1 bytes
Option<String>:     24 bytes
None as Option<&u32> is stored as 0
```

**What happens:**

1. `Shape` needs room for its largest variant, `Rect` (two `f64`, 16 bytes), plus a tag. The tag takes 1 byte, but padding rounds the total up to 24, a multiple of 8.
2. `Color` has no data, only a tag. `#[repr(u8)]` makes the tag a `u8` with exactly the values you wrote, so `Color::Blue as u8` is 4.
3. `Option<u32>` is 8 bytes: every one of the 2³² bit patterns of a `u32` is a valid number, so there is no free pattern to mean `None`. The tag needs its own space: 4 bytes plus padding.
4. `NonZeroU32` is a `u32` that can never be 0. That unused pattern is a **niche**: `Option<NonZeroU32>` stores `None` as 0 and stays 4 bytes.
5. References and `Box` can never be null, so `Option<&u32>` and `Option<Box<u32>>` use the address 0 for `None` and stay 8 bytes. The `transmute` (reinterpret the bits as another type; Lesson 06.6) shows that `None` really is stored as 0. The Rust documentation guarantees this for `Option` of references, `Box` and `NonZero` types, which is why the `unsafe` block is sound here; do not do this with other types.
6. A `bool` uses only the patterns 0 and 1, so `Option<bool>` stores `None` as 2 in the same byte. `Option<String>` is still 24 bytes: the compiler found a niche inside `String` too (its capacity can never be larger than `isize::MAX`).

> 💡 **Tip:** Make "impossible" values impossible in the type, and the niche comes for free. An id that is never 0 can be a `NonZeroU32` instead of a `u32`; then `Option<Id>` costs no extra space. Exercise 3 builds such a type.

## 5. Thin pointers, fat pointers and zero-sized types

Most pointers (`&T`, `&mut T`, `Box<T>`, `*const T`) are one address: 8 bytes. But some types do not have a size known at compile time: a slice `[T]`, a `str`, a `dyn Trait`. A pointer to such a type needs extra information and becomes a **fat pointer** of two words.

```rust
use std::marker::PhantomData;

trait Speak {
    fn speak(&self) -> String;
}

struct Dog;

impl Speak for Dog {
    fn speak(&self) -> String {
        String::from("woof")
    }
}

fn main() {
    println!("{:<16}{:>2} bytes", "&u64", size_of::<&u64>());
    println!("{:<16}{:>2} bytes", "&[u64]", size_of::<&[u64]>());
    println!("{:<16}{:>2} bytes", "&str", size_of::<&str>());
    println!("{:<16}{:>2} bytes", "&dyn Speak", size_of::<&dyn Speak>());
    println!("{:<16}{:>2} bytes", "Box<dyn Speak>", size_of::<Box<dyn Speak>>());
    println!("{:<16}{:>2} bytes", "Vec<u64>", size_of::<Vec<u64>>());
    println!("{:<16}{:>2} bytes", "Dog", size_of::<Dog>());
    println!("{:<16}{:>2} bytes", "()", size_of::<()>());
    println!("{:<16}{:>2} bytes", "PhantomData<u64>", size_of::<PhantomData<u64>>());

    let animal: &dyn Speak = &Dog;
    println!("the dog says {}", animal.speak());

    let units = vec![(); 1_000_000];
    println!("a Vec of {} units, capacity {}", units.len(), units.capacity());
}
```

```text
&u64             8 bytes
&[u64]          16 bytes
&str            16 bytes
&dyn Speak      16 bytes
Box<dyn Speak>  16 bytes
Vec<u64>        24 bytes
Dog              0 bytes
()               0 bytes
PhantomData<u64> 0 bytes
the dog says woof
a Vec of 1000000 units, capacity 18446744073709551615
```

**What happens:**

1. `&u64` is a thin pointer: just the address.
2. `&[u64]` and `&str` are fat: address + length (the number of elements or bytes). That is how `slice.len()` works without looking at memory.
3. `&dyn Speak` and `Box<dyn Speak>` are fat too: address of the data + address of the vtable for `Dog` as `Speak`. A call like `animal.speak()` looks up the function in that table.
4. `Vec<u64>` is not a pointer type but a struct of three words: pointer, capacity and length (the same as `String`, Lesson 03.1).
5. `Dog`, `()` and `PhantomData<u64>` are **zero-sized types**. They exist for the type checker but take no memory. `PhantomData<T>` (from `std::marker`) is a marker that says "this struct logically owns a `T`" without storing one.
6. A `Vec` of a million `()` never allocates: there is nothing to store. Its capacity reports `usize::MAX` (18446744073709551615), because any number of zero-sized values fits in zero bytes.

```text
   &[u64] (16 bytes, on the stack)          the u64 values (somewhere else)
   ┌────────────┬────────────┐              ┌────┬────┬────┐
   │  address ──┼──────────────────────────►│ 10 │ 20 │ 30 │
   ├────────────┼────────────┤              └────┴────┴────┘
   │  length: 3 │
   └────────────┘

   &dyn Speak (16 bytes)                    vtable for (Dog, Speak)
   ┌────────────┐                           ┌───────────────────────┐
   │ data addr ─┼──► the Dog value          │ drop, size, align      │
   │ vtable  ───┼──────────────────────────►│ speak → Dog::speak     │
   └────────────┘                           └───────────────────────┘
```

## 6. Testing a layout

When a layout matters (a file format, FFI, a struct that must fit a cache line), write it down as checks that the compiler enforces. Put this into `src/lib.rs`:

```rust
/// A network packet header. The layout must match the wire format exactly.
#[repr(C)]
pub struct Header {
    pub version: u8,
    pub flags: u8,
    pub length: u16,
    pub id: u32,
    pub checksum: u64,
}

// Checked by the compiler: the build fails if the layout changes.
const _: () = assert!(size_of::<Header>() == 16);
const _: () = assert!(align_of::<Header>() == 8);

#[cfg(test)]
mod tests {
    use super::*;
    use std::mem::offset_of;

    #[test]
    fn header_fields_are_where_the_format_says() {
        assert_eq!(offset_of!(Header, version), 0);
        assert_eq!(offset_of!(Header, flags), 1);
        assert_eq!(offset_of!(Header, length), 2);
        assert_eq!(offset_of!(Header, id), 4);
        assert_eq!(offset_of!(Header, checksum), 8);
    }
}
```

**What happens:**

1. `const _: () = …;` is a constant without a name. Its value is computed **while compiling**. `assert!` inside it stops compilation if the condition is false. These checks cost nothing at run time.
2. The test checks the offset of every field with `offset_of!`. Run it with `cargo test`:

```text
running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(That is `cargo test -q --lib`.) Now change the layout by mistake: add a field `pub ttl: u8,` after `checksum` and run `cargo build`. The size grows to 24 and the build stops:

```text
error[E0080]: evaluation panicked: assertion failed: size_of::<Header>() == 16
  --> src/lib.rs:13:15
   |
13 | const _: () = assert!(size_of::<Header>() == 16);
   |               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ evaluation of `_` failed here
```

Remove the field again. Constant checks also work for other targets without running anything. With the standard library for x86-64 Linux installed (`rustup target add x86_64-unknown-linux-gnu`, as in Lesson 12.7), you can check the same layout for it:

```bash
cargo check --lib --target x86_64-unknown-linux-gnu
```

```text
    Checking p12-layout v0.1.0 (~/projects/rust/code/p12-layout)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.20s
```

The assertions hold on x86-64 too. That is expected: both are 64-bit targets with the same alignment rules for these types.

> 🧪 **Testing:** Use `const` assertions for sizes and alignments (they fail the build, even in `cargo check`), and normal tests with `offset_of!` for field positions. Write them for every `repr(C)` type that crosses a border: C code, files, network, hardware.

## Common mistakes

**1. Relying on the default layout.** Code that reads a `repr(Rust)` struct as raw bytes, or `transmute`s between two such structs with "the same fields", depends on an order the compiler never promised. Add `#[repr(C)]` first.

**2. Expecting `Option<T>` to be free for every `T`.** It is free only when `T` has a niche (references, `Box`, `NonZero*`, `bool`, `char`, many enums). `Option<u32>` and `Option<u64>` are twice as big as the number itself.

**3. Reading padding bytes.** Padding has no defined value. Viewing a struct's memory as a `&[u8]` with `unsafe` code and reading it is UB; Miri (Lesson 06.6) stops with "memory is uninitialized". Serialize field by field instead.

## Try it yourself

1. Run the program from section 1, then add rows for `i8`, `u128`, `(u8, u16)` and `[u64; 4]`. Predict each line before you run it.
2. Run the program from section 2. Add `#[repr(C)]` to `Mixed` and watch its size grow to 24. Then reorder the fields of `MixedC` (biggest first) until it is 16 bytes.
3. Run the program from section 4 and add `Option<char>`, `Option<Option<bool>>` and `Result<u32, ()>`. Explain each size.
4. Create `src/lib.rs` from section 6, run `cargo test`, break the layout on purpose, and read the error.

## Exercises

### Exercise 1 — Predict the sizes (easy)

Write down the size of each type, then check with a program that prints them in this order: `(u8, u64, u8)`, `[u16; 3]`, `Option<u8>`, `Option<Box<[u8]>>`, `&&str`, `Option<Option<bool>>`.

<details>
<summary>Hint</summary>

Tuples use the default representation, so the compiler may reorder them. `Box<[u8]>` is a fat pointer. `&&str` is a reference to a fat pointer, so it is thin.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    println!("{}", size_of::<(u8, u64, u8)>());
    println!("{}", size_of::<[u16; 3]>());
    println!("{}", size_of::<Option<u8>>());
    println!("{}", size_of::<Option<Box<[u8]>>>());
    println!("{}", size_of::<&&str>());
    println!("{}", size_of::<Option<Option<bool>>>());
}
```

```text
16
6
2
16
8
1
```

Why it works: the tuple becomes `u64, u8, u8` + 6 bytes of padding = 16. `[u16; 3]` = 3 × 2. `Option<u8>` has no niche: 1 byte tag + 1 byte value. `Box<[u8]>` is a fat pointer (16) whose address is never null, so `Option` is free. `&&str` points to a `&str`, and a pointer to a sized type is thin. `bool` uses only 0 and 1, so `Some(None)` and `None` take two other patterns of the same byte (with Rust 1.100 they are 2 and 255; the exact values are the compiler's choice).

</details>

### Exercise 2 — Shrink a C struct (medium)

This `repr(C)` struct is sent to a C library, so it must stay `repr(C)`:

```rust
#[repr(C)]
struct EventBefore {
    active: bool,
    time: u64,
    kind: u8,
    value: u32,
}
```

Its size is 24 bytes. Reorder the fields into a new struct `Event` so that it takes 16 bytes, and add a `const` assertion that fails the build if it ever grows again. (In real life the C side must use the new order too.)

<details>
<summary>Hint</summary>

Sort the fields by alignment, biggest first: 8, then 4, then 1 and 1. Then count: 8 + 4 + 1 + 1 = 14, rounded up to a multiple of 8.

</details>

<details>
<summary>Answer</summary>

Add this below `EventBefore` from the task (put `#[allow(dead_code)]` on `EventBefore` too, because its fields are never read):

```rust
#[allow(dead_code)]
#[repr(C)]
struct Event {
    time: u64,
    value: u32,
    active: bool,
    kind: u8,
}

const _: () = assert!(size_of::<Event>() == 16);

fn main() {
    println!("before: {} bytes", size_of::<EventBefore>());
    println!("after:  {} bytes", size_of::<Event>());
}
```

```text
before: 24 bytes
after:  16 bytes
```

Why it works: in the old order, `active` (offset 0) is followed by 7 padding bytes before `time` (offset 8), and `kind` (offset 16) by 3 padding bytes before `value` (offset 20): 10 wasted bytes. In the new order `time` is at 0, `value` at 8, `active` at 12, `kind` at 13, and the only padding is 2 bytes at the end. A third of the memory (and of the cache traffic for an array of events) is saved.

</details>

### Exercise 3 — An id with a free `None` (medium)

Write `pub struct UserId(NonZeroU32)` with `UserId::new(raw: u32) -> Option<UserId>` (returns `None` for 0) and `get(self) -> u32`. Write two tests: `Option<UserId>` is exactly as big as a `u32`, and `UserId::new(0)` is `None` while `UserId::new(7)` gives back 7.

<details>
<summary>Hint</summary>

`NonZeroU32::new(raw)` already returns `Option<NonZeroU32>`; `.map(UserId)` wraps it. A tuple struct's name works as a function. Derive `Debug` and `PartialEq` so that `assert_eq!` can compare and print values.

</details>

<details>
<summary>Answer</summary>

```rust
use std::num::NonZeroU32;

/// A user id. Zero is never a valid id, so the compiler can use it for `None`.
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct UserId(NonZeroU32);

impl UserId {
    pub fn new(raw: u32) -> Option<UserId> {
        NonZeroU32::new(raw).map(UserId)
    }

    pub fn get(self) -> u32 {
        self.0.get()
    }
}

fn main() {
    let id = UserId::new(42).expect("42 is not zero");
    println!("id {} takes {} bytes", id.get(), size_of::<UserId>());
    println!("Option<UserId> takes {} bytes", size_of::<Option<UserId>>());
    println!("UserId::new(0) = {:?}", UserId::new(0));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn option_of_user_id_costs_no_extra_space() {
        assert_eq!(size_of::<Option<UserId>>(), size_of::<u32>());
    }

    #[test]
    fn zero_is_not_a_user_id() {
        assert_eq!(UserId::new(0), None);
        assert_eq!(UserId::new(7).map(UserId::get), Some(7));
    }
}
```

```text
id 42 takes 4 bytes
Option<UserId> takes 4 bytes
UserId::new(0) = None
```

`cargo test` prints `test result: ok. 2 passed`. Why it works: a struct with one field inherits that field's niche, so the compiler stores `None` as 0. The type also documents a rule (ids start at 1) and makes it impossible to break.

</details>

### Exercise 4 — Explain an enum's size (hard)

Without running anything first, predict the sizes of `Message` and `Small`, then check:

```rust
#[allow(dead_code)]
enum Message {
    Quit,
    Move { x: i32, y: i32 },
    Write(String),
}

#[allow(dead_code)]
enum Small {
    Quit,
    Move { x: i32, y: i32 },
}

fn main() {
    println!("Message: {} bytes", size_of::<Message>());
    println!("Small:   {} bytes", size_of::<Small>());
    println!("String:  {} bytes", size_of::<String>());
}
```

<details>
<summary>Hint</summary>

The largest variant of `Message` is `Write(String)`. Does `String` have a niche? (Section 4, `Option<String>`.) `Small` has no field with a niche: `i32` uses all its bit patterns.

</details>

<details>
<summary>Answer</summary>

```text
Message: 24 bytes
Small:   12 bytes
String:  24 bytes
```

Why it works: `Message` fits in 24 bytes, the size of its largest variant, with no extra tag. The capacity inside `String` can never exceed `isize::MAX`, so the compiler stores "this is `Quit`" or "this is `Move`" as impossible capacity values, and the `x` and `y` of `Move` fit in the other bytes. `Small` has no niche: it needs a tag (4 bytes, because the variant data is aligned to 4) plus 8 bytes for `x` and `y`, so 12. These details are chosen by the compiler and may change; only the `Option` cases from section 4 are guaranteed.

</details>

## Quiz

1. Why does a struct have padding?

<details>
<summary>Answer</summary>

So that every field starts at an address that matches its alignment, and so that the size is a multiple of the struct's alignment (then every element of an array of structs is aligned too).

</details>

2. When must you use `#[repr(C)]`?

<details>
<summary>Answer</summary>

Whenever something outside your Rust code depends on the layout: C code via FFI, a binary file or network format, or hardware. The default `repr(Rust)` layout is not guaranteed.

</details>

3. Why is `Option<&T>` 8 bytes but `Option<u64>` 16 bytes?

<details>
<summary>Answer</summary>

A reference can never be null, so the value 0 is a niche, and `None` is stored as 0 without an extra tag. Every bit pattern of a `u64` is a valid number, so `Option<u64>` needs a separate tag, which with padding costs 8 more bytes.

</details>

4. Why is `&[u8]` 16 bytes, and what is stored in the second word?

<details>
<summary>Answer</summary>

It is a fat pointer: the first word is the address of the first element, the second is the length (number of elements). The size of a slice is not known at compile time, so the pointer carries it.

</details>

5. Why does `println!("{}", p.b)` fail to compile when `p` is a `repr(packed)` struct and `b` is a `u32`?

<details>
<summary>Answer</summary>

`println!` takes a reference to its arguments. In a packed struct `b` may be misaligned, and a misaligned reference is UB, so the compiler refuses (E0793). Copy the field into a local variable first.

</details>

## Summary

- Every type has a **size** and an **alignment**; values always start at addresses that are multiples of their alignment, because the CPU handles aligned data best and misaligned references are UB.
- Structs contain **padding** to keep fields aligned. The default `repr(Rust)` may reorder fields to shrink it; nothing about that order is guaranteed.
- `#[repr(C)]` fixes the order (needed for FFI and byte formats); `repr(align(N))` raises alignment; `repr(packed)` removes padding but forbids references to fields; `repr(transparent)` gives a wrapper its field's layout.
- Enums store a **discriminant** next to the largest variant, unless a **niche** lets the compiler hide it: `Option<&T>`, `Option<Box<T>>` and `Option<NonZeroU32>` cost nothing extra.
- Pointers to slices, `str` and `dyn Trait` are **fat** (two words); `Vec` and `String` are three words; **zero-sized types** take no memory at all.
- Check important layouts with `const` assertions and `offset_of!` tests; `cargo check --target` checks them for other platforms.

## Sources and further reading

- [The Rust Reference — Type layout](https://doc.rust-lang.org/reference/type-layout.html) — the official rules for size, alignment and every `repr`.
- [The Rustonomicon — Data representation in Rust](https://doc.rust-lang.org/nomicon/repr-rust.html) and [Alternative representations](https://doc.rust-lang.org/nomicon/other-reprs.html) — the reasons behind the rules, with examples.
- [The Rustonomicon — Exotically sized types](https://doc.rust-lang.org/nomicon/exotic-sizes.html) — dynamically sized and zero-sized types.
- [std::option — Representation](https://doc.rust-lang.org/std/option/index.html) — the list of types for which `Option<T>` is guaranteed to be as small as `T`.
- [std::mem::offset_of!](https://doc.rust-lang.org/std/mem/macro.offset_of.html) and [std::mem::align_of](https://doc.rust-lang.org/std/mem/fn.align_of.html) — the layout tools used in this lesson.
- [std::num::NonZeroU32](https://doc.rust-lang.org/std/num/type.NonZeroU32.html) — the non-zero integer types and their niche.
- [Rust error index — E0793](https://doc.rust-lang.org/error_codes/E0793.html) — the packed-field error explained.

---

[🏠 Start](../START-HERE.md) · [← 12.10 Writing cache-friendly code](10-cache-friendly-code.md) · [📋 Phase 12](README.md) · [12.12 Linking and loading programs →](12-linking-and-loading.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
