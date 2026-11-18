<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 12.6 How the CPU runs instructions](06-how-the-cpu-runs-instructions.md) · [📋 Phase 12](README.md) · [12.8 From Rust to machine code →](08-from-rust-to-machine-code.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/07-reading-assembly.md) <!-- nav -->

# Lesson 12.7 — Reading assembly: arm64 and x86-64

> **Phase 12 — Computer systems** · Lesson 7 of 15 · ⏱ about 135 minutes

## What you will learn

- How to see the assembly that the Rust compiler produces for your functions, on your Mac and for x86-64 Linux.
- How to read an assembly line: labels, instructions, registers, immediate values and memory operands.
- The calling conventions: in which registers arguments arrive and where the result goes, on arm64 and on x86-64.
- How comparisons, branches, loops, function calls and the stack look in assembly.

## Before you start

- You finished [Lesson 12.6 — How the CPU runs instructions](06-how-the-cpu-runs-instructions.md).
- You saw your first assembly (`add w0, w1, w0` and `ret`) in [Lesson 00.3 — How a computer runs a program](../00-getting-started/03-how-a-computer-runs-a-program.md), and `extern "C"` and `#[unsafe(no_mangle)]` in [Lesson 06.10 — FFI: calling C code](../06-rust-advanced/10-ffi-calling-c.md).
- Create a **library** package for Lessons 12.7 and 12.8: `cargo new --lib code/p12-asm`. A library is enough, because we only want to look at the compiled functions, never run them.

You do not need to **write** assembly. You need to **read** short pieces of it, to check what the compiler did with your code. That is a skill every systems engineer uses when a hot loop is slow or a crash happens inside optimised code.

## New words

| Word | Meaning |
|---|---|
| **assembly language** | The human-readable text form of machine code: one line per instruction, with names instead of numbers. |
| **mnemonic** | The short name of an instruction, such as `add`, `ldr` or `ret`. |
| **operand** | An input or output of an instruction: a register, a constant, or a memory address. |
| **immediate** | A constant written directly inside an instruction, like `#3` on arm64 or `8` on x86-64. |
| **label** | A name for a position in the code, ending with `:`. Branches jump to labels. |
| **directive** | A line starting with `.` (like `.globl` or `.cfi_startproc`). It is an instruction for the assembler program, not for the CPU. |
| **ABI** (Application Binary Interface) | The rules for how compiled code works together: where arguments go, which registers a function may change, how the stack looks. |
| **calling convention** | The part of the ABI about function calls: which registers hold the arguments and the result. |
| **AAPCS64** (Procedure Call Standard for the Arm 64-bit Architecture) | The calling convention of arm64. Apple uses it with small changes. |
| **System V AMD64 ABI** | The calling convention of x86-64 on Linux and macOS. ("System V" is the name of an old Unix version whose ABI documents are still used. AMD64 is the name that AMD, Advanced Micro Devices, gave to x86-64.) |
| **stack pointer** | The register that holds the address of the top of the stack: `sp` on arm64, `rsp` on x86-64. |
| **link register** | On arm64, register `x30` (`lr`): `bl` puts the return address there, and `ret` jumps back to it. |
| **prologue / epilogue** | The instructions at the start of a function that save registers and make room on the stack / at the end that restore them. |
| **name mangling** | Encoding the full path of a Rust function (crate, module, name, a hash) into one unique symbol name, like `_RNvCs6NKwHiZURaG_7p12_asm3add`. |

## 1. Getting the assembly

Put three small functions into `src/lib.rs` of `code/p12-asm`:

```rust
#[inline(never)]
pub fn add(a: u64, b: u64) -> u64 {
    a + b
}

#[inline(never)]
pub fn max_of(a: i64, b: i64) -> i64 {
    if a > b { a } else { b }
}

#[inline(never)]
pub fn find(values: &[u32], target: u32) -> Option<usize> {
    for (i, &v) in values.iter().enumerate() {
        if v == target {
            return Some(i);
        }
    }
    None
}
```

