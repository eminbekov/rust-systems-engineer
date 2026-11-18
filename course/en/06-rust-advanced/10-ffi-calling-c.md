<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 06.9 Introduction to async/await](09-async-await-introduction.md) · [📋 Phase 06](README.md) · [06.11 Project: multithreaded web server →](11-project-multithreaded-web-server.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/06-rust-advanced/10-ffi-calling-c.md) <!-- nav -->

# Lesson 06.10 — FFI: calling C code

> **Phase 06 — Rust advanced** · Lesson 10 of 12 · ⏱ about 4 hours (2 sittings)

## What you will learn

- What FFI and the C ABI are, and why systems programmers need them.
- How to declare and call C functions from Rust (`unsafe extern "C"`), with the right types and strings (`CString`, `CStr`).
- How to compile your own C code with a `build.rs` script and the `cc` crate.
- How to wrap unsafe C calls in a small safe Rust API (checks, `Result`, `Drop`).
- How to call Rust from C, and what `bindgen` does.
- The rules that keep FFI code from causing undefined behaviour.

## Before you start

- You finished [Lesson 06.9 — Introduction to async/await](09-async-await-introduction.md).
- You know `unsafe`, raw pointers and `// SAFETY:` comments from [Lesson 06.6 — Unsafe Rust and Miri](06-unsafe-rust-and-miri.md). This lesson uses them in every section.
- A C compiler is installed. On your Mac it came with the Xcode command line tools in Phase 00. Check it:

  ```sh
  clang --version
  ```

- Create a project: `cd ~/projects/rust && cargo new --lib code/p06-ffi --name geo`.

## New words

| Word | Meaning |
|---|---|
| **FFI** (Foreign Function Interface) | The way one programming language calls functions written in another language. |
| **ABI** (Application Binary Interface) | The low-level rules for calling a function in compiled code: where the arguments are placed, where the result comes back, how types are laid out in memory. Two pieces of compiled code can work together only if they follow the same ABI. |
| **C ABI** | The ABI of the C language. It is the common language between almost all programming languages, so every language can call C and be called from C. |
| **libc** | The standard C library: functions such as `strlen`, `abs`, `malloc`, `free`. Your Rust programs already link it. |
| **header file** (`.h`) | A C file that lists the functions and types a library offers, without their code. |
| **NUL byte** | The byte with value `0`. C strings end with it, so C knows where the text stops. |
| **raw pointer** | A pointer with no safety rules: `*const T` (read) or `*mut T` (read and write). Can be null or dangling. You met them in Lesson 06.6. |
| **null pointer** | A pointer that points nowhere (address 0). C functions use it to say "nothing". |
| **dangling pointer** | A pointer to memory that has already been freed or that no longer holds the value. |
| **`#[repr(C)]`** | An attribute that makes a Rust struct use the same memory layout as the same struct in C. |
| **API** (Application Programming Interface) | What a library offers to its users: its functions and types. A "safe API" is one that cannot be used in a way that breaks memory safety. |
| **callback** | A function you give to other code so that it calls you back. |
| **build script** (`build.rs`) | A Rust program in your package root that Cargo runs before it compiles your crate. It can compile C code, generate files and print instructions to Cargo. |
| **static library** | A file (`.a` on macOS and Linux) with compiled code that the linker copies into the final program. |
| **linking** | The final build step that joins compiled pieces and finds the functions they call from each other. |
| **bindgen** | A tool that reads a C header file and writes the matching Rust declarations. |
| **RAII** (Resource Acquisition Is Initialization) | The pattern where a value owns a resource (memory, a file, a C allocation) and gives it back in its `Drop`. |
| **UB** (Undefined Behaviour) | A program state where the language gives no guarantee at all, so the compiler may produce any result. |
| **UTF-8** (8-bit Unicode Transformation Format) | The standard way to store text as bytes. Rust `str` is always valid UTF-8; C strings are just bytes. |

## 1. Why FFI

The world's important libraries are written in C: `zlib` (compression), SQLite (database), OpenSSL (cryptography), and the operating system's own interfaces. A systems engineer must be able to use them, and to be used by them: other languages (Python, Java) also call Rust through C.

C was designed before many ideas of Rust existed. In C, a function does not say who owns a pointer, how long it is valid, or whether it can be null. Rust cannot check any of that, so **every call across the border is `unsafe`** and you, the programmer, promise that the rules hold. The art of FFI is to make that border small and well-guarded:

```text
 your safe Rust code
┌────────────────────────────────────────┐
│  geo::distance(&a, &b)                 │   safe API: types, Result, Drop
├────────────────────────────────────────┤
│  small module with `unsafe` blocks     │   checks + // SAFETY: comments
├────────────────────────────────────────┤
│  unsafe extern "C" { fn point_distance }│   raw declarations
└───────────────────┬────────────────────┘
                    ▼
        C library (point_distance …)
```

## 2. Calling C functions that already exist

Your Rust program is already linked with the C standard library (libc), so its functions are available. You only have to *declare* them. Make a scratch project for the experiments of this section (`cargo new code/p06-ffi-libc`) and put this in its `src/main.rs`:

```rust
use std::ffi::{CString, c_char, c_int};

unsafe extern "C" {
    // `safe fn`: calling it cannot cause undefined behaviour.
    pub safe fn abs(input: c_int) -> c_int;

    // A plain `fn` in an `unsafe extern` block is unsafe to call.
    pub fn strlen(s: *const c_char) -> usize;
}

fn main() {
    println!("abs(-7) = {}", abs(-7));

    let text = CString::new("systems").unwrap();
    // SAFETY: `text.as_ptr()` points to a valid, NUL-terminated string
    // that stays alive until the end of the call.
    let length = unsafe { strlen(text.as_ptr()) };
    println!("strlen = {length}");
}
```

**What happens:**

