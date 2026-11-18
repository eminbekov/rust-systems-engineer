<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 12.11 Memory layout of Rust types](11-memory-layout-of-rust-types.md) · [📋 Phase 12](README.md) · [12.13 How memory allocators work →](13-how-memory-allocators-work.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/12-linking-and-loading.md) <!-- nav -->

# Lesson 12.12 — Linking and loading programs

> **Phase 12 — Computer systems** · Lesson 12 of 15 · ⏱ about 135 minutes

## What you will learn

- The four steps from source code to a running process: compile, object files, link, load.
- What symbols are, how to list them with `nm`, and why Rust mangles names.
- What the linker does, what static and dynamic linking mean, and how to read an "undefined symbol" error.
- How to inspect executables: Mach-O on macOS with `file`, `otool` and `size`; ELF on Linux with `readelf`, `ldd` and `objdump`.
- What the loader does when a program starts, and why addresses change on every run.

## Before you start

- You finished [Lesson 12.11 — Memory layout of Rust types](11-memory-layout-of-rust-types.md).
- You remember "from source code to a running program" from [Lesson 00.3](../00-getting-started/03-how-a-computer-runs-a-program.md), FFI (Foreign Function Interface) and `extern "C"` from [Lesson 06.10 — FFI: calling C code](../06-rust-advanced/10-ffi-calling-c.md), and Docker containers from [Lesson 11.12](../11-linux-shell-and-containers/12-containers-and-docker-basics.md).
- Create the package and put this program into `src/main.rs`. We use it for the whole lesson:

```bash
cd ~/projects/rust
cargo new code/p12-linking
cd code/p12-linking
```

```rust
static GREETING: &str = "hello from a static";

#[inline(never)]
fn add_numbers(a: u64, b: u64) -> u64 {
    a + b
}

fn main() {
    let sum = add_numbers(2, 40);
    println!("{GREETING}: {sum}");
    println!("main is at        {:p}", main as fn() as *const ());
    println!("add_numbers is at {:p}", add_numbers as fn(u64, u64) -> u64 as *const ());
    println!("GREETING is at    {:p}", &GREETING);
    let local = 7;
    println!("a local is at     {:p}", &local);
}
```

`#[inline(never)]` keeps `add_numbers` as a real function, so that we can find it later. `{:p}` prints an address; `main as fn() as *const ()` turns the function into a plain pointer so it can be printed.

## New words

| Word | Meaning |
|---|---|
| **object file** | The output of compiling one piece of code: machine code and data, plus a list of names it defines and names it still needs. Not runnable yet. |
| **symbol** | A name in an object file or executable for a function or a static, together with its address. |
| **symbol table** | The list of all symbols in a file. `nm` prints it. |
| **name mangling** | Encoding a full Rust path (crate, module, generics) into one unique symbol name, such as `_RNvCs…11p12_linking11add_numbers`. |
| **linker** | The program that combines object files and libraries into one executable and fills in every address that was still missing. |
| **relocation** | A note in an object file that says "put the final address of symbol X here". The linker (or the loader) processes it. |
| **static linking** | Copying the needed library code into the executable at build time. |
| **dynamic linking** | Leaving library code in a separate shared library file that is found and connected when the program starts. |
| **shared library** | A library file used by many programs at once: `.dylib` on macOS, `.so` (shared object) on Linux. |
| **loader** (dynamic linker) | The program that starts first when you run a dynamically linked executable: `dyld` on macOS, `ld.so` on Linux. It loads shared libraries and connects symbols. |
| **Mach-O** (Mach object) | The format of executables, libraries and object files on macOS. |
| **ELF** (Executable and Linkable Format) | The format of executables, libraries and object files on Linux and most other Unix systems. |
| **segment / section** | Parts of an executable. Sections are what the linker works with (`.text` for code, `.data` for data); segments are what the loader maps into memory. |
| **ASLR** (Address Space Layout Randomization) | The operating system loading a program at a different random address on every run, which makes attacks harder. |
| **PIE** (Position-Independent Executable) | An executable whose code works at any address, so ASLR can move it. Rust builds PIE by default. |
| **rlib** (Rust library) | The archive format Cargo uses for compiled Rust crates; it contains object files and metadata. |

## 1. From source code to a running process

Building and running a Rust program happens in four steps:

```text
  src/main.rs ──► rustc ──► object files (.o) ─┐
  std, core, …  (already compiled, .rlib)  ────┼──► linker (cc / ld) ──► executable file
  system C library (libSystem, libc)  ─────────┘                          │
                                                               you run it ▼
                                                    loader (dyld / ld.so) ──► process
```

1. **Compile.** `rustc` turns each crate into machine code, cut into one or more object files. At this point a call to `println!` is still only a name: the code of `std` is somewhere else.
2. **Object files and libraries.** The standard library was compiled when Rust was installed and lives in `.rlib` files. The C library of the operating system is a shared library.
3. **Link.** The linker takes all object files and libraries, puts the needed pieces together, decides the final address of every function and static, and patches every place that refers to them. The result is an executable file.
4. **Load.** When you start the program, the operating system and the loader read that file, map it into memory, connect it to the shared libraries, and jump to its first instruction.

Cargo runs steps 1 to 3 for you on every `cargo build`. On macOS and on arm64 Linux, Rust calls the system's C compiler driver (`cc`), which calls the system linker (`ld`). (On x86-64 Linux, recent Rust versions ask `cc` to use `rust-lld` instead, a faster linker that is installed together with Rust.)

## 2. Object files and symbols

Ask `rustc` to stop after step 1 and write the object file of `src/main.rs`:

```bash
mkdir obj
rustc --edition 2024 -C opt-level=3 --emit obj -o obj/main.o src/main.rs
file obj/main.o
nm --demangle obj/main.o | grep -E ' (T|t|U|s) ' | grep -v -E 'l_anon|ltmp'
```

```text
obj/main.o: Mach-O 64-bit object arm64
0000000000000000 T std::rt::lang_start::<()>
0000000000000038 t std::sys::backtrace::__rust_begin_short_backtrace::<fn(), ()>
000000000000004c t std::rt::lang_start::<()>::{closure#0}
0000000000000068 t <std::rt::lang_start<()>::{closure#0} as core::ops::function::FnOnce<()>>::call_once::{shim:vtable#0}
0000000000000084 t main::add_numbers
000000000000008c T main::main
0000000000000228 s main::GREETING
                 U std::rt::lang_start_internal
                 U core::fmt::pointer_fmt_inner
                 U std::io::stdio::_print
0000000000000180 t <&str as core::fmt::Display>::fmt
                 U <u64 as core::fmt::Display>::fmt
                 U <str as core::fmt::Display>::fmt
0000000000000190 t <*const () as core::fmt::Pointer>::fmt
0000000000000198 t <&&str as core::fmt::Pointer>::fmt
00000000000001a0 t <&i32 as core::fmt::Pointer>::fmt
00000000000001a8 T _main
```

**What happens:**

1. `file` says what a file is: an object file (not an executable) for arm64 in the Mach-O format. Compiled alone, the crate is called `main`, after the file name.
2. `nm` lists the **symbol table**. Each line has an address (inside this object file, starting at 0), a letter for the kind of symbol, and the name. `--demangle` turns mangled names back into readable Rust paths. The two `grep` commands only hide compiler-internal labels.
3. `T` / `t`: code (in the "text" section) defined here; capital means visible to other files, small means local. `s`: data in some other section, here our `GREETING`.
4. `U`: **undefined**. This file calls `std::io::stdio::_print` but does not contain it. Filling these holes is the linker's job.
5. `_main` is the entry point that the C runtime calls; it is generated by `rustc` and calls `std::rt::lang_start`, which sets up the standard library and then calls our `main::main`.

Without `--demangle`, the names look like this (from `nm obj/main.o`):

```text
                 U __RNvNtCsi4LIHK2nfKK_3std2rt19lang_start_internal
                 U __RNvNtNtCsi4LIHK2nfKK_3std2io5stdio6__print
```

This is **name mangling**. Two crates may both have a function `add`, a generic function exists once per type, and the linker only understands flat names. So `rustc` encodes the full path into one unique name: `_R` marks Rust's mangling scheme (called v0), `Cs…` is a hash of the crate, and the numbers are lengths (`3std`, `2rt`, `19lang_start_internal`). macOS adds one more `_` in front of every C-level name. The hash part differs between builds and computers.

When a function must have a plain, predictable name, for example because C code calls it, switch mangling off. Create `obj/names.rs`:

```rust
#[unsafe(no_mangle)]
pub extern "C" fn course_add(a: u64, b: u64) -> u64 {
    a + b
}

#[inline(never)]
pub fn rust_add(a: u64, b: u64) -> u64 {
    a + b
}
```

