<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.1 Choosing a specialization](01-choosing-a-specialization.md) · [📋 Phase 22](README.md) · [22.3 OS track: booting a kernel in QEMU →](03-os-track-booting-a-kernel-in-qemu.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/02-os-track-a-freestanding-binary.md) <!-- nav -->

# Lesson 22.2 — OS track: a freestanding Rust binary

> **Phase 22 — Specializations** · Lesson 2 of 30 · ⏱ about 2.5 hours (2 sittings)

> **Not your track?** This lesson starts the OS and kernel track (Lessons 22.2–22.5). If you chose another
> track, skip to [Lesson 22.6 — eBPF track: what eBPF is](06-ebpf-track-what-ebpf-is.md) or straight to
> [Lesson 22.30 — Checkpoint: your specialization](30-checkpoint-your-specialization.md).

## What you will learn

- What kernel work is, why Rust is used for it, and which machine this track uses (and why).
- What a freestanding binary is, and how `std`, `alloc` and `core` differ.
- How to turn a normal Rust program into one with no operating system under it: `#![no_std]`, a panic
  handler, `#![no_main]` and your own entry point.
- How to build for a "no OS" target and look inside the resulting ELF file.
- How to test the pure parts of kernel code on your Mac.

## Before you start

- You finished [Lesson 22.1 — Choosing a specialization](01-choosing-a-specialization.md) and chose this track.
- You know what a kernel does ([Lesson 13.1](../13-operating-systems/01-what-an-operating-system-does.md)),
  how a program is linked and loaded ([Lesson 12.12](../12-computer-systems/12-linking-and-loading.md)) and
  you have read arm64 assembly ([Lesson 12.7](../12-computer-systems/07-reading-assembly.md)).
- You know `unsafe` ([Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)); the next lessons use it a lot.
- Docker Desktop works on your Mac (Phase 11); you need it from Lesson 22.3 on.

## New words

| Word | Meaning |
|---|---|
| **freestanding binary** | A program that does not need an operating system: it brings everything it uses. A kernel is one. |
| **bare metal** | Running directly on the hardware, with no operating system below your code. |
| **`std`** | The Rust standard library. It contains files, threads, `println!` and more, and it needs an OS. |
| **`core`** | The part of the standard library that needs nothing from an OS: `Option`, `Result`, iterators, `fmt`, slices. |
| **`alloc`** | The part with heap types (`Box`, `Vec`, `String`). It needs a memory allocator but no OS. |
| **`#![no_std]`** | A crate attribute: "link only `core`, not `std`". |
| **panic handler** | The function that runs when code panics. With `std` it is ready-made; without `std` you must write it. |
| **entry point** | The address where the CPU (Central Processing Unit) starts to run a program. |
| **target triple** | The name of the platform you compile for, such as `aarch64-apple-darwin`: CPU, vendor, operating system. |
| **cross-compiling** | Building on one platform (your Mac) a program for another platform (a bare arm64 machine). |
| **ELF** (Executable and Linkable Format) | The file format of programs on Linux and of most kernels (Lesson 12.12). |
| **softfloat** | A target variant that never uses the floating-point and SIMD (Single Instruction, Multiple Data) registers. |
| **emulator** | A program that pretends to be another computer. QEMU (Quick Emulator) is the one this track uses. |
| **llvm-tools** | A rustup component with LLVM (the compiler toolkit behind `rustc`) tools to look inside binaries. |

**Other abbreviations in this lesson:** OS (operating system: the program that manages the hardware and runs other
programs, Phase 13) and PC (personal computer: here, an ordinary x86-64 desktop or laptop).

## 1. The OS track, and the machine we use

Kernel engineers write the code that runs *under* every other program: the code that starts the machine,
drives devices, handles interrupts and manages memory. Rust fits this work well: it has no garbage
collector and no runtime that needs an OS, it gives exact control over memory layout, and its checks remove
whole classes of memory bugs from most of the code. `unsafe` stays in small, reviewed places. Today Rust is
used in the Linux kernel, in Android, in Windows components and in research kernels.