1. `unsafe extern "C" { ... }` is a block of **declarations**: it tells the Rust compiler "these functions exist somewhere, and they use the C ABI". There is no code inside. (In Rust edition 2024 the word `unsafe` before `extern` is required: you promise that the declarations are correct.)
2. `pub safe fn abs(...)` marks a function as safe to call: it has no way to hurt memory. The compiler then lets you call `abs(-7)` without an `unsafe` block.
3. `pub fn strlen(...)` has no marker, so it is unsafe to call. It reads memory through a pointer until it finds a NUL byte, so a wrong pointer is a disaster.
4. `CString::new("systems")` makes a C-style string: the bytes of the text followed by a NUL byte. `text.as_ptr()` gives a `*const c_char` pointer to the first byte.
5. The `unsafe` block contains the call, and a `// SAFETY:` comment explains why the call is correct: the pointer is valid, the string ends with NUL, and `text` is alive during the call.

```text
abs(-7) = 7
strlen = 7
```

### Types: `c_int`, `c_char` and friends

C's `int` is not always the same size on every computer. So `std::ffi` has type aliases that always match C on your target: `c_int` (C `int`), `c_char`, `c_long`, `c_double`, and so on. Use them in declarations instead of `i32`.

| C type | Rust type |
|---|---|
| `int` | `c_int` (`i32` on your Mac) |
| `long` | `c_long` (64 bits on macOS and Linux, 32 bits on Windows) |
| `double` | `c_double` (`f64`) |
| `char` | `c_char` |
| `size_t` | `usize` |
| `const char *` | `*const c_char` |
| `int *` (output) | `*mut c_int` |
| `struct Point *` | `*mut Point` (with `#[repr(C)]` on `Point`) |

### Strings: `CString` and `CStr`

A Rust `String` is not a C string: it has no NUL at the end and may contain NUL bytes inside. Two types convert between them:

- **`CString`** — an *owned* C string. You create it from Rust data to **pass to C**. It fails if the text contains a NUL byte inside.
- **`CStr`** — a *borrowed* view of a C string. You make it from a pointer that C gave you, to **read in Rust**.

```rust
use std::ffi::{CStr, CString};

fn main() {
    let ok = CString::new("hello").unwrap();
    println!("{:?}", ok);
    println!("{:?}", ok.as_bytes_with_nul());

    let bad = CString::new("he\0llo");
    println!("{:?}", bad);

    let bytes = b"from C\0";
    let view = CStr::from_bytes_with_nul(bytes).unwrap();
    println!("{}", view.to_str().unwrap());
}
```

**What happens:**

1. `CString::new("hello")` returns `Ok`. `as_bytes_with_nul()` shows the five letters and the final `0`.
2. `CString::new("he\0llo")` returns `Err(NulError(2, ...))`: a NUL at position 2 would cut the string short for C, so Rust refuses.
3. `CStr::from_bytes_with_nul` checks that the bytes end with exactly one NUL and borrows them. `to_str()` checks that they are valid UTF-8 and gives a `&str`.

```text
"hello"
[104, 101, 108, 108, 111, 0]
Err(NulError(2, [104, 101, 0, 108, 108, 111]))
from C
```

> 🔒 **Security:** Text from outside is not always valid UTF-8. Never call `.unwrap()` on `to_str()` for strings that C or a user gave you; handle the error (return an error code, or use `to_string_lossy()`). And never let a NUL byte in user text silently cut a string: that has caused real security bugs.

## 3. Compiling your own C code

Most real FFI work uses a C library you build or link yourself. We write a small library and a Rust wrapper. The layout of the package:

```text
code/p06-ffi/
├── Cargo.toml
├── build.rs            ← compiles the C code
├── c/
│   ├── geometry.h      ← C declarations
│   └── geometry.c      ← C code
├── src/
│   ├── lib.rs          ← the safe Rust API
│   └── ffi.rs          ← raw `unsafe extern` declarations
└── examples/
    └── demo.rs
```

`c/geometry.h`:

```c
#ifndef GEOMETRY_H
#define GEOMETRY_H

#include <stddef.h>

typedef struct {
    double x;
    double y;
} Point;

/* Distance between two points. */
double point_distance(const Point *a, const Point *b);

/* Sum of `len` integers. */
long sum_array(const int *data, size_t len);

/* Divides a by b and stores the result in *out.
   Returns 0 on success and -1 if b is zero. */
int safe_divide(int a, int b, int *out);

/* Returns a new string "Hello, <name>!" allocated with malloc.
   The caller must give it back to free_greeting. */
char *make_greeting(const char *name);
void free_greeting(char *greeting);

/* Calls `callback(value)` for each of the `len` integers and stores
   the results in `out`. */
void map_ints(const int *data, size_t len, int (*callback)(int), int *out);

#endif
```

`c/geometry.c`:

```c
#include "geometry.h"

#include <math.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

double point_distance(const Point *a, const Point *b) {
    double dx = a->x - b->x;
    double dy = a->y - b->y;
    return sqrt(dx * dx + dy * dy);
}

long sum_array(const int *data, size_t len) {
    long total = 0;
    for (size_t i = 0; i < len; i++) {
        total += data[i];
    }
    return total;
}

int safe_divide(int a, int b, int *out) {
    if (b == 0) {
        return -1;
    }
    *out = a / b;
    return 0;
}

char *make_greeting(const char *name) {
    size_t size = strlen(name) + 9; /* "Hello, " + "!" + NUL */
    char *result = malloc(size);
    if (result == NULL) {
        return NULL;
    }
    snprintf(result, size, "Hello, %s!", name);
    return result;
}

void free_greeting(char *greeting) {
    free(greeting);
}

void map_ints(const int *data, size_t len, int (*callback)(int), int *out) {
    for (size_t i = 0; i < len; i++) {
        out[i] = callback(data[i]);
    }
}
```

