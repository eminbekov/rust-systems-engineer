<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 06.5 Lifetimes in depth](05-lifetimes-in-depth.md) · [📋 Phase 06](README.md) · [06.7 Declarative macros →](07-declarative-macros.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/06-rust-advanced/06-unsafe-rust-and-miri.md) <!-- nav -->

# Lesson 06.6 — Unsafe Rust and Miri

> **Phase 06 — Rust advanced** · Lesson 6 of 12 · ⏱ about 4 hours (2 sittings)

## What you will learn

- What `unsafe` means: it is a promise from you to the compiler, not a switch that turns checks off.
- The five things only `unsafe` code can do, and how to use raw pointers.
- How to write unsafe code responsibly: `// SAFETY:` comments, `# Safety` docs, and small safe wrappers.
- What undefined behaviour (UB) is, and why a program with UB can still "work" on your machine.
- How to run your code under Miri, a tool that detects many kinds of UB.
- Edition 2024 rules: `unsafe_op_in_unsafe_fn` and `static_mut_refs`.

## Before you start

- You finished [Lesson 06.5 — Lifetimes in depth](05-lifetimes-in-depth.md).
- You know ownership, borrowing and slices ([Phase 03](../03-ownership-and-borrowing/01-memory-stack-and-heap.md)), `Box` ([Lesson 05.9](../05-rust-intermediate/09-smart-pointers.md)) and tests ([Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md)).
- Create a practice project: `cargo new code/p06-unsafe`.
- To use Miri you need the nightly toolchain with the Miri component. Install it once; your normal (stable) toolchain stays the default:

```sh
rustup toolchain install nightly --component miri
cargo +nightly miri --version
```

The first Miri run prints `Preparing a sysroot for Miri ... done`. That is normal: it builds a special copy of the standard library, which takes a minute.

## New words

| Word | Meaning |
|---|---|
| **`unsafe`** | A keyword that marks code where **you**, not the compiler, guarantee that Rust's rules hold. |
| **raw pointer** | A pointer without the safety rules of references: `*const T` (read) and `*mut T` (read and write). It may be null, dangling or misaligned. |
| **dereference** | Following a pointer to read or write the value it points to (`*pointer`). |
| **UB** (Undefined Behaviour) | A program that breaks the language rules. The compiler assumes this never happens, so the result can be anything: a crash, garbage, or "works fine". |
| **dangling pointer** | A pointer to memory that has been freed or never existed. |
| **aliasing** | Two pointers or references to the same memory. Rust forbids a `&mut` to exist together with any other reference to the same place. |
| **API** (Application Programming Interface) | The functions and types that your code offers to other code. |
| **soundness** | A safe API is **sound** if no safe code can use it to cause UB. An API that can be misused from safe code is **unsound**. |
| **safe abstraction** | A function with a safe signature whose body uses `unsafe` internally, and checks everything needed to be sound. |
| **Miri** | An interpreter for Rust's MIR (Mid-level Intermediate Representation, a simplified form of your program inside the compiler). It runs your program step by step and stops when it sees UB. |
| **Stacked Borrows / Tree Borrows** | Two experimental rule sets that Miri uses to decide which pointer uses are allowed (aliasing rules). |
| **union** | A type whose fields all share the same memory. Reading a field is `unsafe`. |
| **FFI** (Foreign Function Interface) | Calling code written in another language. You learn it in [Lesson 06.10](10-ffi-calling-c.md). |

## 1. What `unsafe` really means

Safe Rust gives you strong guarantees: no use after free, no data races, no out-of-bounds reads. The compiler checks them. But some correct programs cannot be proven correct by the compiler: talking to hardware, calling C code, building a data structure like `Vec` or `Rc`.

For these cases Rust has the keyword `unsafe`. It does **not** switch off the borrow checker or type checking. It only opens five extra abilities (section 2). In exchange you make a **promise**: "I have checked that the rules hold here, even though the compiler cannot".

Think of a bank vault with a safe door. Safe Rust is the door with a lock that the bank checks. `unsafe` is a back door that you open yourself: the bank trusts you to close it correctly. If you do not, the damage can be anywhere.

Two sides use the keyword in two directions:

- `unsafe { ... }` (a block) means: "**I** promise this code is correct". It is used where unsafe operations are performed.
- `unsafe fn` and `unsafe trait` mean: "**you**, the caller or implementor, must follow extra rules that I describe in the docs".

> 🔒 **Security:** Memory-safety bugs (use after free, out-of-bounds access, double free) are a very large share of serious security vulnerabilities in programs written in languages without these checks (for example C, the language you will call from Rust in Lesson 06.10). Safe Rust removes the whole class. Every `unsafe` block is a place where it can come back, so reviewers read them with special care.

## 2. The five unsafe abilities

Inside an `unsafe` block or function you may do exactly these five things that safe code may not:

| # | Ability | Example |
|---|---|---|
| 1 | Dereference a raw pointer | `*pointer` |
| 2 | Call an `unsafe` function or method | `slice.get_unchecked(i)` |
| 3 | Access or modify a mutable `static` variable | `COUNTER += 1` |
| 4 | Implement an `unsafe` trait | `unsafe impl Send for Handle {}` |
| 5 | Access a field of a `union` | `value.int` |

Everything else stays checked: ownership, borrowing, types and bounds checks of normal indexing still work inside `unsafe`.

## 3. Raw pointers

A **raw pointer** is just an address. There are two kinds: `*const T` for reading and `*mut T` for reading and writing. Unlike a reference, a raw pointer:

- may be null or dangling;
- is not tracked by the borrow checker;
- may exist together with other pointers or references to the same place.