```bash
rustc --edition 2024 --crate-type lib --emit obj -C opt-level=3 -o obj/names.o obj/names.rs
nm obj/names.o | grep -v ltmp
```

```text
0000000000000000 T __RNvCskb9wMNpQEXg_5names8rust_add
0000000000000008 T _course_add
```

**What happens:**

1. `#[unsafe(no_mangle)]` keeps the name exactly as written. The attribute is marked `unsafe` (edition 2024) because two items with the same plain name anywhere in the final program would clash, and the compiler cannot check that for you.
2. `extern "C"` makes the function use the C calling convention, so C code can call it (Lesson 06.10).
3. `rust_add` gets a mangled name. Without `#[inline(never)]` it would not appear at all: small public functions are often compiled only in the crate that calls them.

## 3. The linker: static and dynamic linking

Build the program normally and look at the result:

```bash
cargo build --release
./target/release/p12-linking
file target/release/p12-linking
otool -L target/release/p12-linking
```

```text
hello from a static: 42
main is at        0x100e2c914
add_numbers is at 0x100e2c90c
GREETING is at    0x100e70040
a local is at     0x16efd22e4
target/release/p12-linking: Mach-O 64-bit executable arm64
target/release/p12-linking:
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1359.40.3)
```

**What happens:**

1. The linker combined our object files with the parts of `std`, `core` and `alloc` that the program uses. These Rust libraries were linked **statically**: their code is now inside the executable (436,384 bytes here).
2. `otool -L` lists the shared libraries the executable needs at run time. There is only one: `libSystem`, the C library of macOS. It is linked **dynamically**: the executable contains only its name and a list of the symbols it needs from it.
3. You can see that list with `nm -u` (undefined symbols). Among its 70 lines are `_malloc`, `_free`, `_mmap` and `_write`: the standard library gets heap memory and writes to the terminal through the C library.

| | Static linking | Dynamic linking |
|---|---|---|
| Where the library code is | Copied into the executable | In a separate `.dylib` / `.so` file |
| File size | Bigger | Smaller |
| Needs the library at run time | No | Yes, the right version must be installed |
| Library updates (security fixes) | Rebuild your program | Update the library once, all programs benefit |
| Used by Rust for | Rust crates (`std`, your dependencies) | The system C library |

> ⚠️ **Common mistake:** A missing symbol shows up at **link** time, after compilation succeeded. If you declare a C function that no library provides, the error comes from the linker, not from `rustc`. Section "Common mistakes" shows the real message.

## 4. Inside a Mach-O executable (macOS)

An executable is not just machine code. It starts with a **header** and **load commands** that tell the operating system how to load it. `otool -h` prints the header, `otool -l` the load commands, and `size -m` the segments:

```bash
otool -h target/release/p12-linking
otool -l target/release/p12-linking | grep -A2 LC_LOAD_DYLINKER
size -m target/release/p12-linking | grep -E '^Segment'
```

```text
target/release/p12-linking:
Mach header
      magic  cputype cpusubtype  caps    filetype ncmds sizeofcmds      flags
 0xfeedfacf 16777228          0  0x00           2    17       2104 0x00a00085
          cmd LC_LOAD_DYLINKER
      cmdsize 32
         name /usr/lib/dyld (offset 12)
Segment __PAGEZERO: 4294967296 (zero fill) 
Segment __TEXT: 278528
Segment __DATA_CONST: 16384
Segment __DATA: 16384
Segment __LINKEDIT: 131072
```

**What happens:**

1. `magic 0xfeedfacf` marks a 64-bit Mach-O file. `cputype 16777228` is the number for arm64, and `filetype 2` means "executable" (an object file has type 1).
2. `LC_LOAD_DYLINKER` names the **loader** that the kernel must start for this program: `/usr/lib/dyld`.
3. `__TEXT` holds the machine code and read-only data, such as the bytes of the text "hello from a static". `__DATA_CONST` holds data that the loader fills in once and then makes read-only: `GREETING` itself lives here, because it is a pointer plus a length, and the pointer must be adjusted to the address the program is loaded at. `__DATA` holds data the program may change, and `__LINKEDIT` the symbol table and the information for the loader.
4. `__PAGEZERO` is 4 GiB (gibibytes) of address space that is mapped with no access at all. Any use of a small address, like a null pointer, crashes immediately instead of reading garbage.