**What happens:**

1. `#[inline(never)]` tells the compiler to keep each function as a real, separate function. Without it, the compiler treats small `pub` functions as candidates for **inlining** (copying their body into each caller, Lesson 12.8) and does not produce them in the library at all, so there would be nothing to look at.
2. `pub` makes the functions part of the library's interface, so the compiler must keep them.

### Way 1: ask `rustc` for an assembly file

`cargo rustc` passes extra options to the compiler. `--emit asm` asks for assembly text:

```bash
cd ~/projects/rust/code/p12-asm
cargo rustc --release -- --emit asm
find target -name '*.s'
```

```text
target/release/build/p12-asm/a80bd35e5626745d/out/p12_asm-a80bd35e5626745d.s
```

(The hash in the path will be different for you.) The file is about 40 lines. The part for `add` looks like this:

```text
	.globl	__RNvCs6NKwHiZURaG_7p12_asm3add
	.p2align	2
__RNvCs6NKwHiZURaG_7p12_asm3add:
	.cfi_startproc
	add	x0, x1, x0
	ret
	.cfi_endproc
```

The lines starting with `.` are **directives** for the assembler; the strange name is the **mangled** name of `p12_asm::add`. Only two lines are real instructions. This works without any extra tool, but it is noisy.

### Way 2: `cargo asm`

The tool cargo-show-asm finds one function, demangles its name and shows only its code. Install it once:

```bash
cargo install --locked cargo-show-asm
cargo asm --version
```

```text
Version: 0.2.65
```

Without a function name, `cargo asm --lib` lists what it can show; with a name, it shows that function (it builds in release mode by default):

```bash
cargo asm --lib p12_asm::add
```

```text
p12_asm::add:
	add x0, x1, x0
	ret
```

In this lesson we leave out the directive lines (`.cfi_…`, `.p2align`, `.globl` and similar) that `cargo asm` also prints, so that you see only labels and instructions. The numbers in labels such as `LBB2_4` depend on the other functions in the file, so yours may differ.

### x86-64 assembly on your Mac

The compiler can produce code for other targets. For x86-64 Linux, add the target's standard library once with `rustup target add x86_64-unknown-linux-gnu`. Then:

```bash
cargo asm --lib --target x86_64-unknown-linux-gnu p12_asm::add
```

```text
p12_asm::add:
	lea rax, [rdi + rsi]
	ret
```

This works without a Linux linker, because a library is only compiled, not linked into a program (Lesson 12.12 explains linking). `cargo asm` uses the **Intel syntax** for x86-64 (destination first, like arm64). Some tools use the older **AT&T syntax** instead (named after AT&T, the American Telephone and Telegraph company, whose Bell Labs created Unix), where the destination comes last and registers have a `%` sign (`leaq (%rdi,%rsi), %rax`); `cargo asm --att` shows that form.

> 💡 **Tip:** Compiler Explorer (godbolt.org) does the same in a web browser: choose Rust, add the option `-C opt-level=3`, and it shows the assembly next to your code, with colours that connect each source line to its instructions.

## 2. Reading one line

An assembly line is: **mnemonic**, then **operands** separated by commas. On arm64 and in Intel syntax, **the first operand is the destination**:

```text
add  x0, x1, x0          x0 = x1 + x0                 (arm64)
lea  rax, [rdi + rsi]    rax = rdi + rsi              (x86-64)
```

Now you can read `add` completely, with the **calling convention**: the rule of where arguments and results are.

| | arm64 (AAPCS64) | x86-64 (System V AMD64 ABI) |
|---|---|---|
| Integer/pointer arguments 1–8 | `x0`, `x1`, `x2`, `x3`, `x4`, `x5`, `x6`, `x7` | `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9` (only six) |
| More arguments | on the stack | on the stack |
| Return value | `x0` (a second part in `x1`) | `rax` (a second part in `rdx`) |
| Return address | register `x30` (link register) | on the stack (pushed by `call`) |
| Registers a function must keep unchanged for its caller | `x19`–`x28`, `x29`, `sp` | `rbx`, `rbp`, `r12`–`r15`, `rsp` |