Read the C code once and notice the typical C habits: functions return an **error code** and write the result through a pointer (`safe_divide`); memory comes from `malloc` and must go back to `free` (`make_greeting`/`free_greeting`); arrays are a pointer plus a length.

Now the build script. `Cargo.toml`:

```toml
[package]
name = "geo"
version = "0.1.0"
edition = "2024"

[dependencies]

[build-dependencies]
cc = "1.11"
```

`build.rs`:

```rust
fn main() {
    cc::Build::new()
        .file("c/geometry.c")
        .compile("geometry");

    println!("cargo::rerun-if-changed=c/geometry.c");
    println!("cargo::rerun-if-changed=c/geometry.h");
}
```

**What happens:**

1. Cargo sees `build.rs` in the package root. Before compiling `geo`, it compiles and runs this script. The script's dependencies go in `[build-dependencies]`.
2. `cc::Build::new().file(...).compile("geometry")` finds your C compiler (`clang` on macOS), compiles the file, packs it into `libgeometry.a`, and prints the Cargo instructions that tell Cargo to link that library into your crate.
3. Lines that start with `cargo::` are instructions for Cargo. `cargo::rerun-if-changed=...` means: run the script again only when those files change.

### The raw declarations

`src/ffi.rs` copies the C header into Rust, one function at a time:

```rust
//! Raw declarations of the C functions. Everything here is unsafe to use.

use std::ffi::{c_char, c_double, c_int, c_long};

#[repr(C)]
#[derive(Debug, Clone, Copy)]
pub struct Point {
    pub x: c_double,
    pub y: c_double,
}

unsafe extern "C" {
    pub fn point_distance(a: *const Point, b: *const Point) -> c_double;
    pub fn sum_array(data: *const c_int, len: usize) -> c_long;
    pub fn safe_divide(a: c_int, b: c_int, out: *mut c_int) -> c_int;
    pub fn make_greeting(name: *const c_char) -> *mut c_char;
    pub fn free_greeting(greeting: *mut c_char);
    pub fn map_ints(
        data: *const c_int,
        len: usize,
        callback: extern "C" fn(c_int) -> c_int,
        out: *mut c_int,
    );
}
```

**What happens:**

1. `#[repr(C)]` on `Point` makes Rust lay out the two `f64` fields in the order and with the padding that C uses: 16 bytes, `x` at offset 0, `y` at offset 8. Without it, Rust may reorder fields and C would read garbage.
2. Every declaration must match the C header exactly. The compiler **cannot check this**: if you declare `int` as `i64`, you get undefined behaviour with no warning. (That is why people use `bindgen`, see section 7.)
3. `callback: extern "C" fn(c_int) -> c_int` is a function pointer with the C calling convention. A Rust function can be used as a callback only if it is declared `extern "C"`.

## 4. The safe wrapper

The raw functions should not be used all over your program. Wrap them once, in `src/lib.rs`, so that the rest of the program never writes `unsafe`:

```rust
//! A safe wrapper around a tiny C library.

mod ffi;

use std::ffi::{CStr, CString, c_int};

pub use ffi::Point;

/// Distance between two points.
pub fn distance(a: &Point, b: &Point) -> f64 {
    // SAFETY: both pointers come from references, so they are valid,
    // aligned and not null; the C function only reads them.
    unsafe { ffi::point_distance(a, b) }
}

/// Sum of all numbers in the slice.
pub fn sum(data: &[i32]) -> i64 {
    // SAFETY: `data.as_ptr()` is valid for `data.len()` reads of `i32`
    // (an empty slice gives a dangling but non-null pointer, and C reads nothing).
    unsafe { ffi::sum_array(data.as_ptr(), data.len()) as i64 }
}

/// Error returned by [`divide`].
#[derive(Debug, PartialEq)]
pub struct DivideByZero;

/// Divides `a` by `b`.
pub fn divide(a: i32, b: i32) -> Result<i32, DivideByZero> {
    let mut out: c_int = 0;
    // SAFETY: `&mut out` is a valid pointer to one `c_int`.
    let code = unsafe { ffi::safe_divide(a, b, &mut out) };
    if code == 0 { Ok(out) } else { Err(DivideByZero) }
}

/// A greeting allocated by C. It is given back to C when dropped.
pub struct Greeting {
    raw: *mut std::ffi::c_char,
}

impl Greeting {
    /// Returns `None` if `name` contains a NUL byte or C ran out of memory.
    pub fn new(name: &str) -> Option<Greeting> {
        let name = CString::new(name).ok()?;
        // SAFETY: `name` is a valid NUL-terminated string for the whole call.
        let raw = unsafe { ffi::make_greeting(name.as_ptr()) };
        if raw.is_null() { None } else { Some(Greeting { raw }) }
    }

    pub fn as_str(&self) -> &str {
        // SAFETY: `raw` is non-null (checked in `new`), points to a
        // NUL-terminated string, and lives until `drop`.
        unsafe { CStr::from_ptr(self.raw) }.to_str().unwrap_or("")
    }
}

impl Drop for Greeting {
    fn drop(&mut self) {
        // SAFETY: `raw` came from `make_greeting` and is freed exactly once.
        unsafe { ffi::free_greeting(self.raw) }
    }
}

/// Applies `callback` to every number, using C to do the loop.
pub fn map(data: &[i32], callback: extern "C" fn(i32) -> i32) -> Vec<i32> {
    let mut out = vec![0; data.len()];
    // SAFETY: `data` and `out` both have `data.len()` elements.
    unsafe { ffi::map_ints(data.as_ptr(), data.len(), callback, out.as_mut_ptr()) };
    out
}
```

**What happens, function by function:**