## 5. Inside an ELF executable (Linux)

Linux uses ELF. Build and inspect the same program in a Linux container (in the official `rust:1.100` image, which has the binary tools installed). `CARGO_TARGET_DIR` keeps the Linux build apart from the macOS one:

```bash
docker run --rm -it -v "$PWD":/work -w /work -e CARGO_TARGET_DIR=/work/target-linux rust:1.100 bash
```

Inside the container:

```bash
cargo build -q --release
B=target-linux/release/p12-linking
file $B
readelf -d $B | grep NEEDED
readelf -l $B | grep interpreter
ldd $B
```

```text
target-linux/release/p12-linking: ELF 64-bit LSB pie executable, ARM aarch64, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-aarch64.so.1, BuildID[sha1]=67070b5c417c5727d84dc8a67139276832f71a2a, for GNU/Linux 3.7.0, not stripped
 0x0000000000000001 (NEEDED)             Shared library: [libgcc_s.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
      [Requesting program interpreter: /lib/ld-linux-aarch64.so.1]
	linux-vdso.so.1 (0x0000ffff8808b000)
	libgcc_s.so.1 => /lib/aarch64-linux-gnu/libgcc_s.so.1 (0x0000ffff87f90000)
	libc.so.6 => /lib/aarch64-linux-gnu/libc.so.6 (0x0000ffff87dd0000)
	/lib/ld-linux-aarch64.so.1 (0x0000ffff88040000)
```

**What happens:**

1. `file`: a 64-bit ELF **PIE** executable for AArch64 (the Linux name for arm64), dynamically linked. LSB (Least Significant Byte first) means little-endian (Lesson 12.1).
2. `readelf -d` shows the dynamic section. Two `NEEDED` entries: the C library `libc.so.6` and `libgcc_s.so.1`, a small helper library that Rust's standard library uses for unwinding the stack after a panic.
3. The **interpreter** is the loader, `/lib/ld-linux-aarch64.so.1`. On x86-64 Linux it is `/lib64/ld-linux-x86-64.so.2`.
4. `ldd` asks the loader to resolve everything and prints where each library was found. `linux-vdso.so.1` is a tiny library that the kernel itself puts into every process to make some system calls (Phase 13) faster; it has no file.

`readelf -h` prints the ELF header, and `objdump -d` disassembles code (Lesson 12.7). Still inside the container:

```bash
readelf -h $B | grep -E 'Class|Type|Machine|Entry'
nm -C $B | grep -E 'add_numbers|p12_linking::main'
objdump -d -C --no-show-raw-insn $B | grep -A3 '<p12_linking::add_numbers>:'
```

```text
  Class:                             ELF64
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           AArch64
  Entry point address:               0x8700
0000000000008874 t p12_linking::add_numbers
000000000000887c T p12_linking::main
0000000000008874 <p12_linking::add_numbers>:
    8874:	add	x0, x1, x0
    8878:	ret
```

The whole function is two arm64 instructions: add the two arguments into `x0` (the return register) and return. The **entry point** (0x8700) is not `main`: it is `_start` from the C runtime, which prepares the process and then calls `main`.

**A static Linux binary.** On Linux you can ask for the C library to be linked statically too, with a `rustc` codegen option. Still inside the container:

```bash
RUSTFLAGS='-C target-feature=+crt-static' CARGO_TARGET_DIR=target-static cargo build -q --release
file target-static/release/p12-linking
ldd target-static/release/p12-linking
ls -l target-linux/release/p12-linking target-static/release/p12-linking | awk '{print $5, $9}'
exit
```