**Creating** a raw pointer is safe. **Using** it (dereferencing) is unsafe, because only you know whether the address is valid:

```rust
fn main() {
    let mut number = 10;

    // Creating raw pointers is SAFE. Only using them is unsafe.
    let read_ptr: *const i32 = &number;
    let write_ptr: *mut i32 = &mut number;

    // SAFETY: both pointers come from `number`, which is alive and not borrowed
    // by anything else while we use the pointers.
    unsafe {
        *write_ptr += 5;
        println!("through write_ptr: {}", *write_ptr);
        println!("through read_ptr: {}", *read_ptr);
    }

    let nothing: *const i32 = std::ptr::null();
    println!("is null? {}", nothing.is_null());
}
```

```text
through write_ptr: 15
through read_ptr: 15
is null? true
```

**What happens:**

1. `let read_ptr: *const i32 = &number;` converts a reference to a raw pointer. No `unsafe` is needed.
2. `let write_ptr: *mut i32 = &mut number;` does the same for a mutable one.
3. Inside `unsafe`, `*write_ptr += 5` writes through the pointer, and `*read_ptr` reads. Both pointers point to `number`, so they see 15.
4. The comment starting with `// SAFETY:` explains **why** the block is valid. Write one for every `unsafe` block (section 4).
5. `std::ptr::null()` creates a null pointer. `is_null()` is a safe method that checks for it.

Forgetting the `unsafe` block is an error:

```text
error[E0133]: dereference of raw pointer is unsafe and requires unsafe block
 --> src/main.rs:4:20
  |
4 |     println!("{}", *pointer);
  |                    ^^^^^^^^ dereference of raw pointer
  |
  = note: raw pointers may be null, dangling or unaligned; they can violate aliasing rules and cause data races: all of these are undefined behavior

For more information about this error, try `rustc --explain E0133`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

> 📏 **Convention:** Convert references to raw pointers as late as possible and back as early as possible. Keep the `unsafe` block as small as you can, ideally one or two lines.

## 4. Unsafe functions and `// SAFETY:` comments

A function is marked `unsafe fn` when it has requirements that the compiler cannot check. The caller must read the contract and make sure it is satisfied. By convention the contract is written in a `# Safety` section of the documentation comment, and every call site gets a `// SAFETY:` comment:

```rust
/// Returns the element at `index` without checking the bounds.
///
/// # Safety
///
/// `index` must be less than `values.len()`. Otherwise the behaviour is undefined.
unsafe fn get_unchecked(values: &[i32], index: usize) -> i32 {
    // SAFETY: the caller promised that `index < values.len()`,
    // so the pointer stays inside the slice.
    unsafe { *values.as_ptr().add(index) }
}

fn main() {
    let values = [10, 20, 30];
    let index = 1;

    // SAFETY: `index` is 1 and `values` has 3 elements.
    let value = unsafe { get_unchecked(&values, index) };
    println!("value = {value}");
}
```

```text
value = 20
```

**What happens:**

1. `get_unchecked` reads an element without checking `index`. That is faster than normal indexing, but only valid when `index < values.len()`.
2. The `# Safety` section states that condition. It is the contract the **caller** must satisfy.
3. Inside the function, `values.as_ptr().add(index)` computes the address of the element, and `*` reads it. This needs an `unsafe` block, with its own `// SAFETY:` comment that points to the contract.
4. In `main`, the call is wrapped in `unsafe { ... }` with a `// SAFETY:` comment that explains why the contract holds.

**Edition 2024:** the body of an `unsafe fn` is *not* automatically an unsafe block any more. You still need `unsafe { }` for each unsafe operation inside. The lint `unsafe_op_in_unsafe_fn` warns if you forget:

```text
warning[E0133]: dereference of raw pointer is unsafe and requires unsafe block
 --> src/main.rs:2:5
  |
2 |     *pointer
  |     ^^^^^^^^ dereference of raw pointer
  |
  = note: raw pointers may be null, dangling or unaligned; they can violate aliasing rules and cause data races: all of these are undefined behavior
note: an unsafe function restricts its caller, but its body is safe by default
 --> src/main.rs:1:1
  |
1 | unsafe fn read(pointer: *const i32) -> i32 {
  | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  = note: for more information, see <https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-op-in-unsafe-fn.html>
  = note: `#[warn(unsafe_op_in_unsafe_fn)]` (part of `#[warn(rust_2024_compatibility)]`) on by default

For more information about this error, try `rustc --explain E0133`.
```

The reason: a long `unsafe fn` would otherwise hide which lines are truly dangerous.

> 📏 **Convention:** Every `unsafe` block has a `// SAFETY:` comment. Every `unsafe fn` has a `# Safety` doc section. Clippy can enforce this:

```rust
#![warn(clippy::undocumented_unsafe_blocks)]

fn main() {
    let numbers = [1, 2, 3];
    let value = unsafe { *numbers.as_ptr().add(1) };
    println!("{value}");
}
```

```text
warning: unsafe block missing a safety comment
 --> src/main.rs:5:17
  |
5 |     let value = unsafe { *numbers.as_ptr().add(1) };
  |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = help: consider adding a safety comment on the preceding line
  = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.100.0/index.html#undocumented_unsafe_blocks
note: the lint level is defined here
 --> src/main.rs:1:9
  |
1 | #![warn(clippy::undocumented_unsafe_blocks)]
  |         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

You can turn the lint on for a whole crate with `#![warn(clippy::undocumented_unsafe_blocks)]` at the top of `main.rs` or `lib.rs`.

## 5. Safe abstractions: building `split_at_mut`

The most important skill is to put `unsafe` **inside** a function and offer a **safe** function outside. The caller never has to think about it. A classic case is splitting one slice into two mutable halves. Safe Rust cannot do it by hand:

```rust
fn split_at_mut(values: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    (&mut values[..mid], &mut values[mid..])
}

fn main() {
    let mut numbers = [1, 2, 3, 4, 5];
    let (left, right) = split_at_mut(&mut numbers, 2);
    left[0] = 100;
    right[0] = 200;
    println!("{numbers:?}");
}
```

```text
error[E0499]: cannot borrow `*values` as mutable more than once at a time
 --> src/main.rs:2:31
  |
1 | fn split_at_mut(values: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
  |                         - let's call the lifetime of this reference `'1`
2 |     (&mut values[..mid], &mut values[mid..])
  |     --------------------------^^^^^^--------
  |     |     |                   |
  |     |     |                   second mutable borrow occurs here
  |     |     first mutable borrow occurs here
  |     returning this value requires that `*values` is borrowed for `'1`
  |
  = help: use `.split_at_mut(position)` to obtain two mutable non-overlapping sub-slices

For more information about this error, try `rustc --explain E0499`.
error: could not compile `demo` (bin "demo") due to 1 previous error
```

The compiler does not know that `..mid` and `mid..` do not overlap, so it sees two `&mut` borrows of `*values`. The standard library solves this with `unsafe` inside `slice::split_at_mut`. Here is our own version:

```rust
use std::slice;

/// Splits a mutable slice into two non-overlapping mutable slices.
///
/// # Panics
///
/// Panics if `mid > values.len()`.
fn split_at_mut(values: &mut [i32], mid: usize) -> (&mut [i32], &mut [i32]) {
    let len = values.len();
    let pointer = values.as_mut_ptr();

    assert!(mid <= len, "mid must not be greater than the length");

    // SAFETY: `pointer` is valid for `len` elements because it comes from a live
    // slice. The two ranges `0..mid` and `mid..len` do not overlap and both stay
    // inside the slice (checked by the assert above). `values` is borrowed
    // mutably for the whole lifetime of the result, so nobody else can touch it.
    unsafe {
        (
            slice::from_raw_parts_mut(pointer, mid),
            slice::from_raw_parts_mut(pointer.add(mid), len - mid),
        )
    }
}

fn main() {
    let mut numbers = [1, 2, 3, 4, 5];
    let (left, right) = split_at_mut(&mut numbers, 2);
    left[0] = 100;
    right[0] = 200;
    println!("{numbers:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn splits_in_the_middle() {
        let mut data = [1, 2, 3, 4];
        let (a, b) = split_at_mut(&mut data, 2);
        assert_eq!(a, &mut [1, 2]);
        assert_eq!(b, &mut [3, 4]);
    }

    #[test]
    fn edges_give_an_empty_part() {
        let mut data = [1, 2, 3];
        let (a, b) = split_at_mut(&mut data, 0);
        assert!(a.is_empty());
        assert_eq!(b.len(), 3);
        let (a, b) = split_at_mut(&mut data, 3);
        assert_eq!(a.len(), 3);
        assert!(b.is_empty());
    }

    #[test]
    #[should_panic(expected = "mid must not be greater")]
    fn panics_when_mid_is_too_big() {
        let mut data = [1, 2, 3];
        let _ = split_at_mut(&mut data, 4);
    }
}
```

```text
[100, 2, 200, 4, 5]
```

**What happens:**

1. `values.as_mut_ptr()` gives a `*mut i32` to the first element.
2. `assert!(mid <= len, ...)` checks the one condition that makes the `unsafe` block valid. If it fails, the function panics. This is the key: the **safe function checks everything** that the unsafe operations need.
3. `slice::from_raw_parts_mut(pointer, mid)` builds a `&mut [i32]` from an address and a length. It is `unsafe`, because the compiler cannot know that the memory is valid.
4. `pointer.add(mid)` moves the address forward by `mid` elements (not bytes). The second slice starts there and has `len - mid` elements.
5. The `// SAFETY:` comment lists all reasons: the pointer is valid, the two ranges do not overlap, they stay inside, and the original slice is borrowed mutably, so nobody else can reach it.
6. The signature `(&mut [i32], &mut [i32])` is safe to call. The lifetime elision rule ([Lesson 06.5](05-lifetimes-in-depth.md)) ties both results to `values`, so they cannot outlive it.

> 🧪 **Testing:** The tests cover the middle, both edges (an empty half) and the panic with `#[should_panic]`. Run them under Miri too (section 9): `cargo +nightly miri test`.

## 6. Mutable statics

A `static mut` is a global variable that anyone can change. That is dangerous with threads, so reading or writing it is unsafe (ability 3). Edition 2024 goes further: it is an error to take a **reference** to a `static mut`, because the reference can easily be misused:

```rust
static mut COUNTER: u32 = 0;

fn main() {
    unsafe {
        COUNTER += 1;
        println!("{}", COUNTER);
    }
}
```

```text
error: creating a shared reference to mutable static
 --> src/main.rs:6:24
  |
6 |         println!("{}", COUNTER);
  |                        ^^^^^^^ shared reference to mutable static
  |
  = note: shared references to mutable statics are dangerous; it's undefined behavior if the static is mutated or if a mutable reference is created for it while the shared reference lives
  = help: use a type that relies on "interior mutability" instead; to read more on this, visit <https://doc.rust-lang.org/reference/interior-mutability.html>
  = note: for more information, see <https://doc.rust-lang.org/edition-guide/rust-2024/static-mut-references.html>
  = note: `#[deny(static_mut_refs)]` (part of `#[deny(rust_2024_compatibility)]`) on by default