So on arm64, `a` arrives in `x0`, `b` in `x1`, and `add x0, x1, x0` puts the sum into `x0`, the return register. `ret` jumps back to the address in `x30`. On x86-64, `a` is in `rdi`, `b` in `rsi`, and `lea` ("load effective address") is the address calculator used as a fast adder: it computes `rdi + rsi` into `rax` without touching memory.

**Register sizes.** `x0` is a 64-bit register; `w0` is its lower 32 bits (for `u32`, `i32`). On x86-64, `rax` is 64 bits, `eax` its lower 32 bits. That is why the `add` from Lesson 00.3 (for `i32`) used `w` registers and this one (for `u64`) uses `x` registers.

> 📏 **Convention:** The Rust ABI (how normal Rust functions are called) is officially **not stable**: it may change between compiler versions, and the compiler may pass things in other ways if it can prove nobody else calls the function. In practice simple functions follow the tables above. Only `extern "C"` functions are *guaranteed* to follow the platform's C calling convention (Exercise 4 uses one).

## 3. Decisions: compare and select

`max_of` has an `if`. Here is what the compiler made of it:

```bash
cargo asm --lib p12_asm::max_of
cargo asm --lib --target x86_64-unknown-linux-gnu p12_asm::max_of
```

```text
p12_asm::max_of:
	cmp x0, x1
	csel x0, x0, x1, gt
	ret
```

```text
p12_asm::max_of:
	mov rax, rsi
	cmp rdi, rsi
	cmovg rax, rdi
	ret
```

**What happens:**

1. `cmp x0, x1` computes `x0 − x1` and throws the result away, keeping only the **flags** (N, Z, C, V from Lesson 12.2: negative, zero, carry, overflow).
2. `csel x0, x0, x1, gt` is a **conditional select**: "`x0` = if the flags say *greater than*, then `x0`, else `x1`". `gt` is a **condition code** that the CPU computes from the flags (for signed numbers).
3. On x86-64: first `rax = b`; then compare; then `cmovg` (*conditional move if greater*) overwrites `rax` with `a` if `a > b`.

There is **no branch** at all. Both versions compute the answer with a select instruction, so there is nothing for the branch predictor to guess (Lesson 12.6). Condition codes you will often see:

| Condition | Meaning (after `cmp a, b`) |
|---|---|
| `eq` / `ne` (x86: `e` / `ne`) | equal / not equal |
| `gt`, `ge`, `lt`, `le` (x86: `g`, `ge`, `l`, `le`) | signed greater, greater or equal, less, less or equal |
| `hi`, `hs`, `lo`, `ls` (x86: `a`, `ae`, `b`, `be`) | unsigned higher, higher or same, lower, lower or same |

Signed and unsigned comparisons use **different** condition codes, because they read different flags. This is two's complement from Lesson 12.2 at work: the same `cmp` serves both kinds of numbers.

## 4. Loops and memory: `find`

```text
p12_asm::find:
	cbz x1, LBB2_4
	mov x9, x1
	mov x8, x0
	mov x1, #0
	lsl x9, x9, #2
	mov w0, #1
LBB2_2:
	ldr w10, [x8, x1, lsl #2]
	cmp w10, w2
	b.eq LBB2_5
	add x1, x1, #1
	subs x9, x9, #4
	b.ne LBB2_2
LBB2_4:
	mov x0, #0
LBB2_5:
	ret
```

First, the arguments. A slice `&[u32]` is two values (Lesson 03.6): a pointer to the first element and the length. So `values` arrives in **two registers**: the pointer in `x0`, the length in `x1`. `target` comes third, in `w2` (32 bits, because it is a `u32`). The result `Option<usize>` also needs two values: a tag (0 for `None`, 1 for `Some`) in `x0` and the index in `x1`.

**What happens:**