In the four lessons of this track you build a small kernel step by step:

| Lesson | What your kernel can do afterwards |
|---|---|
| 22.2 (this one) | It exists: a freestanding binary that builds for a machine without an OS. |
| 22.3 | It boots in QEMU and prints text over a serial line. |
| 22.4 | It handles exceptions, timer interrupts and uses its own page tables. |
| 22.5 | Project: a small interactive kernel, plus where to go next. |

**Which machine?** A kernel is written for one kind of machine. This track uses the **`virt` machine of
QEMU with an arm64 CPU** (arm64 is also called AArch64). The reasons:

- Your Mac has an arm64 CPU. The instructions and registers are the ones you read in Lesson 12.7.
- `virt` is a simple, well-documented machine made for emulators. QEMU loads our ELF file directly into
  memory and jumps to its entry point, so we need **no bootloader and no disk image**.
- Only stable Rust is needed. The target `aarch64-unknown-none-softfloat` is ready-made by rustup.

The other common choice is an x86-64 PC with the `bootloader` crate (version 0.11), which the popular
*Writing an OS in Rust* blog uses. It is a fine path too; Lesson 22.5 lists it as a next step.

**Code layout.** The whole track works in one package, `code/p22-kernel`. Every lesson changes it a bit, and
the project in Lesson 22.5 continues it. Commit after each lesson.

## 2. What `std` needs, and what `core` gives you

`println!` works in a normal program because `std` asks the operating system to write to standard output.
Files, threads, time, the network and even the start of `main` are the same: `std` is a layer on top of the
OS.

```text
   your program
        |
   std  (files, threads, println!, process start)   <- needs an operating system
        |
   alloc (Box, Vec, String)                        <- needs an allocator
        |
   core  (Option, Result, iterators, fmt, slices)   <- needs nothing
        |
   hardware
```

A kernel *is* the operating system, so nobody is below it to answer `std`. It can only use `core` (and later
`alloc`, once it has its own allocator — you built one in Lesson 12.14). Everything else you write yourself.
That is what this lesson does, one compiler error at a time.

## 3. Step 1: `#![no_std]`

Create the package on your Mac:

```bash
cd ~/projects/rust
cargo new code/p22-kernel
cd code/p22-kernel
```

Replace `src/main.rs` with a normal program that only adds one line at the top:

```rust
#![no_std]

fn main() {
    println!("Hello, world!");
}
```

`#![no_std]` (with `!`, so it applies to the whole crate) tells the compiler: do not link `std`, only
`core`. Build it with `cargo build`:

```text
error: cannot find macro `println` in this scope
 --> src/main.rs:4:5
  |
4 |     println!("Hello, world!");
  |     ^^^^^^^

error: `#[panic_handler]` function required, but not found

error: unwinding panics are not supported without std
  |
  = help: using nightly cargo, use -Zbuild-std with panic="abort" to avoid unwinding
  = note: since the core library is usually precompiled with panic="unwind", rebuilding your crate with panic="abort" may not be enough to fix the problem

error: could not compile `p22-kernel` (bin "p22-kernel") due to 3 previous errors
```

**What happens:**

1. `println!` lives in `std`, so it is gone. Printing needs a device to print to; Lesson 22.3 writes a driver.
2. A panic must end *somewhere*. `std` normally prints the message and stops the thread. Without `std` the
   compiler asks you for a function marked `#[panic_handler]`.
3. By default a panic **unwinds**: it walks back up the stack and runs `drop` for every value. Unwinding
   needs support code from `std`. A kernel simply stops instead, which is called **abort**.

## 4. Step 2: a panic handler, and `panic = "abort"`

Add the handler and remove `println!`:

```rust
#![no_std]

use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

fn main() {}
```

Then add these lines at the end of `Cargo.toml`, so that both build profiles abort on panic:

```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

**What happens:**

1. `#[panic_handler]` marks the one function in the whole program that a panic calls.
2. It receives `&PanicInfo` (the message and the source location) and returns `!`, the "never" type: this
   function never returns. For now it loops forever.
3. `panic = "abort"` removes unwinding, so the third error disappears.

`cargo build` now gives a new error:

```text
error: using `fn main` requires the standard library
  |
  = help: use `#![no_main]` to bypass the Rust generated entrypoint and declare a platform specific entrypoint yourself, usually with `#[no_mangle]`
```

`main` is not really the first function of a program. On macOS and Linux, start-up code from the C library
and `std` runs first: it prepares the arguments and the environment, then calls your `main`. Without `std`,
that code does not exist.

## 5. Step 3: `#![no_main]` and your own entry point

Replace `src/main.rs` again:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;

#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
    loop {}
}

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

**What happens:**

1. `#![no_main]` says: there is no `main` and no generated start-up code.
2. `_start` is the conventional name of an entry point. The linker looks for a symbol with exactly this name.
3. `#[unsafe(no_mangle)]` keeps the name `_start` in the binary. Normally the compiler **mangles** names into
   long unique strings (you will see one in Section 7); a mangled name could not be found by the linker. The
   attribute is `unsafe` because two functions with the same unmangled name would clash.
4. `extern "C"` uses the C calling convention (Lesson 06.10), the simple, stable way that machine code calls
   functions.
5. `-> !`: there is nobody to return to, so the function must never return.

`cargo build` still fails, but now in the **linker**:

```text
error: linking with `cc` failed: exit status: 1
  |
  = note: Undefined symbols for architecture arm64:
            "_main", referenced from:
                <initial-undefines>
          ld: symbol(s) not found for architecture arm64
          clang: error: linker command failed with exit code 1 (use -v to see invocation)
```

The compiler did its job. But Cargo builds for your Mac by default (the target `aarch64-apple-darwin`), and
the macOS linker insists on a program with `main` that runs under macOS. We do not want a macOS program.

## 6. Step 4: build for a machine with no OS

A **target triple** names the platform you build for. Read `aarch64-unknown-none-softfloat` from left to
right:

| Part | Meaning |
|---|---|
| `aarch64` | The CPU: 64-bit Arm, the same family as your Mac's CPU. |
| `unknown` | The vendor: nobody in particular. |
| `none` | The operating system: **none**. This is what makes it a bare-metal target. |
| `softfloat` | Never use the floating-point and SIMD registers. |

Why `softfloat`? When an interrupt stops the kernel, the kernel must save every register that its handler
might change (Lesson 22.4). If the kernel never touches the 32 large floating-point registers, it never has to
save them. Most kernels follow this rule; floating point is for user programs.

Rust ships a ready-made `core` for this target, so you only add it once (this is your normal rustup; the
command changes nothing else):

```bash
rustup target add aarch64-unknown-none-softfloat
rustup target list --installed
```

Example output (yours may list more targets):

```text
aarch64-apple-darwin
aarch64-unknown-none-softfloat
```

Now build for it:

```bash
cargo build --target aarch64-unknown-none-softfloat
file target/aarch64-unknown-none-softfloat/debug/p22-kernel
```

```text
   Compiling p22-kernel v0.1.0 (/Users/student/projects/rust/code/p22-kernel)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.23s
target/aarch64-unknown-none-softfloat/debug/p22-kernel: ELF 64-bit LSB executable, ARM aarch64, version 1 (SYSV), statically linked, with debug_info, not stripped
```

This is cross-compiling: your Mac produced an ELF file for an arm64 machine without an OS. The linker is
`rust-lld`, which comes with Rust, so nothing else is needed. You cannot run this file on your Mac; Lesson
22.3 boots it in QEMU.

Typing `--target` every time is tiring. Create `.cargo/config.toml` in the package (a config file that
Cargo reads for every command in this folder):