error: could not compile `demo` (bin "demo") due to 1 previous error
```

The macro `println!` takes a reference to its arguments, which is exactly what the lint forbids. If you must use `static mut`, copy the value out first:

```rust
static mut COUNTER: u32 = 0;

fn main() {
    // SAFETY: this program has only one thread, so nobody else can access COUNTER.
    unsafe {
        COUNTER += 1;
        let now = COUNTER; // a COPY of the value, not a reference to the static
        println!("{now}");
    }
}
```

```text
1
```

**What happens:** `COUNTER += 1` writes the static directly (no reference is created). `let now = COUNTER;` copies the number into a local variable. `println!` then borrows `now`, not the static.

In real code, you almost never need `static mut`. Use an atomic type. It is a number built so that many threads can change it safely, without `unsafe`:

```rust
use std::sync::atomic::{AtomicU32, Ordering};
use std::thread;

// No `unsafe` and no `mut`: the atomic type is built to be shared between threads.
static COUNTER: AtomicU32 = AtomicU32::new(0);

fn main() {
    let handles: Vec<_> = (0..4)
        .map(|_| {
            thread::spawn(|| {
                for _ in 0..1000 {
                    COUNTER.fetch_add(1, Ordering::SeqCst);
                }
            })
        })
        .collect();
    for handle in handles {
        handle.join().unwrap();
    }
    println!("total = {}", COUNTER.load(Ordering::SeqCst));
}
```

```text
total = 4000
```

**What happens:** four threads each add 1 a thousand times. `fetch_add` is one indivisible step, so no update is lost, and the total is 4000. `Ordering::SeqCst` is the strictest and safest choice for now; atomics and memory ordering get a full lesson in Phase 14.

## 7. Unsafe traits: `Send` and `Sync`

A trait is `unsafe` when implementing it makes a promise that the compiler cannot check. The two most important are the marker traits `Send` ("the value can move to another thread") and `Sync` ("`&T` can be shared between threads"). The compiler implements them automatically when all fields have them. A raw pointer has neither, so any struct containing one loses them:

```text
error[E0277]: `*mut i32` cannot be sent between threads safely
  --> src/main.rs:8:32
   |
 8 |       let worker = thread::spawn(move || {
   |                    ------------- ^------
   |                    |             |
   |  __________________|_____________within this `{closure@src/main.rs:8:32: 8:39}`
   | |                  |
   | |                  required by a bound introduced by this call
 9 | |         let handle = handle;
10 | |         handle.0 as usize
11 | |     });
   | |_____^ `*mut i32` cannot be sent between threads safely
   |
   = help: within `{closure@src/main.rs:8:32: 8:39}`, the trait `Send` is not implemented for `*mut i32`
note: required because it appears within the type `Handle`
  --> src/main.rs:3:15
   |
 3 | struct Handle(*mut i32);
   |               ^^^^^^^^ required by this field
note: required because `handle` is used within this closure
  --> src/main.rs:9:22
   |
 9 |         let handle = handle;
   |                      ^^^^^^
note: required by a bound in `spawn`
  --> /rustc/0ef1bc386059596dada8fb65b50425aab2ea637a/library/std/src/thread/functions.rs:125:0
```

If you know that your type is safe to send, you promise it with `unsafe impl`:

```rust
use std::thread;

struct Handle(*mut i32);

// SAFETY: `Handle` is only ever used to access a leaked allocation that no other
// code touches, and only one thread at a time owns the handle.
unsafe impl Send for Handle {}

fn main() {
    let raw: *mut i32 = Box::into_raw(Box::new(41));
    let handle = Handle(raw);

    let worker = thread::spawn(move || {
        let handle = handle; // move the whole struct, not only its field
        // SAFETY: `handle.0` is a valid, unique pointer from Box::into_raw.
        unsafe {
            *handle.0 += 1;
            *handle.0
        }
    });
    let result = worker.join().unwrap();
    println!("worker returned {result}");

    // SAFETY: the worker is finished, and `raw` came from Box::into_raw.
    drop(unsafe { Box::from_raw(raw) });
}
```

```text
worker returned 42
```

**What happens:**

1. `unsafe impl Send for Handle {}` tells the compiler: "trust me". The `// SAFETY:` comment explains why: here, only one thread uses the pointer at any time.
2. `thread::spawn(move || ...)` now accepts the closure. Inside, `let handle = handle;` moves the whole struct into the closure (without it, the closure would capture only the field `handle.0`, a bare `*mut i32`, which is not `Send`).
3. The worker increments the number to 42 and returns it.
4. After `join`, `Box::from_raw(raw)` turns the pointer back into a `Box` and `drop` frees it. Exactly once.

A wrong `unsafe impl Send` is a classic source of data races, so do it rarely and always explain it.

## 8. Unions

A **union** has several fields that share the same bytes. Rust cannot know which field was written last, so reading a field is unsafe:

```rust
#[repr(C)]
union IntOrFloat {
    int: u32,
    float: f32,
}

fn main() {
    let value = IntOrFloat { float: 1.0 };
    // SAFETY: every bit pattern is a valid u32, so reading the bytes of an f32
    // as a u32 is fine.
    let bits = unsafe { value.int };
    println!("1.0f32 has the bits {bits:#010x}");

    // The safe way to get the same answer:
    println!("{:#010x}", 1.0f32.to_bits());
}
```

```text
1.0f32 has the bits 0x3f800000
0x3f800000
```

**What happens:** the float `1.0` is stored as the 32-bit pattern `0x3f800000` (this is the IEEE 754 format, named after the Institute of Electrical and Electronics Engineers that published the standard; it is the usual way computers store numbers with a fractional part, and you will study it in Phase 12). Reading the same bytes as `u32` shows that pattern. `#[repr(C)]` fixes the layout like C does. The safe method `f32::to_bits` gives the same answer, so prefer it. You mostly meet unions when talking to C code.