1. `cbz x1, LBB2_4` — "compare and branch if zero": if the length is 0, jump to `LBB2_4`, which returns `None`.
2. The `mov` instructions copy the pointer to `x8`, start the index `x1` at 0, and set `w0 = 1` (the `Some` tag, ready in case we find the value). `#0` and `#1` are **immediates**. `lsl x9, x9, #2` shifts the length left by 2: `x9` = length × 4 = the number of bytes left to look at (each `u32` is 4 bytes).
3. The loop starts at the label `LBB2_2`. `ldr w10, [x8, x1, lsl #2]` **loads** 4 bytes from memory at the address `x8 + (x1 << 2)`, that is `values[i]`. The square brackets mean "the memory at this address".
4. `cmp w10, w2` and `b.eq LBB2_5`: if `values[i] == target`, jump to the end. At that point `x0` is 1 and `x1` is `i`: the function returns `Some(i)`.
5. Otherwise `add x1, x1, #1` (`i += 1`), and `subs x9, x9, #4` subtracts 4 from the byte counter **and sets the flags** (the `s` at the end of `subs` means "set flags"). `b.ne LBB2_2` jumps back while the counter is not zero.
6. When the loop ends without a match, `mov x0, #0` sets the `None` tag and `ret` returns.

The x86-64 version has the same shape:

```text
p12_asm::find:
	test rsi, rsi
	je .LBB2_1
	mov ecx, edx
	shl rsi, 2
	xor eax, eax
	xor edx, edx
.LBB2_5:
	cmp dword ptr [rdi + 4*rdx], ecx
	je .LBB2_6
	inc rdx
	add rsi, -4
	jne .LBB2_5
	ret
.LBB2_1:
	xor eax, eax
	ret
.LBB2_6:
	mov eax, 1
	ret
```

New things here: `test rsi, rsi` sets the flags from `rsi & rsi`, a short way to ask "is it zero?". `xor eax, eax` sets `eax` to 0 (XOR, *exclusive or*, from Lesson 12.4: a value XOR itself is 0; this is the usual way to clear a register). `cmp dword ptr [rdi + 4*rdx], ecx` compares a 4-byte value (*dword*, double word) **straight from memory**, which x86-64 allows and arm64 does not (Lesson 12.6: arm64 is a load/store design). The index is in `rdx`, which is also the second return register, so `Some(i)` is returned in `rax` = 1 and `rdx` = `i`.

## 5. Calls and the stack

When a function calls another function, it must save what it still needs, because the callee may overwrite the argument registers. Add this to `src/lib.rs`:

```rust
#[inline(never)]
fn square(x: u64) -> u64 {
    x.wrapping_mul(x)
}

#[inline(never)]
pub fn sum_of_squares(a: u64, b: u64) -> u64 {
    square(a).wrapping_add(square(b))
}
```

```text
p12_asm::sum_of_squares:
	stp x20, x19, [sp, #-32]!
	stp x29, x30, [sp, #16]
	add x29, sp, #16
	mov x19, x1
	bl p12_asm::square
	mov x20, x0
	mov x0, x19
	bl p12_asm::square
	add x0, x0, x20
	ldp x29, x30, [sp, #16]
	ldp x20, x19, [sp], #32
	ret
```

**What happens:**

1. **Prologue.** `stp x20, x19, [sp, #-32]!` ("store pair") first moves the stack pointer down by 32 bytes (the `!` means "update `sp` first"), then stores `x20` and `x19` there. The function will use these registers, and the ABI says it must give them back unchanged. `stp x29, x30, [sp, #16]` saves the **frame pointer** `x29` and the **link register** `x30`; the next `bl` will overwrite `x30`, and without the saved copy the function could not return. `add x29, sp, #16` points the frame pointer at this function's frame (debuggers and profilers use this chain of frames to print a backtrace).
2. `mov x19, x1` keeps `b` in a register that `square` is not allowed to change.
3. `bl p12_asm::square` — "branch with link": jump to `square` and put the return address into `x30`. `a` is already in `x0`, the first argument register. The result comes back in `x0` and is kept in `x20`.
4. `b` is moved into `x0` and `square` is called again. Then `add x0, x0, x20` adds the two squares.
5. **Epilogue.** `ldp` ("load pair") restores the saved registers and moves `sp` back up by 32 (`[sp], #32` means "use `sp`, then add 32"). `ret` jumps to the restored `x30`.