```toml
[build]
target = "aarch64-unknown-none-softfloat"
```

From now on `cargo build` builds the kernel.

> 💡 **Tip:** RustRover reads `.cargo/config.toml` too, so code completion and errors in the editor now use the
> bare-metal target. If the editor still shows `std` items, reload the Cargo project.

## 7. Look inside the binary

The `llvm-tools` component contains tools that understand ELF files on any computer. Install it and remember
where the tools are:

```bash
rustup component add llvm-tools
LLVM_BIN="$(rustc --print sysroot)/lib/rustlib/aarch64-apple-darwin/bin"
ls "$LLVM_BIN"
```

`rustc --print sysroot` prints the folder of the active toolchain; the tools are in a subfolder. Among the
files you will see `llvm-objdump`, `llvm-readobj`, `llvm-nm` and `llvm-size`.

Before you look, give `_start` something to do. Pure helper code goes into a **library** in the same package,
so that it can be tested later. Create `src/lib.rs`:

```rust
//! Code that does not touch hardware. It builds for the kernel
//! and can be tested on your Mac.
#![cfg_attr(not(test), no_std)]

/// Rounds `addr` up to the next multiple of `align` (a power of two).
pub fn align_up(addr: usize, align: usize) -> usize {
    assert!(align.is_power_of_two(), "align must be a power of two");
    (addr + align - 1) & !(align - 1)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn rounds_up_to_the_next_page() {
        assert_eq!(align_up(0x4008_0001, 4096), 0x4008_1000);
    }

    #[test]
    fn keeps_aligned_addresses() {
        assert_eq!(align_up(0x4008_0000, 4096), 0x4008_0000);
        assert_eq!(align_up(0, 16), 0);
    }
}
```

And replace `src/main.rs`:

```rust
#![no_std]
#![no_main]

use core::panic::PanicInfo;
use p22_kernel::align_up;

/// The first Rust code of the kernel. Nobody calls it from Rust: the machine jumps here.
#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
    let _first_page = align_up(0x4008_0001, 4096);
    halt()
}

/// Called by `panic!` and by failed checks such as an index out of bounds.
#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    halt()
}

/// Stops here forever, telling the CPU that we are only waiting.
fn halt() -> ! {
    loop {
        core::hint::spin_loop();
    }
}
```

**What happens:**

1. `align_up` rounds an address up to a boundary; kernels need this all the time (pages are 4096 bytes).
   Because `align` is a power of two, `align - 1` is a mask of low bits: `+ (align - 1)` pushes the address
   over the next boundary, and `& !(align - 1)` clears the low bits (Lesson 12.4).
2. `#![cfg_attr(not(test), no_std)]` means: "be `no_std`, except when compiled for tests". Tests need the
   test harness from `std`; the kernel does not get `std`.
3. `main.rs` uses the library as `p22_kernel::align_up` (the dash in the package name becomes `_`).
4. `halt` replaces the empty `loop {}`. `core::hint::spin_loop()` emits an instruction that tells the CPU
   "I am only waiting"; Clippy warns about an empty loop.

Build in release mode and look at the machine code. `-d` disassembles, `-C` turns mangled names back into
readable Rust names:

```bash
cargo build --release
"$LLVM_BIN/llvm-objdump" -d -C target/aarch64-unknown-none-softfloat/release/p22-kernel
```

```text
Disassembly of section .text:

0000000000210244 <__rustc::rust_begin_unwind>:
  210244: d5033fdf     	isb
  210248: 17ffffff     	b	0x210244 <__rustc::rust_begin_unwind>

000000000021024c <_start>:
  21024c: f81f0ffe     	str	x30, [sp, #-0x10]!
  210250: 52800020     	mov	w0, #0x1                // =1
  210254: 52820001     	mov	w1, #0x1000             // =4096
  210258: 72a80100     	movk	w0, #0x4008, lsl #16
  21025c: 94000003     	bl	0x210268 <p22_kernel::align_up>
  210260: d5033fdf     	isb
  210264: 17ffffff     	b	0x210260 <_start+0x14>

0000000000210268 <p22_kernel::align_up>:
  210268: d1000428     	sub	x8, x1, #0x1
  21026c: ca080029     	eor	x9, x1, x8
  210270: eb08013f     	cmp	x9, x8
  210274: 540000c9     	b.ls	0x21028c <p22_kernel::align_up+0x24>
  ...
```