## 9. Undefined behaviour and Miri

**Undefined behaviour (UB)** means the program broke a rule of the language. The compiler is allowed to assume that UB never happens, and it optimizes with this assumption. So when UB happens, *anything* can follow: a crash, a wrong value, a security hole, or a perfectly normal result today that breaks after you upgrade the compiler.

The main kinds of UB in Rust:

- Using a dangling or null pointer, or a misaligned one.
- Reading outside of an allocation (out of bounds).
- Reading uninitialized memory as a value.
- Creating an invalid value: a `bool` that is not 0 or 1, a reference that is null, a `char` that is not a valid character.
- Breaking the aliasing rules: using a `&mut` while another pointer has been used to access the same place.
- A data race (two threads, one write, no synchronization).

The scary part: UB often *looks fine*. These programs have UB, and here is what they do when run normally:

```rust
fn main() {
    let numbers = [1, 2, 3];
    let pointer = numbers.as_ptr();

    // BUG: index 3 is one past the end of the array.
    let value = unsafe { *pointer.add(3) };
    println!("value = {value}");
}
```

```text
value = 0
```

It prints some number and exits successfully, although it read memory that does not belong to the array. The next program reads memory that was never written:

```rust
use std::mem::MaybeUninit;

fn main() {
    let slot: MaybeUninit<u32> = MaybeUninit::uninit();

    // BUG: the memory was never written, but we claim it holds a valid u32.
    let number = unsafe { slot.assume_init() };
    println!("number = {number}");
}
```

```text
number = 36850252
```

It prints a random-looking number; on your machine it will be another one. And a program that reads a string after freeing it (`ub_use_after_free.rs`, shown below) crashes on this Mac:

```text
Segmentation fault (exit code 139)
```

On another machine or with another compiler, the same program might print `hello`. **"My tests pass" says nothing about UB.** You need a tool that checks the rules.

That tool is **Miri**. It is an interpreter: instead of running machine code, it executes your program step by step on a model of Rust's memory, and it keeps track of every allocation, every pointer and every borrow. When something breaks a rule, it stops and tells you where. It is slow (many times slower than normal), it needs the nightly toolchain, and it can only check the paths that your program actually executes, but it finds bugs that no normal test can.

Use it on the same programs:

```sh
cargo +nightly miri run     # run main under Miri
cargo +nightly miri test    # run all tests under Miri
```

**Out-of-bounds read:**

```text
error: Undefined Behavior: memory access failed: attempting to access 4 bytes, but got alloc196+0xc which is at or beyond the end of the allocation of size 12 bytes
 --> src/main.rs:6:26
  |
6 |     let value = unsafe { *pointer.add(3) };
  |                          ^^^^^^^^^^^^^^^ Undefined Behavior occurred here
  |
  = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
  = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
help: alloc196 was allocated here:
 --> src/main.rs:2:9
  |
2 |     let numbers = [1, 2, 3];
  |         ^^^^^^^
```

**Use after free:**

```rust
fn main() {
    let boxed = Box::new(String::from("hello"));
    let raw: *mut String = Box::into_raw(boxed);

    // SAFETY: `raw` came from Box::into_raw, so this gives the Box back.
    drop(unsafe { Box::from_raw(raw) }); // the String is freed here

    // BUG: `raw` now dangles. Reading through it is undefined behaviour.
    let text = unsafe { &*raw };
    println!("{text}");
}
```

```text
error: Undefined Behavior: constructing invalid value of type &std::string::String: encountered a dangling reference (use-after-free)
 --> src/main.rs:9:25
  |
9 |     let text = unsafe { &*raw };
  |                         ^^^^^ Undefined Behavior occurred here
  |
  = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
  = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
```

**Reading uninitialized memory:**

```text
error: Undefined Behavior: constructing invalid value of type u32: encountered uninitialized memory, but expected an integer
 --> src/main.rs:7:27
  |
7 |     let number = unsafe { slot.assume_init() };
  |                           ^^^^^^^^^^^^^^^^^^ Undefined Behavior occurred here
  |
  = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
  = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
```

**An invalid `bool`:**

```rust
fn main() {
    let byte: u8 = 3;

    // BUG: a bool must be 0 or 1. Any other byte is not a valid bool.
    let flag: bool = unsafe { std::mem::transmute::<u8, bool>(byte) };
    if flag {
        println!("true");
    } else {
        println!("false");
    }
}
```

```text
error: Undefined Behavior: constructing invalid value of type bool: encountered 0x03, but expected a boolean
 --> src/main.rs:5:31
  |
5 |     let flag: bool = unsafe { std::mem::transmute::<u8, bool>(byte) };
  |                               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Undefined Behavior occurred here
  |
  = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
  = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
```

**Breaking the aliasing rules:**

```rust
fn main() {
    let mut value = 5;
    let raw: *mut i32 = &mut value;

    // SAFETY (wrong!): two `&mut` to the same place exist at the same time.
    let first: &mut i32 = unsafe { &mut *raw };
    let second: &mut i32 = unsafe { &mut *raw };

    *second += 1;
    *first += 10; // BUG: `first` was invalidated when `second` was created
    println!("value = {value}");
}
```