```text
the stack inside sum_of_squares (addresses grow upwards):

          ┌──────────────┐  ← sp before the call (sp + 32)
 sp + 24  │ x30 (return) │
 sp + 16  │ x29 (frame)  │  ← x29 points here
 sp + 8   │ x19 (saved)  │
 sp + 0   │ x20 (saved)  │  ← sp after the prologue
          └──────────────┘
```

This block of memory is the function's **stack frame**, the "box on the stack" from Lesson 03.1, now with real addresses. On x86-64 the same function uses `push` (move `rsp` down and store) and `pop`, and `call` pushes the return address on the stack instead of using a link register:

```text
p12_asm::sum_of_squares:
	push r14
	push rbx
	push rax
	mov rbx, rsi
	call p12_asm::square
	mov r14, rax
	mov rdi, rbx
	call p12_asm::square
	add rax, r14
	add rsp, 8
	pop rbx
	pop r14
	ret
```

(`push rax` only moves `rsp` by 8 more bytes; the x86-64 ABI wants the stack aligned to 16 bytes at every `call`. `add rsp, 8` undoes it.)

> 🔒 **Security:** The return address lives on the stack (x86-64) or is saved there (arm64). A classic attack overwrites it through a buffer overflow, so that `ret` jumps into the attacker's code. Safe Rust prevents the buffer overflow itself. Modern CPUs add more protection: on Apple Silicon, the return address can be signed with **pointer authentication** (you may see instructions like `pacibsp` and `retab` in system libraries) so that a changed address is detected before the jump.

## 6. Back to Lesson 12.6: the branch that disappeared

Add `sum_big` from Lesson 12.6 to the library (with `#[inline(never)]`) and run `cargo asm --lib p12_asm::sum_big`. The output is about 180 lines, because the compiler made a **vectorised** main loop: instructions like `uaddw.2d v24, v24, v30` work on the 128-bit **vector registers** `v0`–`v31`, several numbers at once (the `.2d`, `.16b` suffixes say how many lanes and how wide: 2 × 64 bits, 16 × 8 bits). At the end there is a small scalar loop for the last few bytes, and it is worth reading:

```text
LBB6_14:
	ldrsb w11, [x9], #1
	and w11, w11, w11, asr #31
	add x8, x8, w11, uxtb
	cmp x9, x10
	b.ne LBB6_14
```

**What happens:**

1. `ldrsb w11, [x9], #1` loads one byte and **sign-extends** it (Lesson 12.2), then moves the pointer `x9` forward by 1. A byte ≥ 128 has its top bit set, so as a signed byte it is negative.
2. `and w11, w11, w11, asr #31`: the last operand is `w11` **arithmetically shifted** right by 31 (Lesson 12.4). For a negative value that is all ones, for a positive value all zeros. AND with it keeps the value or turns it into 0. This is "if `v >= 128` keep `v`, else 0" without a branch.
3. `add x8, x8, w11, uxtb` adds the lowest byte of `w11`, **zero-extended** (`uxtb`), to the sum in `x8`.
4. The loop runs until the pointer reaches the end `x10`.

All the tricks of this phase — two's complement, sign extension, arithmetic shifts, masks — in four instructions, chosen by the compiler.

> 🧪 **Testing:** Reading assembly tells you *how* a function works, not *whether* it is correct. Keep normal tests next to functions you study this way; in `code/p12-asm`, Exercise 3 adds tests for `find` and a new function before you read their assembly.

## Common mistakes

> ⚠️ **Common mistake: the function is not there.** Without `#[inline(never)]`, a small function in a library is often not compiled on its own:

```rust
pub fn double(x: u64) -> u64 {
    x * 2
}
```

```text
Can't find any items matching ["p12_asm::double"]
```

Add `#[inline(never)]`, or look at a bigger function that calls it. Run `cargo asm --lib` without a name to see the list of functions that exist.

> ⚠️ **Common mistake: reading debug assembly.** Debug builds keep every variable on the stack and add overflow checks, so even `add` becomes many lines. `cargo asm` uses release mode by default; with `cargo rustc`, always pass `--release`.

> ⚠️ **Common mistake: mixing up the operand order.** In AT&T syntax (`gcc`, `objdump` by default on Linux) the destination is **last**: `mov %rsi, %rax` copies `rsi` into `rax`. In Intel syntax and on arm64 it is first. Check which syntax a tool prints before you read it.

## Try it yourself

1. Add `pub fn is_even(x: u32) -> bool { x % 2 == 0 }` (with `#[inline(never)]`) and predict the instructions before you look. (Hint: think of Lesson 12.4. A `bool` is returned as 0 or 1 in `w0`.)
2. Look at `find` for x86-64 with `--att` and compare the two syntaxes line by line.
3. Change `max_of` to take `u64` instead of `i64`. Which condition code changes in `csel` and in `cmov`?

## Exercises

### Exercise 1 — Read it (easy)

What do these two arm64 functions compute? The argument is in `x0` (or `x0`, `x1`, `x2`):

```text
f:
	lsl x0, x0, #3
	ret
g:
	madd x0, x1, x0, x2
	ret
```

<details>
<summary>Hint</summary>

`lsl` is "logical shift left". `madd d, a, b, c` means `d = a × b + c`.

</details>

<details>
<summary>Answer</summary>

`f` returns `x * 8` (a shift left by 3 multiplies by 2³). `g` returns `a * b + c`, where the multiply and the add are **one** instruction. These are the real outputs for:

```rust
#[inline(never)]
pub fn times_eight(x: u64) -> u64 {
    x * 8
}

#[inline(never)]
pub fn mul_add(a: u64, b: u64, c: u64) -> u64 {
    a.wrapping_mul(b).wrapping_add(c)
}
```

On x86-64 they become `lea rax, [8*rdi]` and `imul rdi, rsi` followed by `lea rax, [rdi + rdx]`.

Why it works: the compiler replaces `* 8` with a shift (Lesson 12.4 promised you would see this) and joins multiply and add because arm64 has an instruction for both.

</details>

### Exercise 2 — Predict, then check (medium)

Add `pub fn clamp_to_100(x: i64) -> i64` that returns `x` if it is at most 100, else 100. Before you run `cargo asm`, write down the instructions you expect on arm64. Then compare.

<details>
<summary>Hint</summary>

It is like `max_of`, but against a constant. arm64 `cmp` can compare with an immediate: `cmp x0, #100`. Which condition do you need in `csel`?

</details>

<details>
<summary>Answer</summary>

```rust
#[inline(never)]
pub fn clamp_to_100(x: i64) -> i64 {
    if x > 100 { 100 } else { x }
}
```

```text
p12_asm::clamp_to_100:
	mov w8, #100
	cmp x0, #100
	csel x0, x0, x8, lt
	ret
```

Why it works: the constant 100 must be in a register for `csel`, so it is loaded into `w8` first (writing `w8` also clears the upper half of `x8`). The compiler chose `lt` ("keep `x0` if it is less than 100"), which gives the same result as your `if` for every input, also for `x == 100`. Your prediction may differ in such details and still be right in meaning.

</details>

### Exercise 3 — Test first, then read the loop (medium)

Write `pub fn rfind(values: &[u32], target: u32) -> Option<usize>` that returns the index of the **last** match, using a loop `for i in (0..values.len()).rev()` and `values[i]`. Write tests for both `find` and `rfind` (a match in the middle, at the edges, no match, an empty slice). Then read the arm64 assembly of `rfind`. How does the loop end, and is there a bounds check for `values[i]`?