Read it with what you know from Lesson 12.7:

1. Our panic handler got the internal name `rust_begin_unwind`. It is the `isb` + `b` loop: `spin_loop()`
   became the `isb` instruction.
2. `_start` puts `0x4008_0001` into `w0` (in two halves, `mov` and `movk`), `4096` into `w1`, and calls
   `align_up` with `bl`. Then it loops forever.
3. `align_up` checks "power of two" first; if the check fails it jumps to code that calls the panic machinery,
   which ends in our handler. Even `assert!` works without `std`.
4. Look at the very first instruction of `_start`: `str x30, [sp, #-0x10]!` stores a register **on the stack**.
   But nobody has set up a stack! On a real machine `sp` holds a random value here. Lesson 22.3 fixes this
   with a few assembly instructions that run before any Rust code.

The ELF header tells where execution starts:

```bash
"$LLVM_BIN/llvm-readobj" --file-headers target/aarch64-unknown-none-softfloat/release/p22-kernel
```

```text
  Type: Executable (0x2)
  Machine: EM_AARCH64 (0xB7)
  Version: 1
  Entry: 0x21024C
```

`Entry` is exactly the address of `_start` in the disassembly. The address `0x21024C` was chosen by the linker
on its own. On the QEMU `virt` machine, memory starts at `0x4000_0000`, so this address would be useless
there; in Lesson 22.3 a **linker script** decides where everything goes.

## 8. Test the pure parts on your Mac

The kernel target has no `std`, so the test harness cannot run there. But `src/lib.rs` builds for any target,
so you can run its tests on your Mac by naming the Mac target:

```bash
cargo test --lib --target aarch64-apple-darwin
```

```text
running 2 tests
test tests::rounds_up_to_the_next_page ... ok
test tests::keeps_aligned_addresses ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`--lib` builds only the library (the kernel binary cannot be a test), and `--target` overrides the target from
`.cargo/config.toml`. Make it short with a Cargo **alias**: replace `.cargo/config.toml` with

```toml
[build]
target = "aarch64-unknown-none-softfloat"

[alias]
# Runs the tests of src/lib.rs on the Mac instead of the kernel target.
host-test = "test --lib --target aarch64-apple-darwin"
```

From now on `cargo host-test` runs the tests, and `cargo build` builds the kernel.

> 🧪 **Testing:** This is the main testing rule for the whole track: hardware code stays small, and every
> calculation (addresses, bit fields, parsing) lives in `src/lib.rs` with ordinary unit tests. Testing inside
> QEMU is possible too; Lesson 22.5 mentions how.

## Common mistakes

> ⚠️ **Common mistake:** Forgetting `#[unsafe(no_mangle)]` on `_start`. The build only warns, and the result is
> useless:
>
> ```text
> warning: linker stderr: rust-lld: cannot find entry symbol _start; not setting start address
> ```
>
> The function got a mangled name, the linker found no `_start`, set the entry point to `0x0` and removed the
> "unused" code. Always read linker warnings in kernel work.

> ⚠️ **Common mistake:** Running plain `cargo test` in the kernel package. It builds the tests for the kernel
> target, where the test harness does not exist:
>
> ```text
> error[E0463]: can't find crate for `std`
> error[E0463]: can't find crate for `test`
> ```
>
> Use `cargo host-test` (Section 8).

> ⚠️ **Common mistake:** Writing `std::` in kernel code, for example `std::ptr::write_volatile`. Most such items
> live in `core` too: write `core::ptr::write_volatile`.