```text
error: Undefined Behavior: attempting a read access using <554> at alloc196[0x0], but that tag does not exist in the borrow stack for this location
  --> src/main.rs:10:5
   |
10 |     *first += 10; // BUG: `first` was invalidated when `second` was created
   |     ^^^^^^^^^^^^ this error occurs as part of an access at alloc196[0x0..0x4]
   |
   = help: this indicates a potential bug in the program: it performed an invalid operation, but the Stacked Borrows rules it violated are still experimental
   = help: see https://github.com/rust-lang/unsafe-code-guidelines/blob/master/wip/stacked-borrows.md for further information
help: <554> was created by a Unique retag at offsets [0x0..0x4]
  --> src/main.rs:6:36
   |
 6 |     let first: &mut i32 = unsafe { &mut *raw };
   |                                    ^^^^^^^^^
help: <554> was later invalidated at offsets [0x0..0x4] by a Unique retag
  --> src/main.rs:7:37
   |
 7 |     let second: &mut i32 = unsafe { &mut *raw };
   |                                     ^^^^^^^^^
```

**What happens (aliasing):**

1. `raw` is a raw pointer to `value`. Two lines create two `&mut i32` from it. Safe Rust would never allow two live `&mut` to the same place.
2. Creating `second` invalidates `first` in Miri's model. Miri calls each pointer a "tag" (here `<554>`; the number differs on your machine).
3. Using `first` afterwards is the violation. Miri says which line created the tag, and which line invalidated it. That is the exact bug.
4. The rules Miri applies are called **Stacked Borrows**. A newer model, **Tree Borrows**, is more permissive in some cases. Try it with `MIRIFLAGS="-Zmiri-tree-borrows" cargo +nightly miri run`; it also reports this program as UB.
5. A correct version uses one `&mut` at a time, and Miri stays silent:

```rust
fn main() {
    let mut value = 5;

    // Safe Rust says it clearly: use one &mut at a time.
    let first = &mut value;
    *first += 10;
    let second = &mut value;
    *second += 1;
    println!("value = {value}");
}
```

```text
value = 16
```

> 💡 **Tip:** Run `cargo +nightly miri test` before you publish any crate with `unsafe`. Add it to your Continuous Integration (CI, a server that runs your checks on every change) in Phase 09. The Rust standard library itself is tested with Miri.

Things Miri cannot do: it cannot check code paths that never run, it cannot run most FFI calls into C libraries, and it does not find logic bugs. It does not replace a careful review.

## 10. Rules of thumb for writing `unsafe`

1. **Try safe Rust first.** `unsafe` for speed is almost never needed; measure first ([Lesson 05.12](../05-rust-intermediate/12-benchmarking-basics.md)).
2. **Keep it small.** Put the unsafe operations in a few lines inside one function or module.
3. **Make the outside safe.** A public function that can cause UB from safe code is *unsound* (Exercise 4). Check the conditions inside, or mark the function `unsafe` and document the contract.
4. **Document.** `// SAFETY:` on every block, `# Safety` on every `unsafe fn`, and a note in the type docs about the invariants (facts that must always be true, such as "`len` never exceeds the capacity").
5. **Test, and run Miri.**
6. **Forbid it where not needed.** In a crate that must contain no unsafe code, write `#![forbid(unsafe_code)]` at the top: the compiler then rejects any `unsafe` in that crate. The tool `cargo-geiger` counts `unsafe` in your dependencies (Phase 10 covers supply-chain checks).

## Common mistakes

### 1. `unsafe` as a way to silence the compiler

If the compiler rejects your code, the cause is nearly always a real problem. Do not add `unsafe` to "make it work". Find the safe fix first (a `RefCell`, a clone, a different structure).

### 2. A safe function that is unsound

```rust
pub fn get(values: &[i32], index: usize) -> i32 {
    unsafe { *values.as_ptr().add(index) }
}
```

Anyone can call `get(&[1], 5)` from safe code. Check the index, or make the function `unsafe fn`.

### 3. Missing `// SAFETY:` comment

If you cannot write down *why* the block is valid, it probably is not.

### 4. References to `static mut`

`error: creating a shared reference to mutable static` (edition 2024). Use an atomic, a `Mutex`, or copy the value out.

### 5. Believing "it ran, so it is fine"

UB can look fine. Run `cargo +nightly miri test`.

### 6. Two `&mut` from one raw pointer

Make the reference, use it, finish with it; only then create the next one.

## Try it yourself

1. Open `~/projects/rust/code/p06-unsafe`.
2. Copy `split_at_mut.rs` (section 5) into `src/main.rs`. Run `cargo test`.
3. Install Miri (the commands in "Before you start") and run `cargo +nightly miri test`. All three tests must pass.
4. Break the code on purpose: change `slice::from_raw_parts_mut(pointer.add(mid), len - mid)` to `slice::from_raw_parts_mut(pointer.add(mid), len - mid + 1)` (one element too long). Run `cargo test` first: it may still pass. Then run `cargo +nightly miri test` and read the error. Restore the code.
5. Add `#![warn(clippy::undocumented_unsafe_blocks)]` at the top. Remove one `// SAFETY:` comment and run `cargo clippy` to see the warning. Restore it.
6. Commit: `git add code/p06-unsafe` and `git commit -m "feat(p06): split_at_mut with unsafe, checked with Miri"`.

## Exercises

### Exercise 1 — Double a number through a raw pointer (easy)

Write `fn double_in_place(value: &mut i32)` that turns `value` into a `*mut i32` and doubles the number through the pointer. Add a `// SAFETY:` comment and a test.

<details>
<summary>Hint</summary>

`let pointer: *mut i32 = value;` converts a `&mut i32` automatically. Then `unsafe { *pointer *= 2; }`.

</details>

<details>
<summary>Answer</summary>