```text
warning: linker stderr: /usr/bin/ld: … in function `std::sys::net::connection::socket::lookup_host::{closure#0}':
         … Using 'getaddrinfo' in statically linked applications requires at runtime the shared libraries from the glibc version used for linking
target-static/release/p12-linking: ELF 64-bit LSB pie executable, ARM aarch64, version 1 (GNU/Linux), static-pie linked, BuildID[sha1]=f3fc44f638e2af7c69362c71f1d522f5f4d76722, for GNU/Linux 3.7.0, not stripped
	statically linked
469448 target-linux/release/p12-linking
1258976 target-static/release/p12-linking
```

**What happens:**

1. `crt-static` (C runtime, static) links glibc, the C library of the GNU project (a large free-software project; GNU stands for "GNU's Not Unix"), into the program. The binary needs no shared libraries any more (`ldd`: "statically linked"), and it is almost three times bigger.
2. The linker warns (shortened here): a few glibc functions, such as looking up host names, still load shared libraries at run time even in a static binary. That is a known limit of glibc. For truly self-contained Linux binaries, Rust projects usually build for the `…-unknown-linux-musl` targets, which use the small musl C library instead of glibc.

> 💡 **Tip:** Run this kind of Linux experiment in a container or in your VM (virtual machine) `dev` from Phase 11. The macOS tools (`otool`) do not read ELF, and the Linux tools (`readelf`, `ldd`) are not installed on macOS.

## 6. Loading: what happens when the program starts

When you type `./target/release/p12-linking`, the kernel and the loader do this:

1. The kernel opens the file, reads the header, and sees which loader the program asks for (`dyld` or `ld.so`).
2. It maps the program's segments into a fresh address space: code as read-only and executable, data as read-write. It picks a **random base address** (ASLR).
3. It maps the loader too and starts it.
4. The loader reads the list of needed shared libraries, finds them, and maps them at random addresses as well.
5. It processes the **relocations**: every place that needs the address of `malloc` or `write` gets the real address in this run.
6. It jumps to the entry point. The C runtime starts, then Rust's `lang_start`, then your `main`.

Run the program twice on your Mac:

```text
main is at        0x100e2c914
add_numbers is at 0x100e2c90c
GREETING is at    0x100e70040
a local is at     0x16efd22e4
```

```text
main is at        0x100df0914
add_numbers is at 0x100df090c
GREETING is at    0x100e34040
a local is at     0x16f00e2e4
```

All addresses move between runs, but the **distances stay the same**: `main` is always 8 bytes after `add_numbers`, and the end digits `…914`, `…90c`, `…040` never change. The whole program slides as one block. The stack (`a local`) is placed at a separate random address. This works only because the code is position-independent: it finds its data relative to its own position instead of using fixed addresses.

> 🔒 **Security:** ASLR is a defence against attacks that need to know where code or data is in memory, such as jumping into existing code after a buffer overflow. It works only for PIE executables; never switch PIE off for a program that handles untrusted input. On Linux, `file` shows `pie executable` when it is on.

## Common mistakes

**1. Calling a function that no library provides.** This compiles but does not link:

```rust
unsafe extern "C" {
    fn answer_from_c() -> i32;
}

fn main() {
    // SAFETY: we promise that a C function with this signature exists.
    let answer = unsafe { answer_from_c() };
    println!("the answer is {answer}");
}
```

```text
error: linking with `cc` failed: exit status: 1
  |
  = note:  "cc" … (a very long command line, shortened here)
  = note: some arguments are omitted. use `--verbose` to show all linker arguments
  = note: Undefined symbols for architecture arm64:
            "_answer_from_c", referenced from:
                __RNvCsbDkgxTN66NU_11p12_linking4main in p12_linking.3naovqe6q21388ozee9hq60ud.1tu0v35.rcgu.o
          ld: symbol(s) not found for architecture arm64
          clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

Read it from the bottom: the linker (`ld`) did not find the symbol `_answer_from_c`, which is used by `p12_linking::main` (the mangled name). Fix it by providing the function: compile the C code with the `cc` crate (Lesson 06.10) or link the library that contains it. On Linux the same mistake says ``undefined reference to `answer_from_c'`` (GNU `ld`) or `undefined symbol: answer_from_c` (`rust-lld`).

**2. A shared library that the loader cannot find.** A program can link fine and still fail at start: `error while loading shared libraries: libgreet.so: cannot open shared object file`. The linker found the library in the build folder, but the loader looks only in the system folders and in the paths you give it. Exercise 4 shows this and the fix.

**3. Thinking `main` is the first code that runs.** The loader, the C runtime and Rust's `lang_start` all run before `main`. They set up the stack, the arguments, the standard streams and the panic handling.

## Try it yourself

1. Build `p12-linking` in release mode, run it three times, and check that the distance between `main` and `add_numbers` never changes.
2. Run `nm -u target/release/p12-linking | wc -l` and then `nm -u target/release/p12-linking | grep write`. Which C functions does even a "hello" program need?
3. Create `obj/main.o` as in section 2 and compare `nm obj/main.o` with `nm --demangle obj/main.o`.
4. Do section 5 in a container and compare the size of the dynamic and the static binary.
5. Clean up: `rm -rf obj target-linux target-static` and `cargo clean`.

## Exercises

### Exercise 1 — Read a symbol table (easy)

Here are three lines from `nm --demangle` of an object file. For each one, say whether the symbol is defined in this file, whether other files can use it, and what kind of thing it probably is.

```text
0000000000000084 t main::add_numbers
                 U std::io::stdio::_print
0000000000000228 s main::GREETING
```

<details>
<summary>Hint</summary>

Capital letter: visible to other files; small letter: local. `U` has no address because the code is not in this file.

</details>

<details>
<summary>Answer</summary>

1. `t main::add_numbers`: code defined in this file at offset 0x84, local (small `t`): a private function.
2. `U std::io::stdio::_print`: not defined here; the file uses it and the linker must find it (in the `std` rlib).
3. `s main::GREETING`: data defined in this file, local: our `static`.

</details>

### Exercise 2 — A C-callable function with tests (medium)

Add `src/lib.rs` to `p12-linking` with a function `course_checksum(data: *const u8, len: usize) -> u32` that C code can call: it adds up all bytes, wrapping on overflow, and returns 0 for `len == 0` without reading `data`. Write two tests (`[1, 2, 3, 250]` gives 256; a null pointer with length 0 gives 0), run them also under Miri, and check with `nm` that the symbol name is not mangled.

<details>
<summary>Hint</summary>

The function must be `pub unsafe extern "C" fn` with `#[unsafe(no_mangle)]`: the caller has to promise that the pointer is valid, so the function itself is `unsafe` to call. Inside, `std::slice::from_raw_parts(data, len)` turns the pointer into a slice. After `cargo build --release --lib`, run `nm target/release/libp12_linking.rlib`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Adds up all bytes of a buffer, wrapping on overflow. Callable from C.
///
/// # Safety
///
/// `data` must point to `len` readable bytes (or `len` must be 0).
#[unsafe(no_mangle)]
pub unsafe extern "C" fn course_checksum(data: *const u8, len: usize) -> u32 {
    if len == 0 {
        return 0;
    }
    // SAFETY: the caller promises that `data` points to `len` readable bytes.
    let bytes = unsafe { std::slice::from_raw_parts(data, len) };
    bytes.iter().fold(0u32, |sum, &b| sum.wrapping_add(u32::from(b)))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn sums_the_bytes() {
        let data = [1u8, 2, 3, 250];
        // SAFETY: `data` is a live array of exactly `data.len()` bytes.
        let sum = unsafe { course_checksum(data.as_ptr(), data.len()) };
        assert_eq!(sum, 256);
    }

    #[test]
    fn empty_input_gives_zero() {
        // SAFETY: with len 0 the pointer is never read.
        let sum = unsafe { course_checksum(std::ptr::null(), 0) };
        assert_eq!(sum, 0);
    }
}
```

```bash
cargo test -q --lib
cargo +nightly miri test -q --lib
cargo build -q --release --lib
nm target/release/libp12_linking.rlib 2>/dev/null | grep course
```

```text
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.24s
0000000000000000 T _course_checksum
```

Why it works: `from_raw_parts` with a null pointer is undefined behaviour (UB) even for length 0, so the early `return` matters, and Miri would catch it if it were missing. The `# Safety` section documents the promise the caller must keep (Lesson 06.6). An `.rlib` is an archive of object files, so `nm` can read it; `2>/dev/null` hides its notes about the metadata members, which have no symbols.

</details>

### Exercise 3 — Direct and indirect dependencies (medium)

Compare `otool -L /bin/ls` on your Mac with `ldd /bin/ls` and `readelf -d /bin/ls | grep NEEDED` in a `rust:1.100` container. Why does `ldd` list a library that `readelf -d` does not?

<details>
<summary>Hint</summary>

`readelf` only reads the file you give it. `ldd` asks the loader to load everything, including the libraries that the libraries need.

</details>

<details>
<summary>Answer</summary>

```text
/bin/ls:
	/usr/lib/libutil.dylib (compatibility version 1.0.0, current version 1.0.0)
	/usr/lib/libncurses.5.4.dylib (compatibility version 5.4.0, current version 5.4.0)
	/usr/lib/libSystem.B.dylib (compatibility version 1.0.0, current version 1359.40.3)
```

In the container:

```text
	linux-vdso.so.1 (0x0000ffff8522c000)
	libselinux.so.1 => /lib/aarch64-linux-gnu/libselinux.so.1 (0x0000ffff85140000)
	libcap.so.2 => /lib/aarch64-linux-gnu/libcap.so.2 (0x0000ffff85110000)
	libc.so.6 => /lib/aarch64-linux-gnu/libc.so.6 (0x0000ffff84f50000)
	/lib/ld-linux-aarch64.so.1 (0x0000ffff851f0000)
	libpcre2-8.so.0 => /lib/aarch64-linux-gnu/libpcre2-8.so.0 (0x0000ffff84e90000)
 0x0000000000000001 (NEEDED)             Shared library: [libselinux.so.1]
 0x0000000000000001 (NEEDED)             Shared library: [libcap.so.2]
 0x0000000000000001 (NEEDED)             Shared library: [libc.so.6]
 0x0000000000000001 (NEEDED)             Shared library: [ld-linux-aarch64.so.1]
```

Why it works: `ls` itself needs `libselinux`, `libcap` and `libc` (its `NEEDED` entries). `libselinux` in turn needs `libpcre2-8`, so the loader loads that too, and `ldd` shows the whole tree. `otool -L` on macOS, like `readelf -d`, shows only the direct dependencies.

</details>

### Exercise 4 — Your own shared library (hard)

In a container, build a Rust **shared library** with the function `course_add(a: u64, b: u64) -> u64`, and a small C program that calls it. Link them with `gcc`, then run the program: first as it is, then with `LD_LIBRARY_PATH` pointing to the library folder. Explain both results.

Use this C program (`main.c`, next to `Cargo.toml`):

```c
#include <stdint.h>
#include <stdio.h>

uint64_t course_add(uint64_t a, uint64_t b);

int main(void) {
    printf("2 + 40 = %llu\n", (unsigned long long)course_add(2, 40));
    return 0;
}
```

<details>
<summary>Hint</summary>

`cargo new --lib greet`, then add `[lib]` with `crate-type = ["cdylib"]` to `Cargo.toml`: a C-compatible dynamic library, `libgreet.so`. Link with `gcc main.c -o main -L <folder> -lgreet`: `-L` says where to look at link time, `-lgreet` means `libgreet.so`.

</details>

<details>
<summary>Answer</summary>

`Cargo.toml` of `greet` ends with:

```toml
[lib]
crate-type = ["cdylib"]
```

`src/lib.rs`:

```rust
/// Returns a + b. Callable from C as `uint64_t course_add(uint64_t, uint64_t)`.
#[unsafe(no_mangle)]
pub extern "C" fn course_add(a: u64, b: u64) -> u64 {
    a + b
}
```

Inside a `rust:1.100` container started in the `greet` folder (as in section 5):

```bash
cargo build -q --release
nm -D --defined-only target-linux/release/libgreet.so | grep course
gcc main.c -o main -L target-linux/release -lgreet
./main; echo "exit $?"
LD_LIBRARY_PATH=target-linux/release ldd ./main
LD_LIBRARY_PATH=target-linux/release ./main
```

```text
0000000000000648 T course_add
./main: error while loading shared libraries: libgreet.so: cannot open shared object file: No such file or directory
exit 127
	linux-vdso.so.1 (0x0000ffffb2d57000)
	libgreet.so => target-linux/release/libgreet.so (0x0000ffffb2cb0000)
	libc.so.6 => /lib/aarch64-linux-gnu/libc.so.6 (0x0000ffffb2af0000)
	libgcc_s.so.1 => /lib/aarch64-linux-gnu/libgcc_s.so.1 (0x0000ffffb2ab0000)
	/lib/ld-linux-aarch64.so.1 (0x0000ffffb2d10000)
2 + 40 = 42
```

Why it works: `nm -D` shows the **dynamic** symbol table, the names a shared library offers to others; `course_add` is there with its plain name. At link time `-L` told the linker where `libgreet.so` is, so linking worked, but the executable stores only the name `libgreet.so`. At start, the loader searches the system folders, does not find it, and stops with exit code 127. `LD_LIBRARY_PATH` adds a folder to the loader's search. Real programs install the library into a system folder or store a search path in the executable instead.

</details>

## Quiz

1. What is inside an object file, and why can you not run it?

<details>
<summary>Answer</summary>

Machine code and data for one piece of the program, plus a symbol table of the names it defines and the names it still needs (`U`), and relocations for every place that needs a final address. It cannot run because those missing pieces and final addresses are only filled in by the linker.

</details>

2. Why does `rustc` mangle names, and how do you turn it off for one function?

<details>
<summary>Answer</summary>

The linker only knows flat names, but Rust has modules, crates with equal function names, and generic functions that exist once per type. Mangling encodes the full path into one unique name. `#[unsafe(no_mangle)]` keeps a function's plain name, usually together with `extern "C"`.

</details>

3. Which libraries does a normal Rust program on macOS link statically, and which dynamically?

<details>
<summary>Answer</summary>

Statically: all Rust crates, including `std`, `core`, `alloc` and your dependencies. Dynamically: the system C library, `libSystem` (on Linux: `libc.so.6` and `libgcc_s.so.1`).

</details>

4. Your build fails with "Undefined symbols … referenced from …". Is it a compiler error or a linker error, and what does it mean?

<details>
<summary>Answer</summary>

A linker error. The code compiled, but some symbol that the code uses is not defined in any object file or library given to the linker. Provide the code (for example compile the C file) or link the right library.

</details>

5. What do `dyld` and `ld.so` do?

<details>
<summary>Answer</summary>

They are the loaders (dynamic linkers) of macOS and Linux. They run first when a dynamically linked program starts: they map the needed shared libraries into memory, fill in the addresses of their symbols (relocations), and then jump to the program's entry point.

</details>

6. Why does `main` have a different address on every run, and why does the program still work?

<details>
<summary>Answer</summary>

ASLR loads the program at a random base address each time. The program is position-independent (PIE): its code reaches its own functions and data relative to its own position, so it works at any address, and the distances between its parts stay the same.

</details>

## Summary

- Source becomes a process in four steps: `rustc` compiles to **object files**, the **linker** combines them with libraries into an executable, and the **loader** maps it into memory and connects shared libraries.
- **Symbols** are the names of functions and statics; `nm` lists them (`T` defined, `U` needed). Rust **mangles** names; `nm --demangle` makes them readable, and `#[unsafe(no_mangle)]` with `extern "C"` gives a plain name for C.
- Rust crates are linked **statically**; the system C library **dynamically**. `otool -L` (macOS) and `readelf -d` / `ldd` (Linux) show the shared libraries.
- macOS uses **Mach-O** (header, load commands, segments like `__TEXT`, loader `/usr/lib/dyld`); Linux uses **ELF** (`readelf -h`, `NEEDED`, interpreter `ld-linux…so`).
- `-C target-feature=+crt-static` makes a static glibc binary on Linux, with limits; musl targets are the usual way to fully static binaries.
- Missing symbols are **link** errors; missing shared libraries are **load** errors at start-up.
- **ASLR** and **PIE** move the whole program to a random address on every run; distances inside it stay the same.

## Sources and further reading

- [The Rust Reference — Linkage](https://doc.rust-lang.org/reference/linkage.html) — crate types (`bin`, `rlib`, `cdylib`, …), static and dynamic C runtimes.
- [The rustc book — Symbol mangling](https://doc.rust-lang.org/rustc/symbol-mangling/index.html) — how Rust builds symbol names, including the v0 scheme.
- [The Rust Reference — Application binary interface](https://doc.rust-lang.org/reference/abi.html) — `no_mangle`, `extern` and related attributes.
- [The rustc book — Codegen options](https://doc.rust-lang.org/rustc/codegen-options/index.html) — `target-feature`, `link-arg` and other linker-related options.
- [The Cargo Book — Cargo targets](https://doc.rust-lang.org/cargo/reference/cargo-targets.html) — where `crate-type` is configured.
- [elf(5)](https://man7.org/linux/man-pages/man5/elf.5.html), [ld.so(8)](https://man7.org/linux/man-pages/man8/ld.so.8.html) and [ldd(1)](https://man7.org/linux/man-pages/man1/ldd.1.html) — the ELF format and the Linux loader, from the Linux manual pages.
- [David Drysdale — Beginner's Guide to Linkers](https://www.lurklurk.org/linkers/linkers.html) — a gentle, longer introduction to symbols, linkers and shared libraries.
- [Computer Systems: A Programmer's Perspective (CS:APP)](https://csapp.cs.cmu.edu/) — chapter 7 is the classic explanation of linking.

---

[🏠 Start](../START-HERE.md) · [← 12.11 Memory layout of Rust types](11-memory-layout-of-rust-types.md) · [📋 Phase 12](README.md) · [12.13 How memory allocators work →](13-how-memory-allocators-work.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