> ⚠️ **Common mistake:** Using the target `aarch64-unknown-none` (without `softfloat`) and then floating-point
> code. It builds, but the first floating-point instruction traps on the real machine until the kernel
> enables the floating-point unit. This track avoids the problem with the `softfloat` target.

## Try it yourself

1. Follow Sections 3 to 6 exactly and read every error before you fix it.
2. Run `cargo host-test` and `cargo clippy`; both must be clean.
3. Commit: `git add code/p22-kernel && git commit -m "p22-kernel: freestanding binary"`.

## Exercises

### Exercise 1 — How big is a kernel? (easy)

Run `"$LLVM_BIN/llvm-size"` on the debug and the release build. How many bytes of code (`text`) does each have? Then compare
with the size of the files on disk (`ls -l`). Why are the files so much bigger?

<details>
<summary>Hint</summary>

The `text` column is machine code and constant data. Debug builds also contain debug information, which is
not loaded into memory.

</details>

<details>
<summary>Answer</summary>

```text
   text	   data	    bss	    dec	    hex	filename
    917	      0	      0	    917	    395	target/aarch64-unknown-none-softfloat/debug/p22-kernel
    372	      0	      0	    372	    174	target/aarch64-unknown-none-softfloat/release/p22-kernel
```

`ls -l` shows about 713 KB for the debug file and about 8 KB for the release file. The extra bytes are debug
information and the symbol table: useful for debuggers, but never loaded into the machine's memory. The kernel
itself is a few hundred bytes. (Your numbers may differ a little.)

</details>

### Exercise 2 — `is_aligned`, with a test (medium)

Add `pub fn is_aligned(addr: usize, align: usize) -> bool` to `src/lib.rs` and a test
`checks_alignment` with at least three checks. Run it with `cargo host-test`.

<details>
<summary>Hint</summary>

An address is aligned when its low bits (the bits inside `align - 1`) are all zero.

</details>

<details>
<summary>Answer</summary>

```rust
/// Is `addr` a multiple of `align` (a power of two)?
pub fn is_aligned(addr: usize, align: usize) -> bool {
    addr & (align - 1) == 0
}
```

And inside `mod tests`:

```rust
    #[test]
    fn checks_alignment() {
        assert!(is_aligned(0x4008_0000, 4096));
        assert!(!is_aligned(0x4008_0008, 4096));
        assert!(is_aligned(0x4008_0008, 8));
    }
```

Why it works: `0x4008_0008 & 0xfff` is `8`, not zero, so the address is not page-aligned, but `8 & 7` is `0`,
so it is 8-byte aligned.

</details>

### Exercise 3 — `align_down`, tested on ten thousand addresses (hard)

Add `align_down` (round down to a multiple of `align`, with the same power-of-two check). Then write two tests:
one that loops over the addresses `0..10_000` and checks that `align_down(a, 64) <= a <= align_up(a, 64)`, that
both results are aligned, and that they are either equal or 64 apart; and one `#[should_panic]` test for
`align = 3`.

<details>
<summary>Hint</summary>

Rounding down only clears the low bits: `addr & !(align - 1)`. `#[should_panic(expected = "power of two")]`
checks that the panic message contains this text (Lesson 05.6).

</details>

<details>
<summary>Answer</summary>

```rust
/// Rounds `addr` down to a multiple of `align` (a power of two).
pub fn align_down(addr: usize, align: usize) -> usize {
    assert!(align.is_power_of_two(), "align must be a power of two");
    addr & !(align - 1)
}
```

```rust
    #[test]
    fn align_down_and_up_surround_every_address() {
        for addr in 0..10_000 {
            let down = align_down(addr, 64);
            let up = align_up(addr, 64);
            assert!(down <= addr && addr <= up, "addr {addr}");
            assert!(is_aligned(down, 64) && is_aligned(up, 64));
            assert!(up - down == 0 || up - down == 64);
        }
    }

    #[test]
    #[should_panic(expected = "power of two")]
    fn rejects_an_alignment_that_is_not_a_power_of_two() {
        align_down(100, 3);
    }
```