<details>
<summary>Hint</summary>

A bounds check would show up as a call (`bl`) to a function with `panic_bounds_check` in its name. Look for `cmn`: it is `cmp` with the negated operand, so `cmn x9, #1` compares `x9` with −1.

</details>

<details>
<summary>Answer</summary>

```rust
#[inline(never)]
pub fn rfind(values: &[u32], target: u32) -> Option<usize> {
    for i in (0..values.len()).rev() {
        if values[i] == target {
            return Some(i);
        }
    }
    None
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn find_returns_first_match() {
        assert_eq!(find(&[5, 7, 7, 9], 7), Some(1));
        assert_eq!(find(&[5, 7, 7, 9], 5), Some(0));
        assert_eq!(find(&[5, 7, 7, 9], 4), None);
        assert_eq!(find(&[], 4), None);
    }

    #[test]
    fn rfind_returns_last_match() {
        assert_eq!(rfind(&[5, 7, 7, 9], 7), Some(2));
        assert_eq!(rfind(&[5, 7, 7, 9], 9), Some(3));
        assert_eq!(rfind(&[5, 7, 7, 9], 4), None);
        assert_eq!(rfind(&[], 4), None);
    }
}
```

```text
p12_asm::rfind:
	cbz x1, LBB3_5
	mov x8, x0
	sub x9, x1, #1
	mov w0, #1
LBB3_2:
	mov x1, x9
	ldr w9, [x8, x9, lsl #2]
	cmp w9, w2
	b.eq LBB3_6
	sub x9, x1, #1
	cmn x9, #1
	b.ne LBB3_2
	mov x0, #0
	ret
LBB3_5:
	mov x0, #0
LBB3_6:
	ret
```

Why it works: the index starts at `len − 1` and goes down; after index 0, `sub` makes it −1 (all ones, the wrapped value), and `cmn x9, #1` with `b.ne` stops the loop exactly then. There is **no** bounds check: the compiler proved that `i` is always less than `len`, so `values[i]` cannot panic. Lesson 12.8 shows a case where it cannot prove this.

</details>

### Exercise 4 — Nine arguments (hard)

The calling convention has only 8 argument registers on arm64 and 6 on x86-64. Write

```rust
#[unsafe(no_mangle)]
pub extern "C" fn sum9(a: i64, b: i64, c: i64, d: i64, e: i64, f: i64, g: i64, h: i64, i: i64) -> i64 {
    a + b + c + d + e + f + g + h + i
}
```

and look at it on both targets (`cargo asm --lib sum9`). Where does each argument come from?

<details>
<summary>Hint</summary>

Look for memory operands with `sp` or `rsp`. On x86-64, the return address of the caller sits at `[rsp]` when the function starts.

</details>

<details>
<summary>Answer</summary>

```text
_sum9:
	ldr x8, [sp]
	add x9, x1, x0
	add x10, x2, x3
	add x9, x9, x10
	add x10, x4, x5
	add x10, x10, x6
	add x9, x9, x10
	add x9, x9, x7
	add x0, x9, x8
	ret
```

```text
sum9:
	add rsi, rdi
	lea rax, [rdx + rcx]
	add rax, rsi
	add rax, r8
	add rax, r9
	add rax, qword ptr [rsp + 8]
	add rax, qword ptr [rsp + 16]
	add rax, qword ptr [rsp + 24]
	ret
```

Why it works: on arm64, `a`–`h` arrive in `x0`–`x7` and only `i` comes from the stack (`ldr x8, [sp]`). On x86-64, `a`–`f` arrive in `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`, and `g`, `h`, `i` are read from the stack at `rsp + 8`, `+ 16`, `+ 24` (`[rsp]` holds the return address pushed by `call`). Note also the name: macOS adds an underscore to the front of every symbol name (`_sum9`; that is also why the mangled names in Section 1 start with `__RNv`), Linux does not. With `#[unsafe(no_mangle)]` the name is not mangled, which is what C code needs to find it (Lesson 06.10). And see how the compiler adds in pairs to shorten the dependency chain (Lesson 12.6) — integer addition may be reordered.