1. `distance`: Rust references `&Point` are always valid and aligned, and they coerce to `*const Point`. The C function only reads. One line of `unsafe`.
2. `sum`: a Rust slice is exactly "pointer + length", the same as C's convention. The wrapper passes both, so C can never read past the end.
3. `divide`: turns the C pattern "error code + out-pointer" into a Rust `Result`. The caller never sees the pointer.
4. `Greeting`: the string was allocated by C's `malloc`. **Rust must not free it** (Rust's allocator is a different one!). The struct owns the pointer and calls C's `free_greeting` in `Drop`. This is **RAII**: the memory is freed exactly once, even when an early `return` or a panic happens. The constructor checks for a NUL inside the name and for a null result.
5. `map`: passes a Rust `extern "C" fn` as the callback. `out` is allocated by Rust with the right length, so C writes inside it.

Use it from `examples/demo.rs`:

```rust
use geo::{Greeting, Point, distance, divide, map, sum};

extern "C" fn square(x: i32) -> i32 {
    x * x
}

fn main() {
    let a = Point { x: 0.0, y: 0.0 };
    let b = Point { x: 3.0, y: 4.0 };
    println!("distance = {}", distance(&a, &b));

    println!("sum = {}", sum(&[10, 20, 30]));

    println!("10 / 3 = {:?}", divide(10, 3));
    println!("1 / 0 = {:?}", divide(1, 0));

    let greeting = Greeting::new("Bobur").unwrap();
    println!("{}", greeting.as_str());

    println!("squares = {:?}", map(&[1, 2, 3, 4], square));
}
```

Run it with `cargo run --example demo`:

```text
distance = 5
sum = 60
10 / 3 = Ok(3)
1 / 0 = Err(DivideByZero)
Hello, Bobur!
squares = [1, 4, 9, 16]
```

> 📏 **Convention:** Keep `unsafe extern` declarations in one private module (`ffi`), and give the rest of the program only safe functions. Name the module `ffi` or `sys`. Big projects split it in two crates: `foo-sys` (raw declarations) and `foo` (the safe wrapper).

> 🧪 **Testing:** Test the **wrapper**, not the C library. Test the normal case, the empty input, and every error branch. Add tests to the end of `lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn distance_of_3_4_5() {
        let a = Point { x: 0.0, y: 0.0 };
        let b = Point { x: 3.0, y: 4.0 };
        assert_eq!(distance(&a, &b), 5.0);
    }

    #[test]
    fn sum_of_numbers() {
        assert_eq!(sum(&[1, 2, 3, 4]), 10);
        assert_eq!(sum(&[]), 0);
    }

    #[test]
    fn divide_ok_and_error() {
        assert_eq!(divide(10, 2), Ok(5));
        assert_eq!(divide(1, 0), Err(DivideByZero));
    }

    #[test]
    fn greeting_round_trip() {
        let g = Greeting::new("Aziza").unwrap();
        assert_eq!(g.as_str(), "Hello, Aziza!");
    }

    #[test]
    fn greeting_rejects_nul() {
        assert!(Greeting::new("a\0b").is_none());
    }

    extern "C" fn double(x: i32) -> i32 {
        x * 2
    }

    #[test]
    fn map_doubles() {
        assert_eq!(map(&[1, 2, 3], double), vec![2, 4, 6]);
    }
}
```

`cargo test` prints `test result: ok. 6 passed; 0 failed`.

**Miri and FFI.** In Lesson 06.6 you used Miri to find undefined behaviour in Rust code. Miri is an *interpreter for Rust*; it cannot run compiled C code. Here is what happens:

```text
$ cargo +nightly miri test    (shortened output)
test tests::distance_of_3_4_5 ... error: unsupported operation: can't call foreign function `point_distance` on OS `macos`
  --> src/lib.rs:13:14
   |
13 |     unsafe { ffi::point_distance(a, b) }
   |              ^^^^^^^^^^^^^^^^^^^^^^^^^ unsupported operation occurred here
   |
   = help: this means the program tried to do something Miri does not support; it does not indicate a bug in the program
```

So Miri checks only the pure-Rust part of an FFI crate. For the C side, people use C sanitizers and Valgrind; later phases cover tools like these.

## 5. Calling Rust from C

The border has two directions. To let C call a Rust function, make it `extern "C"` and prevent name mangling (Rust normally rewrites function names so that different items do not clash; C needs the plain name):

```rust
use std::ffi::{CStr, c_char, c_int};

/// Adds two numbers. Callable from C as `rustmath_add`.
#[unsafe(no_mangle)]
pub extern "C" fn rustmath_add(a: c_int, b: c_int) -> c_int {
    a.wrapping_add(b)
}

/// Returns the number of characters (not bytes) in a UTF-8 C string,
/// or -1 if the pointer is null or the text is not valid UTF-8.
///
/// # Safety
/// `text` must be null or point to a NUL-terminated string.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn rustmath_char_count(text: *const c_char) -> c_int {
    if text.is_null() {
        return -1;
    }
    // SAFETY: the caller promised that `text` is a NUL-terminated string.
    let text = unsafe { CStr::from_ptr(text) };
    match text.to_str() {
        Ok(s) => s.chars().count() as c_int,
        Err(_) => -1,
    }
}
```

In `Cargo.toml` ask for a static library:

```toml
[lib]
crate-type = ["staticlib"]
```

Write a C header by hand (the tool **cbindgen** can generate it from your Rust code), `rustmath.h`:

```c
#ifndef RUSTMATH_H
#define RUSTMATH_H

int rustmath_add(int a, int b);
int rustmath_char_count(const char *text);

#endif
```

and a C program, `main.c`:

```c
#include <stdio.h>
#include "rustmath.h"

int main(void) {
    printf("2 + 3 = %d\n", rustmath_add(2, 3));
    printf("characters in \"salom\": %d\n", rustmath_char_count("salom"));
    printf("null pointer: %d\n", rustmath_char_count(NULL));
    return 0;
}
```