```rust
fn double_in_place(value: &mut i32) {
    let pointer: *mut i32 = value;
    // SAFETY: `pointer` was made from a valid `&mut i32`, which is alive and
    // exclusive for the whole call.
    unsafe {
        *pointer *= 2;
    }
}

fn main() {
    let mut number = 21;
    double_in_place(&mut number);
    println!("{number}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn doubles_the_value() {
        let mut x = 4;
        double_in_place(&mut x);
        assert_eq!(x, 8);
    }
}
```

```text
42
```

```text

running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the pointer comes from a valid exclusive reference that lives for the whole call, so writing through it is valid. (In real code you would simply write `*value *= 2`.)

</details>

### Exercise 2 — Swap two elements with raw pointers (medium)

Write `fn swap_two(values: &mut [i32], i: usize, j: usize)` using `std::ptr::swap`. It must panic with the message `index out of bounds` if an index is too large. Test: a normal swap, `i == j`, and the panic. Run the tests under Miri.

<details>
<summary>Hint</summary>

Check both indexes with `assert!`. Then `let pointer = values.as_mut_ptr();` and `std::ptr::swap(pointer.add(i), pointer.add(j))` inside `unsafe`.

</details>

<details>
<summary>Answer</summary>

```rust
fn swap_two(values: &mut [i32], i: usize, j: usize) {
    assert!(i < values.len() && j < values.len(), "index out of bounds");
    let pointer = values.as_mut_ptr();
    // SAFETY: both indexes are below `values.len()` (checked above), so both
    // pointers stay inside the slice. `ptr::swap` accepts equal pointers.
    unsafe {
        std::ptr::swap(pointer.add(i), pointer.add(j));
    }
}

fn main() {
    let mut data = [1, 2, 3, 4];
    swap_two(&mut data, 0, 3);
    println!("{data:?}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn swaps_two_elements() {
        let mut data = [1, 2, 3, 4];
        swap_two(&mut data, 0, 3);
        assert_eq!(data, [4, 2, 3, 1]);
    }

    #[test]
    fn same_index_changes_nothing() {
        let mut data = [1, 2, 3];
        swap_two(&mut data, 1, 1);
        assert_eq!(data, [1, 2, 3]);
    }

    #[test]
    #[should_panic(expected = "index out of bounds")]
    fn rejects_bad_index() {
        let mut data = [1, 2, 3];
        swap_two(&mut data, 0, 3);
    }
}
```

```text
[4, 2, 3, 1]
```

```text

running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```text

running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.40s
```

Why it works: the `assert!` guarantees both pointers stay inside the slice, which is the condition for `ptr::swap`. (The standard library has the safe `slice::swap`; this is practice.)

</details>

### Exercise 3 — Find the double free with Miri (medium)

This program frees the same allocation twice. Run it under Miri, read the message, and fix it so that Miri reports nothing.

```rust
fn main() {
    let raw: *mut String = Box::into_raw(Box::new(String::from("data")));
    unsafe {
        drop(Box::from_raw(raw));
        drop(Box::from_raw(raw));
    }
    println!("done");
}
```

<details>
<summary>Hint</summary>

`Box::into_raw` gives up ownership; `Box::from_raw` takes it back. After the first `from_raw` the allocation is already freed, so the second one is a use after free.

</details>

<details>
<summary>Answer</summary>

Miri's first message line (the lines after it show a stack trace into the standard library and `src/main.rs`):

```text
error: Undefined Behavior: constructing invalid value of type std::boxed::Box<std::string::String>: encountered a dangling box (use-after-free)
```

The fixed program:

```rust
fn main() {
    let raw: *mut String = Box::into_raw(Box::new(String::from("data")));
    // SAFETY: `raw` came from Box::into_raw and is turned back into a Box exactly once.
    drop(unsafe { Box::from_raw(raw) });
    println!("done");
}
```

```text
done
```

Why it works: `Box::from_raw` must be called exactly once for each `Box::into_raw`. Miri tracks the allocation, so it notices the second call on freed memory.

</details>

### Exercise 4 — Make an unsound function sound (hard)

This safe function is unsound. Show with Miri that it can cause UB, then write a sound `get` that returns `Option<i32>`, with a test for in-range, out-of-range and empty input.

```rust
pub fn get_unsound(values: &[i32], index: usize) -> i32 {
    unsafe { *values.as_ptr().add(index) }
}
```

<details>
<summary>Hint</summary>

Compare `index` with `values.len()` before the `unsafe` block. Return `None` when it is not smaller. Call `get_unsound(&numbers, 3)` on a three-element array in a small `main` and run it with `cargo +nightly miri run`.

</details>

<details>
<summary>Answer</summary>

Running the unsound function with a bad index under Miri:

```rust
/// UNSOUND: safe code can call this with a bad index and cause undefined behaviour.
pub fn get_unsound(values: &[i32], index: usize) -> i32 {
    unsafe { *values.as_ptr().add(index) }
}

fn main() {
    let numbers = [10, 20, 30];
    println!("{}", get_unsound(&numbers, 3));
}
```

```text
error: Undefined Behavior: memory access failed: attempting to access 4 bytes, but got alloc196+0xc which is at or beyond the end of the allocation of size 12 bytes
 --> src/main.rs:3:14
  |
3 |     unsafe { *values.as_ptr().add(index) }
  |              ^^^^^^^^^^^^^^^^^^^^^^^^^^^ Undefined Behavior occurred here
  |
  = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
  = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
help: alloc196 was allocated here:
 --> src/main.rs:7:9
  |
7 |     let numbers = [10, 20, 30];
  |         ^^^^^^^
  = note: stack backtrace:
          0: get_unsound
              at src/main.rs:3:14: 3:41
          1: main
              at src/main.rs:8:20: 8:44
```

The sound version:

```rust
/// UNSOUND: safe code can call this with a bad index and cause undefined behaviour.
#[allow(dead_code)]
pub fn get_unsound(values: &[i32], index: usize) -> i32 {
    unsafe { *values.as_ptr().add(index) }
}

/// Sound: the bounds are checked, so no caller can cause undefined behaviour.
pub fn get(values: &[i32], index: usize) -> Option<i32> {
    if index < values.len() {
        // SAFETY: `index < values.len()` was just checked.
        Some(unsafe { *values.as_ptr().add(index) })
    } else {
        None
    }
}

fn main() {
    let numbers = [10, 20, 30];
    println!("{:?} {:?}", get(&numbers, 2), get(&numbers, 3));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn in_range_and_out_of_range() {
        let data = [7, 8, 9];
        assert_eq!(get(&data, 0), Some(7));
        assert_eq!(get(&data, 2), Some(9));
        assert_eq!(get(&data, 3), None);
        assert_eq!(get(&[], 0), None);
    }
}
```

```text
Some(30) None
```

```text

running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the bounds check turns the contract ("index must be valid") into a condition that the function itself verifies. Now no caller can cause UB, which is what *sound* means. (Normal code uses `values.get(index).copied()`; the point here is the reasoning.)

</details>

## Quiz

1. What does `unsafe` turn off in Rust?

<details>
<summary>Answer</summary>

Nothing is turned off. It only allows five extra operations (dereferencing raw pointers, calling unsafe functions, using mutable statics, implementing unsafe traits, accessing union fields). Borrow checking and type checking still apply.

</details>

2. Is creating a raw pointer safe? Is dereferencing it?

<details>
<summary>Answer</summary>

Creating is safe. Dereferencing is unsafe, because only the programmer knows whether the address is valid, aligned and not aliased.

</details>

3. What goes into a `// SAFETY:` comment and a `# Safety` doc section?

<details>
<summary>Answer</summary>

`// SAFETY:` explains why the rules hold at this `unsafe` block. `# Safety` in the docs of an `unsafe fn` states what the caller must guarantee.

</details>

4. What is undefined behaviour, and why can it look harmless?

<details>
<summary>Answer</summary>

A program that breaks the language rules. The compiler assumes it never happens, so the outcome is unpredictable: it may crash, give wrong values, or appear to work until a different compiler, optimization level or machine changes the result.

</details>

5. What does a sound safe API guarantee?

<details>
<summary>Answer</summary>

No safe code can use it to cause UB, whatever arguments are passed. If misuse from safe code is possible, the API is unsound and the function should check its inputs or be marked `unsafe`.

</details>

6. What does Miri do, and what are its limits?

<details>
<summary>Answer</summary>

It interprets the program step by step and reports UB such as dangling pointers, out-of-bounds access, uninitialized reads, invalid values and aliasing violations. It is slow, needs nightly, checks only the paths that run, and cannot do most FFI calls.

</details>

7. Why can you not write `println!("{}", COUNTER)` for a `static mut` in edition 2024?

<details>
<summary>Answer</summary>

`println!` takes a reference to the static, and the `static_mut_refs` lint (deny by default in edition 2024) forbids references to mutable statics. Copy the value to a local first, or use an atomic or `Mutex`.

</details>

8. What changed for `unsafe fn` bodies in edition 2024?

<details>
<summary>Answer</summary>

The body is no longer an implicit unsafe block. Unsafe operations inside still need their own `unsafe { }` (lint `unsafe_op_in_unsafe_fn` warns), so the dangerous lines stay visible.

</details>

## Summary

- `unsafe` is a promise, not a switch. It allows five operations: raw pointer dereference, calling unsafe functions, mutable statics, unsafe traits, union fields.
- Creating raw pointers is safe; using them is not. Keep `unsafe` blocks tiny and write a `// SAFETY:` comment for each; document `unsafe fn` with `# Safety`.
- Build safe abstractions: do the checks inside, expose a safe signature, like `split_at_mut`.
- UB gives no guarantees and often looks fine. Passing tests does not prove its absence.
- Miri (`cargo +nightly miri test`) finds many kinds of UB: dangling pointers, out-of-bounds, uninitialized reads, invalid values, aliasing violations.
- Edition 2024: `unsafe_op_in_unsafe_fn` warns, and references to `static mut` are denied. Prefer atomics, `Mutex`, or `LazyLock`.
- A safe API that can cause UB is unsound. Check inputs or mark the function `unsafe`.

## Sources and further reading

- [The Rust Programming Language — 20.1 Unsafe Rust](https://doc.rust-lang.org/book/ch20-01-unsafe-rust.html) — the five abilities, raw pointers, safe abstractions and `extern`.
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) — the complete guide to writing unsafe Rust correctly; start with "Meet Safe and Unsafe".
- [The Rust Reference — Behavior considered undefined](https://doc.rust-lang.org/reference/behavior-considered-undefined.html) — the official list of UB.
- [Miri on GitHub](https://github.com/rust-lang/miri) — installation, flags (`-Zmiri-tree-borrows`) and limitations.
- [Edition Guide — unsafe_op_in_unsafe_fn](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-op-in-unsafe-fn.html) — the edition 2024 change for unsafe functions.
- [Edition Guide — static mut references](https://doc.rust-lang.org/edition-guide/rust-2024/static-mut-references.html) — why references to `static mut` are denied.
- [Clippy lints](https://rust-lang.github.io/rust-clippy/master/index.html) — look for `undocumented_unsafe_blocks` and other `unsafe` lints.

---

[🏠 Start](../START-HERE.md) · [← 06.5 Lifetimes in depth](05-lifetimes-in-depth.md) · [📋 Phase 06](README.md) · [06.7 Declarative macros →](07-declarative-macros.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