</details>

## Quiz

1. In `add x0, x1, x0`, which register receives the result?

<details>
<summary>Answer</summary>

`x0`, the first operand. On arm64 and in Intel syntax the destination comes first.

</details>

2. In which registers do the first two integer arguments arrive on arm64 and on x86-64 Linux, and where is the result returned?

<details>
<summary>Answer</summary>

arm64: `x0` and `x1`, result in `x0`. x86-64: `rdi` and `rsi`, result in `rax`.

</details>

3. Why does a `&[u32]` argument use two registers?

<details>
<summary>Answer</summary>

A slice is a pointer plus a length; each needs one register.

</details>

4. What does `ldr w10, [x8, x1, lsl #2]` do?

<details>
<summary>Answer</summary>

It loads 4 bytes from the memory address `x8 + (x1 << 2)` into `w10`, that is element `x1` of a `u32` array that starts at `x8`.

</details>

5. Why does `sum_of_squares` save `x30` in its prologue, but `add` does not?

<details>
<summary>Answer</summary>

`sum_of_squares` calls other functions with `bl`, which overwrites `x30` (the return address). `add` calls nothing, so `x30` stays intact until its `ret`.

</details>

6. What is the difference between `csel` and a conditional branch like `b.gt`?

<details>
<summary>Answer</summary>

`csel` picks one of two values without changing the flow of the program, so nothing has to be predicted. `b.gt` jumps to another place if the condition is true, which the branch predictor must guess.

</details>

7. Why do you need `#[inline(never)]` to see a small library function with `cargo asm`?

<details>
<summary>Answer</summary>

Small `pub` functions are treated as inlining candidates and may not be compiled as separate functions in the library; `#[inline(never)]` forces a separate copy.

</details>

## Summary

- `cargo rustc --release -- --emit asm` writes a `.s` file; `cargo asm --lib <path>` shows one function cleanly, also for `--target x86_64-unknown-linux-gnu`. Mark small functions `#[inline(never)]`.
- A line is mnemonic plus operands; on arm64 and in Intel syntax the destination comes first. `[...]` is memory, `#n` (arm64) is an immediate.
- Arguments go in `x0`–`x7` (arm64) or `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9` (x86-64); results in `x0` / `rax`. Slices and `Option<usize>` use two registers. Only `extern "C"` guarantees this ABI.
- `cmp` sets flags; `b.<cond>`, `cbz` branch on them; `csel` / `cmov` select without branching. Signed and unsigned comparisons use different condition codes.
- A function that calls others saves registers and the return address in its stack frame (prologue) and restores them (epilogue).
- Reading assembly shows how the compiler implemented your code; tests still decide whether it is correct.

## Sources and further reading

- [cargo-show-asm](https://github.com/pacak/cargo-show-asm) — the tool used in this lesson, with all its options.
- [Apple: Writing ARM64 code for Apple platforms](https://developer.apple.com/documentation/xcode/writing-arm64-code-for-apple-platforms) — where Apple's ABI differs from the standard arm64 one.
- [Arm: Procedure Call Standard for the Arm 64-bit Architecture (AAPCS64)](https://github.com/ARM-software/abi-aa/blob/main/aapcs64/aapcs64.rst) — the official arm64 calling convention.
- [System V AMD64 ABI](https://gitlab.com/x86-psABIs/x86-64-ABI) — the official x86-64 calling convention for Linux.
- [x86 and amd64 instruction reference](https://www.felixcloutier.com/x86/) — look up any x86-64 instruction.
- [Compiler Explorer](https://godbolt.org/) — see the assembly of Rust code in the browser.

---

[🏠 Start](../START-HERE.md) · [← 12.6 How the CPU runs instructions](06-how-the-cpu-runs-instructions.md) · [📋 Phase 12](README.md) · [12.8 From Rust to machine code →](08-from-rust-to-machine-code.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