Build and link them:

```sh
cargo build --release
clang main.c target/release/librustmath.a -o demo
./demo
```

```text
2 + 3 = 5
characters in "salom": 5
null pointer: -1
```

**What happens:**

1. `#[unsafe(no_mangle)]` keeps the symbol name `rustmath_add`. In edition 2024 this attribute must be written inside `unsafe(...)`, because a wrong name can break the program.
2. `crate-type = ["staticlib"]` makes `cargo build` produce `librustmath.a`, a file with all the code, including the parts of the Rust standard library it needs.
3. `clang` compiles `main.c` and links it with that file. The C program calls Rust like any C function.
4. `rustmath_char_count` is `unsafe extern "C"` because its safety depends on what the C caller passes. Its `# Safety` doc section tells the caller the rule. It still checks for a null pointer, which is cheap.
5. `wrapping_add` avoids a Rust panic on overflow: **a panic must never cross the border into C**.

> ⚠️ **Common mistake:** letting a panic escape from an `extern "C"` function. Unwinding through C frames is undefined in C. Since Rust 1.81, Rust aborts the process instead. Real output (shortened) of a program whose `extern "C" fn boom()` calls `panic!("oh no")`:
>
> ```text
> calling boom
>
> thread 'main' panicked at src/main.rs:2:5:
> oh no
>
> thread 'main' panicked at library/core/src/panicking.rs:225:5:
> panic in a function that cannot unwind
> thread caused non-unwinding panic. aborting.
> ```
>
> (The process ends with exit code 134. That is the signal `SIGABRT`, "abort": the program tells the operating system to stop it at once.) Inside `extern "C"` functions, return error codes and avoid `unwrap()`, `expect()` and array indexing that can panic. If a panic is possible, catch it with `std::panic::catch_unwind` and turn it into an error code.

## 6. The rules of the border

When you write `unsafe` FFI code, check this list for **every** call. Each item is a real source of bugs.

1. **Types match.** The Rust declaration has the same argument types, the same struct layout (`#[repr(C)]`) and the same calling convention (`extern "C"`) as the C code.
2. **Pointers are valid.** Not null (unless C accepts null), aligned, pointing to enough initialized memory, and alive for the whole call. A temporary is the classic mistake (see Exercise 3).
3. **Lengths are right.** A pointer to an array always travels with a length, and the length is never larger than the real data. A wrong length is a **buffer overflow**.
4. **Strings are NUL-terminated** when going into C (`CString`), and are not assumed to be UTF-8 when coming out (`CStr::to_str` returns a `Result`).
5. **Ownership is clear.** Whoever allocates, frees, with the same allocator. Memory from C's `malloc` goes back through C's `free`. Rust memory is never freed by C.
6. **Mutability and aliasing.** If C writes through a pointer, give it `*mut` made from `&mut`. While C holds a pointer, do not use the same data from Rust.
7. **No panics cross the border**, in either direction.
8. **Threads.** If the C library is not thread-safe, do not call it from several threads at once (`Send` and `Sync` for your wrapper types tell the compiler; raw pointers are neither by default, which protects you).
9. **Callbacks.** A callback must be `extern "C"`. If the C side may store the pointer and call it later, it must stay valid that long. C APIs often add a `void *user_data` argument so that you can pass state to the callback.
10. **Document it.** Every `unsafe` block gets a `// SAFETY:` comment, every `unsafe fn` gets a `# Safety` section.

> 🔒 **Security:** FFI is where Rust's memory safety ends. A mismatch in a length, or a pointer used after `free`, gives the same exploitable bugs as in C. Keep the unsafe surface tiny, test the wrapper with unexpected inputs (empty, huge, NUL inside), and have the code reviewed.

## 7. `bindgen`: writing declarations by machine

Copying declarations by hand is error-prone. **bindgen** reads the C header and writes them. In `Cargo.toml`, add `bindgen = "0.73"` to `[build-dependencies]` (it needs `libclang`, which comes with the Xcode command line tools), and in `build.rs`:

```rust
use std::env;
use std::path::PathBuf;

fn main() {
    cc::Build::new().file("c/geometry.c").compile("geometry");

    let bindings = bindgen::Builder::default()
        .header("c/geometry.h")
        .parse_callbacks(Box::new(bindgen::CargoCallbacks::new()))
        .generate()
        .expect("bindgen could not generate bindings");

    let out_dir = PathBuf::from(env::var("OUT_DIR").unwrap());
    bindings
        .write_to_file(out_dir.join("bindings.rs"))
        .expect("could not write bindings.rs");
}
```

and in the Rust code, include the generated file:

```rust
include!(concat!(env!("OUT_DIR"), "/bindings.rs"));
```

`OUT_DIR` is a folder that Cargo gives to the build script for its output files. Here is a part of the real generated file:

```rust
#[repr(C)]
#[derive(Debug, Copy, Clone)]
pub struct Point {
    pub x: f64,
    pub y: f64,
}
const _: () = {
    ["Size of Point"][::std::mem::size_of::<Point>() - 16usize];
    ["Alignment of Point"][::std::mem::align_of::<Point>() - 8usize];
    ["Offset of field: Point::x"][::std::mem::offset_of!(Point, x) - 0usize];
    ["Offset of field: Point::y"][::std::mem::offset_of!(Point, y) - 8usize];
};
unsafe extern "C" {
    pub fn point_distance(a: *const Point, b: *const Point) -> f64;
}
unsafe extern "C" {
    pub fn map_ints(
        data: *const ::std::os::raw::c_int,
        len: usize,
        callback: ::std::option::Option<
            unsafe extern "C" fn(arg1: ::std::os::raw::c_int) -> ::std::os::raw::c_int,
        >,
        out: *mut ::std::os::raw::c_int,
    );
}
```