```text
test tests::rejects_an_alignment_that_is_not_a_power_of_two - should panic ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: a test that checks a *property* over many inputs finds mistakes that two or three hand-picked
examples miss (Lesson 09.4 does this with a library). It uses `is_aligned` from Exercise 2.

</details>

## Quiz

1. What is the difference between `core`, `alloc` and `std`?

<details>
<summary>Answer</summary>

`core` needs nothing (types, traits, iterators, formatting). `alloc` adds heap types and needs a memory
allocator. `std` adds files, threads, printing and process start and needs an operating system. A kernel can use
`core` always and `alloc` once it has an allocator.

</details>

2. Why does a `#![no_std]` binary need a `#[panic_handler]`?

<details>
<summary>Answer</summary>

Every panic must end in some function. `std` normally provides it (print and stop the thread). Without `std`
there is no such function, so you write one; it must never return (`-> !`).

</details>

3. Why do we write `#![no_main]` and `_start` instead of `fn main`?

<details>
<summary>Answer</summary>

`main` is called by start-up code from the C library and `std`, which needs an operating system. A kernel has
no such code; the machine jumps straight to the entry point, which we provide as `_start`.

</details>

4. What does `#[unsafe(no_mangle)]` do, and what happens without it on `_start`?

<details>
<summary>Answer</summary>

It keeps the exact symbol name `_start` instead of a mangled one. Without it the linker finds no `_start`, sets
the entry point to `0x0`, and removes the code, which it now thinks is unused.

</details>

5. The disassembly of `_start` begins with `str x30, [sp, #-0x10]!`. Why is that a problem on real hardware?

<details>
<summary>Answer</summary>

It writes to the stack, but at the entry point nobody has set the stack pointer `sp`, so the write goes to a
random address. The kernel must set up a stack in assembly before any Rust code runs (Lesson 22.3).

</details>

## Summary

- A kernel is a freestanding binary: no operating system is below it, so it uses only `core` (and later `alloc`).
- `#![no_std]` drops `std`; then you need a `#[panic_handler]` and `panic = "abort"`.
- `#![no_main]` drops the start-up code; `#[unsafe(no_mangle)] pub extern "C" fn _start() -> !` is the entry point.
- The target `aarch64-unknown-none-softfloat` means arm64, no OS, no floating-point registers; set it in
  `.cargo/config.toml`.
- `llvm-objdump -d -C` and `llvm-readobj --file-headers` from `llvm-tools` show the code and the entry point.
- Pure logic lives in `src/lib.rs` with `#![cfg_attr(not(test), no_std)]` and is tested on the Mac with
  `cargo host-test`.

## Sources and further reading

- [The Embedded Rust Book — A `no_std` Rust Environment](https://docs.rust-embedded.org/book/intro/no-std.html) — what
  `core`, `alloc` and `std` provide.
- [The Rust Reference — the `panic_handler` attribute](https://doc.rust-lang.org/reference/panic.html) — the exact rules
  for panic handlers.
- [rustc book — aarch64-unknown-none](https://doc.rust-lang.org/rustc/platform-support/aarch64-unknown-none.html) — the
  official page of the bare-metal arm64 targets.
- [The `core` library documentation](https://doc.rust-lang.org/core/) — everything a kernel can use without `std`.
- [Writing an OS in Rust — A Freestanding Rust Binary](https://os.phil-opp.com/freestanding-rust-binary/) — the same
  first step for x86-64, in a well-known blog series.

---

[🏠 Start](../START-HERE.md) · [← 22.1 Choosing a specialization](01-choosing-a-specialization.md) · [📋 Phase 22](README.md) · [22.3 OS track: booting a kernel in QEMU →](03-os-track-booting-a-kernel-in-qemu.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