Notice two things. The generated `const _` block checks the struct's size and offsets **at compile time**: it fails to build if Rust and C disagree. And the callback is `Option<unsafe extern "C" fn(...)>`: in C a function pointer can be `NULL`, and `Option` is how Rust says "maybe nothing". bindgen is the safer way for big headers; you still write the safe wrapper by hand. The opposite tool, **cbindgen**, writes a C header from your Rust `extern "C"` functions.

## Common mistakes

**1. Calling an `unsafe` function without `unsafe`.**

```text
error[E0133]: call to unsafe function `strlen` is unsafe and requires unsafe block
 --> src/main.rs:9:13
  |
9 |     let n = strlen(text.as_ptr());
  |             ^^^^^^^^^^^^^^^^^^^^^ call to unsafe function
  |
  = note: consult the function's documentation for information on how to avoid undefined behavior
```

Fix: wrap the call in `unsafe { ... }` with a `// SAFETY:` comment, or build a safe wrapper function.

**2. Old-style `extern` block (edition 2024).**

```text
error: extern blocks must be unsafe
 --> src/main.rs:3:1
  |
3 |   extern "C" {
  |   ^
  |   |
  |  _help: needs `unsafe` before the extern keyword: `unsafe`
```

Fix: write `unsafe extern "C" { ... }`.

**3. Passing a Rust `String` where C wants a pointer.**

```text
error[E0308]: mismatched types
 --> src/main.rs:9:29
  |
9 |     let n = unsafe { strlen(text) };
  |                      ------ ^^^^ expected `*const i8`, found `String`
  |
  = note: expected raw pointer `*const i8`
                  found struct `String`
```

Fix: `let c = CString::new(text)?; strlen(c.as_ptr())`.

**4. Old-style `#[no_mangle]` (edition 2024).**

```text
error: unsafe attribute used without unsafe
 --> src/main.rs:1:3
  |
1 | #[no_mangle]
  |   ^^^^^^^^^ usage of unsafe attribute
  |
help: wrap the attribute in `unsafe(...)`
```

Fix: `#[unsafe(no_mangle)]`.

**5. A dangling pointer from a temporary.** The compiler warns you (see Exercise 3).

**6. Wrong declaration.** Declaring `fn abs(x: i64) -> i64` for C's `int abs(int)`: no error, wrong results or crashes. The compiler cannot check C. Use `std::ffi` types, and `bindgen` for big headers.

## Try it yourself

1. Build the `geo` package: create the files from sections 3 and 4 (`c/geometry.h`, `c/geometry.c`, `build.rs`, `src/ffi.rs`, `src/lib.rs`, `examples/demo.rs`) and add `cc` to `[build-dependencies]`.
2. Run `cargo run --example demo`. Compare with the output in section 4.
3. Run `cargo test`. All six tests from section 4 should pass.
4. Add a test `distance_of_6_8_10` that checks the distance between `(0, 0)` and `(6, 8)` is `10.0`. Run `cargo test` again: seven tests pass.
5. Make a Git commit: `git add code/p06-ffi && git commit -m "feat(p06-ffi): safe wrapper around a small C library"`. Add `target/` to `.gitignore` if it is not there.

## Exercises

### Exercise 1 — `atoi` from C (easy)

Declare the C function `int atoi(const char *)` and write `fn parse_with_c(text: &str) -> Option<i32>` that converts text to a number with it. It must return `None` if the text contains a NUL byte. Print the result for `"1234"`, `"42abc"`, `"hello"` and `"a\0b"`.

<details>
<summary>Hint</summary>

Make a `CString` with `CString::new(text).ok()?`. Pass `text.as_ptr()` inside an `unsafe` block. `atoi` returns 0 for text that is not a number.

</details>

<details>
<summary>Answer</summary>

```rust
use std::ffi::{CString, c_char, c_int};

unsafe extern "C" {
    fn atoi(s: *const c_char) -> c_int;
}

fn parse_with_c(text: &str) -> Option<i32> {
    let text = CString::new(text).ok()?;
    // SAFETY: `text` is a valid NUL-terminated string that outlives the call.
    Some(unsafe { atoi(text.as_ptr()) })
}

fn main() {
    println!("{:?}", parse_with_c("1234"));
    println!("{:?}", parse_with_c("42abc"));
    println!("{:?}", parse_with_c("hello"));
    println!("{:?}", parse_with_c("a\0b"));
}
```

```text
Some(1234)
Some(42)
Some(0)
None
```

Why it works: `atoi` stops at the first character that is not a digit and returns 0 when there are no digits at all (so it cannot tell "0" from an error; Rust's `str::parse` is a better tool, this is only practice). `CString::new` refuses the text with an inner NUL.

</details>

### Exercise 2 — `count_positive` with tests (medium)

Add a C function `size_t count_positive(const int *data, size_t len)` to `geometry.c` and `geometry.h`. Declare it in `ffi.rs`, write the safe wrapper `pub fn count_positive(data: &[i32]) -> usize` and tests. Also write a test that checks that `Point` has the C layout: size 16, alignment 8, `x` at offset 0, `y` at offset 8.

<details>
<summary>Hint</summary>

The wrapper is almost the same as `sum`. For the layout test use `std::mem::size_of`, `std::mem::align_of` and `std::mem::offset_of!(Point, y)`. Remember to add the new C function to the header too.

</details>

<details>
<summary>Answer</summary>

Add to `geometry.h` (before `#endif`):

```c
/* Counts how many of the `len` integers are greater than zero. */
size_t count_positive(const int *data, size_t len);
```

Add to `geometry.c`:

```c
size_t count_positive(const int *data, size_t len) {
    size_t count = 0;
    for (size_t i = 0; i < len; i++) {
        if (data[i] > 0) {
            count++;
        }
    }
    return count;
}
```

In `ffi.rs`, inside the `extern` block:

```rust
    pub fn count_positive(data: *const c_int, len: usize) -> usize;
```

In `lib.rs`:

```rust
/// How many numbers in the slice are greater than zero.
pub fn count_positive(data: &[i32]) -> usize {
    // SAFETY: `data.as_ptr()` is valid for `data.len()` reads of `i32`.
    unsafe { ffi::count_positive(data.as_ptr(), data.len()) }
}
```

Tests:

```rust
    #[test]
    fn counts_positive_numbers() {
        assert_eq!(count_positive(&[3, -1, 0, 7, -5, 2]), 3);
        assert_eq!(count_positive(&[]), 0);
    }

    #[test]
    fn point_has_the_c_layout() {
        assert_eq!(std::mem::size_of::<Point>(), 16);
        assert_eq!(std::mem::align_of::<Point>(), 8);
        assert_eq!(std::mem::offset_of!(Point, x), 0);
        assert_eq!(std::mem::offset_of!(Point, y), 8);
    }
```

`cargo test` shows `test result: ok. 8 passed`.

Why it works: the build script has `rerun-if-changed` for both C files, so Cargo recompiles them. `#[repr(C)]` guarantees the layout that the layout test checks.

</details>

### Exercise 3 — The dangling pointer (medium)

This program compiles with a warning. Read it, explain what is wrong, and fix it.

```rust
use std::ffi::{CString, c_char};

unsafe extern "C" {
    fn strlen(s: *const c_char) -> usize;
}

fn main() {
    let ptr = CString::new("systems").unwrap().as_ptr();
    let n = unsafe { strlen(ptr) };
    println!("{n}");
}
```

<details>
<summary>Hint</summary>

When is the `CString` created by `CString::new(...).unwrap()` dropped? Look at the end of the `let ptr = ...;` statement.

</details>

<details>
<summary>Answer</summary>

The compiler says:

```text
warning: this creates a dangling pointer because temporary `CString` is dropped at end of statement
 --> src/main.rs:8:48
  |
8 |     let ptr = CString::new("systems").unwrap().as_ptr();
  |               -------------------------------- ^^^^^^ pointer created here
  |               |
  |               this `CString` is dropped at end of statement
  |
  = help: bind the `CString` to a variable such that it outlives the pointer returned by `as_ptr`
```

The `CString` is a temporary value. It is freed at the `;`, so `ptr` points to freed memory. `strlen` then reads memory that is no longer ours: undefined behaviour (in one run it printed `1`, but any result, or a crash, is possible). Fix: keep the `CString` in a variable that lives as long as the pointer is used.

```rust
use std::ffi::{CString, c_char};

unsafe extern "C" {
    fn strlen(s: *const c_char) -> usize;
}

fn main() {
    let text = CString::new("systems").unwrap();
    // SAFETY: `text` lives until the end of `main`, so the pointer stays valid.
    let n = unsafe { strlen(text.as_ptr()) };
    println!("{n}");
}
```

```text
7
```

Why it works: `text` owns the bytes and is dropped at the end of `main`, after the call. A pointer is only as valid as the owner that is behind it.

</details>

### Exercise 4 — Export `rust_reverse` to C (hard)

Create a static library with a function `ssize_t rust_reverse(const char *input, char *out, size_t cap)`. It writes the characters of `input` in reverse order into the buffer `out` (as a NUL-terminated string) and returns the number of bytes written. It returns `-1` for a null pointer, text that is not valid UTF-8, or a buffer that is too small. Call it from a C program with a buffer of 16 bytes and then with a buffer of 4 bytes, for the text `"salom"`.

<details>
<summary>Hint</summary>

Declare it `pub unsafe extern "C" fn` with `#[unsafe(no_mangle)]`, and parameters `*const c_char`, `*mut c_char`, `usize`; return `isize`. Build the reversed `String` with `text.chars().rev().collect()`. You need `bytes.len() + 1` bytes of room (for the NUL). Copy with `std::ptr::copy_nonoverlapping` and write the final `0` with `*out.add(len) = 0`.

</details>

<details>
<summary>Answer</summary>

`src/lib.rs` (the `Cargo.toml` has `crate-type = ["staticlib"]`):

```rust
use std::ffi::{CStr, c_char};

/// Writes the reverse of `input` into `out` (as a NUL-terminated string).
/// Returns the number of bytes written (without the NUL), or -1 on error:
/// a null pointer, text that is not valid UTF-8, or a buffer that is too small.
///
/// # Safety
/// `input` must be null or a NUL-terminated string. `out` must be null or
/// valid for writes of `cap` bytes.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn rust_reverse(input: *const c_char, out: *mut c_char, cap: usize) -> isize {
    if input.is_null() || out.is_null() {
        return -1;
    }
    // SAFETY: the caller promised that `input` is a NUL-terminated string.
    let Ok(text) = unsafe { CStr::from_ptr(input) }.to_str() else {
        return -1;
    };
    let reversed: String = text.chars().rev().collect();
    let bytes = reversed.as_bytes();
    if bytes.len() + 1 > cap {
        return -1;
    }
    // SAFETY: `out` is valid for `cap` bytes and `bytes.len() + 1 <= cap`.
    unsafe {
        std::ptr::copy_nonoverlapping(bytes.as_ptr(), out as *mut u8, bytes.len());
        *out.add(bytes.len()) = 0;
    }
    bytes.len() as isize
}
```

`main.c`:

```c
#include <stdio.h>
#include <sys/types.h>

ssize_t rust_reverse(const char *input, char *out, size_t cap);

int main(void) {
    char buffer[16];
    ssize_t n = rust_reverse("salom", buffer, sizeof buffer);
    printf("%zd: %s\n", n, buffer);

    char tiny[4];
    n = rust_reverse("salom", tiny, sizeof tiny);
    printf("too small: %zd\n", n);
    return 0;
}
```

```sh
cargo build --release
clang main.c target/release/librevlib.a -o demo
./demo
```

```text
5: molas
too small: -1
```

(The static library is named after the package; here the package is `revlib`.)

Why it works: the function never writes more than `cap` bytes. It checks for room **before** writing, which is the rule that prevents buffer overflows. It uses `chars().rev()` so that multi-byte UTF-8 letters stay intact.

</details>

### Exercise 5 — Your own `Greeting` for long names (hard)

The C function `make_greeting` allocates `strlen(name) + 9` bytes. Write a test for `Greeting::new` with a name of 10,000 characters and check that `as_str()` has length 10,008. Then answer: what happens if `Greeting` did *not* implement `Drop`? Write a short test-like loop that would leak memory.

<details>
<summary>Hint</summary>

`"x".repeat(10_000)` makes the long name. "Hello, " is 7 characters and "!" is 1. A loop of `Greeting::new("x").unwrap()` that calls `std::mem::forget(g)` leaks the same way as a missing `Drop`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn long_name_works() {
        let name = "x".repeat(10_000);
        let g = Greeting::new(&name).unwrap();
        assert_eq!(g.as_str().len(), 10_008);
    }
```

Without `Drop`, the C memory from `malloc` would never be freed. A loop like this would leak about 10 bytes per iteration, forever:

```rust
for _ in 0..1_000_000 {
    let g = Greeting::new("x").unwrap();
    std::mem::forget(g); // skips Drop: same as having no Drop at all
}
```

Why it works: `10_000 + 7 + 1 = 10_008`. Rust does not manage memory that came from C; only your `Drop` implementation gives it back, which is why the wrapper type owns the pointer (RAII).

</details>

## Quiz

1. What does `unsafe extern "C" { fn f(); }` do?

<details>
<summary>Answer</summary>

It declares that a function `f` exists outside Rust and uses the C calling convention. There is no code in the block. The `unsafe` means you promise that the declarations match the real functions.

</details>

2. Why are calls to C functions `unsafe` by default?

<details>
<summary>Answer</summary>

The Rust compiler cannot see the C code and cannot check pointers, lengths, ownership or types. The programmer must guarantee the rules. A function can be marked `safe fn` in the declaration only when it cannot cause undefined behaviour.

</details>

3. What is the difference between `CString` and `CStr`?

<details>
<summary>Answer</summary>

`CString` is an owned C string made in Rust (to pass to C; ends with NUL, cannot contain NUL inside). `CStr` is a borrowed view of a C string that already exists (to read a pointer that C gave you).

</details>

4. Why does `Greeting` call `free_greeting` instead of letting Rust free the pointer?

<details>
<summary>Answer</summary>

The memory was allocated by C's `malloc`. It must be freed by the matching C function. Rust's allocator is a different one, and freeing with the wrong allocator is undefined behaviour.

</details>

5. What does `#[repr(C)]` do?

<details>
<summary>Answer</summary>

It makes the struct use C's memory layout (field order and padding), so Rust and C agree on sizes and offsets.

</details>

6. What happens if a panic escapes from an `extern "C"` function?

<details>
<summary>Answer</summary>

Since Rust 1.81 the process aborts (`panic in a function that cannot unwind`). Unwinding through C is not allowed, so such functions must not panic: return error codes, or catch panics with `catch_unwind`.

</details>

7. Why does Miri fail on tests that call a C library?

<details>
<summary>Answer</summary>

Miri interprets Rust code and cannot run foreign compiled code. It reports `can't call foreign function`.

</details>

## Summary

- FFI lets Rust call C and C call Rust through the C ABI. The border is `unsafe`: you promise that types, pointers, lengths and ownership are right.
- Declare C functions in `unsafe extern "C" { ... }`; use `std::ffi` types (`c_int`, `c_char`); mark structs `#[repr(C)]`.
- Use `CString` to pass text to C and `CStr` to read text from C. Handle inner NULs and invalid UTF-8.
- Compile your own C code with `build.rs` and the `cc` crate; `bindgen` writes declarations from headers.
- Hide the unsafe code in one small module and expose a safe API: `Result` instead of error codes, RAII (`Drop`) for C allocations.
- Export Rust with `#[unsafe(no_mangle)] pub extern "C" fn`, build a `staticlib`, and never let a panic cross the border.
- Document every `unsafe` block with `// SAFETY:` and test the safe wrapper, including errors and empty input.

## Sources and further reading

- [The Rustonomicon — Foreign Function Interface](https://doc.rust-lang.org/nomicon/ffi.html) — the long official guide to FFI, callbacks, and panics.
- [std::ffi module](https://doc.rust-lang.org/std/ffi/index.html) — `CString`, `CStr` and the `c_int`-style type aliases.
- [Edition Guide — Unsafe extern blocks](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-extern.html) — why edition 2024 requires `unsafe extern` and how `safe fn` works.
- [Edition Guide — Unsafe attributes](https://doc.rust-lang.org/edition-guide/rust-2024/unsafe-attributes.html) — why `#[unsafe(no_mangle)]`.
- [Cargo Book — Build scripts](https://doc.rust-lang.org/cargo/reference/build-scripts.html) — what `build.rs` can do and the `cargo::` instructions.
- [cc crate](https://docs.rs/cc/latest/cc/) — compiling C code from a build script.
- [The bindgen User Guide](https://rust-lang.github.io/rust-bindgen/) — generating Rust declarations from C headers.

---

[🏠 Start](../START-HERE.md) · [← 06.9 Introduction to async/await](09-async-await-introduction.md) · [📋 Phase 06](README.md) · [06.11 Project: multithreaded web server →](11-project-multithreaded-web-server.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
