<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 19.6 Data layout for speed](06-data-layout-for-speed.md) · [📋 Phase 19](README.md) · [19.8 Build settings for speed: profiles, LTO and PGO →](08-build-settings-for-speed.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/19-performance-and-profiling/07-simd.md) <!-- nav -->

# Lesson 19.7 — SIMD: one instruction, many data

> **Phase 19 — Performance and profiling** · Lesson 7 of 11 · ⏱ about 135 minutes

## What you will learn

- What SIMD is, what vector registers and lanes are, and how wide they are on your Mac and on x86-64 computers.
- How the compiler turns ordinary loops into SIMD code by itself (auto-vectorization), and how to check this in the assembly.
- Why a simple float sum is *not* vectorized, and how to write it so that it is.
- How to use NEON intrinsics from `std::arch::aarch64` safely, with a runtime check and a test against a simple version.

## Before you start

- You finished [Lesson 19.6 — Data layout for speed](06-data-layout-for-speed.md).
- You can read arm64 assembly and use `cargo asm` ([Lesson 12.7 — Reading assembly](../12-computer-systems/07-reading-assembly.md)).
- You know dependency chains and the trick with four separate sums ([Lesson 12.6 — How the CPU runs instructions](../12-computer-systems/06-how-the-cpu-runs-instructions.md)), and why the order of float additions matters ([Lesson 12.3 — Floating-point numbers](../12-computer-systems/03-floating-point-numbers.md)).
- You know `unsafe` blocks and `// SAFETY:` comments ([Lesson 06.6 — Unsafe Rust and Miri](../06-rust-advanced/06-unsafe-rust-and-miri.md)) and `black_box` ([Lesson 05.12 — Benchmarking basics](../05-rust-intermediate/12-benchmarking-basics.md)).
- Create the practice package: `cargo new --lib code/p19-simd` (from `~/projects/rust`).

## New words

| Word | Meaning |
|---|---|
| **SIMD** (Single Instruction, Multiple Data) | A kind of CPU (Central Processing Unit) instruction that does the same operation on several values at once, for example four additions in one instruction. |
| **scalar** | One single value, or code that works on one value at a time. The opposite of vector. |
| **vector register** | A wide CPU register that holds several values side by side. On arm64 there are 32 of them, `v0`–`v31`, each 128 bits wide. |
| **lane** | One slot of a vector register. A 128-bit register has 4 lanes of 32 bits, or 16 lanes of 8 bits. |
| **NEON** | The SIMD instruction set of arm64 CPUs, including the Apple M-series chips. It is always present on 64-bit Arm. |
| **SSE / AVX** (Streaming SIMD Extensions / Advanced Vector Extensions) | The SIMD instruction sets of x86-64 CPUs. SSE2 uses 128-bit registers and is always present; AVX2 uses 256-bit registers and exists only on newer CPUs. |
| **auto-vectorization** | The compiler turns a normal loop into SIMD code by itself, without any change in your source. |
| **intrinsic** | A function that the compiler replaces by one specific CPU instruction. `vaddq_f32` becomes one NEON `fadd` on four floats. |
| **target feature** | An optional part of an instruction set (such as `neon` or `avx2`) that the compiler may use only when it knows the CPU has it. |
| **reduction** | Combining all values into one result: a sum, a maximum, a count. |
| **horizontal operation** | An operation between the lanes of one register, for example adding its four lanes together. |

## 1. One instruction, four additions

Think of a cashier who scans one item at a time. A SIMD cashier has a wide scanner that reads four items in one move. The work per item is the same; the number of moves is four times smaller.

A normal register (`x0`, `w0`) holds one number. A **vector register** holds several numbers side by side, in **lanes**. On your Mac's arm64 CPU there are 32 vector registers, `v0` to `v31`, each 128 bits wide. The assembly names say how a register is split:

```text
  v0 as .16b : 16 lanes of 8 bits   (u8, i8)
  v0 as .8h  :  8 lanes of 16 bits  (u16, i16)
  v0 as .4s  :  4 lanes of 32 bits  (u32, i32, f32)
  v0 as .2d  :  2 lanes of 64 bits  (u64, i64, f64)

  add.4s v0, v1, v2      one instruction, four additions:

  v1:  [  1 |  2 |  3 |  4 ]
  v2:  [ 10 | 20 | 30 | 40 ]
        +    +    +    +
  v0:  [ 11 | 22 | 33 | 44 ]
```

The same register can also hold a single scalar value: then the assembly calls it `s0` (one 32-bit float) or `d0` (one 64-bit float). So when you read assembly, `fadd s0, s0, s1` adds **one** float, and `fadd.4s v0, v0, v1` adds **four**.

The width depends on the CPU family:

| CPU family | Instruction set | Register width | `f32` values per instruction |
|---|---|---|---|
| arm64 (your Mac, most phones, many servers) | NEON | 128 bits | 4 |
| x86-64, every CPU | SSE2 | 128 bits | 4 |
| x86-64, newer CPUs | AVX2 | 256 bits | 8 |
| x86-64, some server CPUs | AVX-512 | 512 bits | 16 |

NEON and SSE2 are always present, so the compiler uses them by default. AVX2 and AVX-512 are **target features**: the compiler uses them only if you tell it that the CPU has them (you will do this in [Lesson 19.8](08-build-settings-for-speed.md)).

## 2. Auto-vectorization: the compiler does it for you

You rarely write SIMD by hand. In release mode the compiler looks at each loop and asks: "can I do several iterations at once?" If yes, it writes SIMD code for you. This is **auto-vectorization**. Put this into `src/lib.rs` of `code/p19-simd`:

```rust
/// Adds all values. Wraps around on overflow instead of panicking.
#[inline(never)]
pub fn sum_u32(values: &[u32]) -> u32 {
    let mut total = 0u32;
    for &v in values {
        total = total.wrapping_add(v);
    }
    total
}

/// Adds all values one by one, from left to right.
#[inline(never)]
pub fn sum_f32(values: &[f32]) -> f32 {
    let mut total = 0.0;
    for &v in values {
        total += v;
    }
    total
}
```

**What happens:**

1. `#[inline(never)]` keeps each function as a real function, so `cargo asm` can show it (Lesson 12.7).
2. `wrapping_add` says what to do on overflow: wrap around (Lesson 12.2). This makes the result the same in debug and release builds.
3. Both functions do the same thing: a **reduction** of a slice to one number.

Now look at the assembly of the integer version (here and below, the directive lines that start with `.` are left out):

```bash
cd ~/projects/rust/code/p19-simd
cargo asm --lib p19_simd::sum_u32
```

The important part, the main loop:

```text
LBB3_7:
	ldp q4, q5, [x8, #-32]
	ldp q6, q7, [x8], #64
	add.4s v0, v4, v0
	add.4s v1, v5, v1
	add.4s v2, v6, v2
	add.4s v3, v7, v3
	subs x14, x14, #16
	b.ne LBB3_7
	add.4s v0, v1, v0
	add.4s v0, v2, v0
	add.4s v0, v3, v0
	addv.4s s0, v0
```

**What happens:**

1. `ldp q4, q5, [x8, #-32]` loads a *pair* of 128-bit registers (`q4` and `q5` are the 128-bit names of `v4` and `v5`) from memory: 8 numbers. The next line loads 8 more. So one trip through the loop reads 16 numbers.
2. The four `add.4s` lines each add 4 numbers into a running vector total. There are four totals, `v0`–`v3`, so the four additions do not wait for each other. This is the "four chains" idea from Lesson 12.6, done by the compiler, and each chain is 4 lanes wide.
3. `subs x14, x14, #16` counts down by 16 numbers, and `b.ne` jumps back while numbers are left.
4. After the loop, the three `add.4s` lines combine the four totals into one register. `addv.4s s0, v0` is a **horizontal** addition: it adds the 4 lanes of `v0` into one number.
5. Code that is not shown handles the last few numbers when the length is not a multiple of 16.

(Label numbers such as `LBB3_7` and register numbers may differ for you, as in Lesson 12.7.)

The source code has one simple loop. The machine code does 16 additions per trip. You got this for free.

Now the float version:

```bash
cargo asm --lib p19_simd::sum_f32
```

(Later, when more names start with `sum_f32`, `cargo asm` prints a numbered list of the matches instead; then add the number, for example `cargo asm --lib p19_simd::sum_f32 0`.) The main loop:

```text
LBB2_3:
	ldp s1, s2, [x12, #-8]
	ldp s3, s4, [x12], #16
	fadd s0, s0, s1
	fadd s0, s0, s2
	fadd s0, s0, s3
	fadd s0, s0, s4
	subs x14, x14, #4
	b.ne LBB2_3
```

**What happens:**

1. The loads use `s1`…`s4`: one float each. No `.4s` anywhere.
2. All four `fadd` lines write to `s0` and read `s0`. Each addition must wait for the one before it: one long dependency chain.
3. The compiler did unroll the loop (four additions per trip, to save on loop counting), but it did **not** vectorize it.

Why the difference? You learned it in Lesson 12.3: for floats, `(a + b) + c` is not always equal to `a + (b + c)`. A SIMD sum adds the numbers in a *different order* (lane 0 gets values 0, 4, 8, …, lane 1 gets 1, 5, 9, …). For integers the order does not matter, so the compiler is free to do it. For floats it would change your result, and the compiler never changes your result just to be faster.

## 3. Helping the compiler: write the order you want

If *you* accept a different order, write it in the code. Keep 8 separate running totals ("lanes") and add them up at the end. Add this to `src/lib.rs`:

```rust
/// Adds the values in 8 separate running totals ("lanes"), then adds the lanes.
#[inline(never)]
pub fn sum_f32_lanes(values: &[f32]) -> f32 {
    let mut lanes = [0.0f32; 8];
    let (chunks, rest) = values.as_chunks::<8>();
    for chunk in chunks {
        for (lane, &v) in lanes.iter_mut().zip(chunk) {
            *lane += v;
        }
    }
    let mut total: f32 = lanes.iter().sum();
    for &v in rest {
        total += v;
    }
    total
}
```

**What happens:**

1. `values.as_chunks::<8>()` splits the slice into two parts: a slice of arrays `[f32; 8]` (as many full groups of 8 as fit) and the `rest`, a normal slice with the 0 to 7 values that are left.
2. For each group, value number `i` goes into `lanes[i]`. Lane 0 gets values 0, 8, 16, …; lane 1 gets values 1, 9, 17, …. This is the order a SIMD sum uses, and now it is written in the source.
3. Because the group is an array with a fixed length of 8, the compiler knows the inner loop has exactly 8 steps and needs no bounds checks.
4. At the end the 8 lanes are added together, and then the rest.

Check the assembly with `cargo asm --lib p19_simd::sum_f32_lanes`:

```text
LBB1_2:
	ldp q2, q3, [x11], #32
	fadd.4s v0, v3, v0
	fadd.4s v1, v2, v1
	subs x10, x10, #32
	b.ne LBB1_2
```

Eight float additions per trip, in two independent vector chains (`v0` and `v1`, 4 lanes each). The `lanes` array lives in two vector registers and never touches memory.

> 📏 **Convention:** Clippy (Lesson 05.6) warns about `chunks_exact(8)` with a constant size and suggests `as_chunks::<8>()`. Prefer `as_chunks` when the size is a constant: you get real arrays, so the compiler knows the length.

### Is the result the same?

Not always, and you must know this. Put this into `examples/order.rs`:

```rust
use p19_simd::{sum_f32, sum_f32_lanes};

fn main() {
    let values = vec![0.1f32; 1_000_000];
    println!("one by one: {}", sum_f32(&values));
    println!("8 lanes:    {}", sum_f32_lanes(&values));
    println!("exact:      {}", 0.1f64 * 1_000_000.0);
}
```

Run it with `cargo run --release --example order`:

```text
one by one: 100958.34
8 lanes:    99910.32
exact:      100000
```

**What happens:**

1. One million times `0.1`. The exact answer is 100,000.
2. The one-by-one sum is off by almost 1,000. When `total` becomes large, each small `0.1` is rounded badly when it is added to it.
3. The 8-lane sum is off by about 90. Each lane only grows to about 12,500, so each addition loses less. The lane version is faster *and* here more accurate, but it is **different**. If a program must print exactly the same numbers as before (Lesson 19.10 has such a test), a changed summation order can break that.

## 4. Measuring the difference

Put this into `src/main.rs`. It uses the `black_box` idea from Lesson 05.12 and takes the best of five runs to reduce noise:

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

use p19_simd::{sum_f32, sum_f32_lanes, sum_f32_neon, sum_u32};

const LEN: usize = 4096; // 16 KiB of data: it stays in the L1 cache
const ROUNDS: u32 = 20_000;

/// Runs `f` ROUNDS times, five times over, and prints the fastest time per call.
fn time<T: std::fmt::Display>(name: &str, mut f: impl FnMut() -> T) {
    let mut best = Duration::MAX;
    let mut result = None;
    for _ in 0..5 {
        let start = Instant::now();
        for _ in 0..ROUNDS {
            result = Some(black_box(f()));
        }
        best = best.min(start.elapsed() / ROUNDS);
    }
    println!("{name:<14} {best:>9.2?} per call  (result {})", result.unwrap());
}

fn main() {
    let ints: Vec<u32> = (0..LEN as u32).collect();
    let floats: Vec<f32> = (0..LEN).map(|i| (i % 10) as f32 * 0.5).collect();

    time("sum_u32", || sum_u32(black_box(&ints)));
    time("sum_f32", || sum_f32(black_box(&floats)));
    time("sum_f32_lanes", || sum_f32_lanes(black_box(&floats)));
    time("sum_f32_neon", || sum_f32_neon(black_box(&floats)));
}
```

`sum_f32_neon` comes in Section 5; add it before you run this. Output of `cargo run --release` on an M4 (your numbers will differ):

```text
sum_u32         139.00ns per call  (result 8386560)
sum_f32           2.90µs per call  (result 9210)
sum_f32_lanes   304.00ns per call  (result 9210)
sum_f32_neon    625.00ns per call  (result 9210)
```

**What happens:**

1. 4,096 floats one by one take about 2.9 µs (microseconds): about 0.7 ns per addition, which is the latency of one float addition on this CPU. The chain decides the speed.
2. The 8-lane version is about **9 times faster** with the same result. It is faster for two reasons together: 4 values per instruction (SIMD), and two independent chains instead of one.
3. The integer sum is the fastest: four vector chains, 16 values per trip.
4. The data are small numbers like 0.5 and 4.5, so every sum is exact and all versions print the same 9210.

> ⚡ **Performance:** SIMD helps most when the data are already in the cache. When the data come from main memory, the limit can become **memory bandwidth** (how many bytes per second the memory delivers, Lesson 12.9), and then a wider instruction cannot go faster. Always measure with data of a realistic size.

## 5. Writing SIMD by hand: NEON intrinsics

Sometimes the compiler cannot find the SIMD version, and you know exactly which instructions you want. Then you use **intrinsics**: functions in `std::arch` that each become one CPU instruction. The NEON ones live in `std::arch::aarch64`. Their names are short codes; for example `vaddq_f32` means "**v**ector **add**, **q** = full 128-bit register, of **f32** values". Add this to `src/lib.rs`:

```rust
#[cfg(target_arch = "aarch64")]
#[target_feature(enable = "neon")]
fn sum_f32_neon_inner(values: &[f32]) -> f32 {
    use std::arch::aarch64::{vaddq_f32, vaddvq_f32, vdupq_n_f32, vld1q_f32};

    let (chunks, rest) = values.as_chunks::<4>();
    let mut acc = vdupq_n_f32(0.0);
    for chunk in chunks {
        // SAFETY: `chunk` is a [f32; 4], so reading 4 values (16 bytes) stays in bounds.
        let v = unsafe { vld1q_f32(chunk.as_ptr()) };
        acc = vaddq_f32(acc, v);
    }
    let mut total = vaddvq_f32(acc);
    for &v in rest {
        total += v;
    }
    total
}

/// Adds the values with NEON instructions, 4 at a time.
#[cfg(target_arch = "aarch64")]
#[inline(never)]
pub fn sum_f32_neon(values: &[f32]) -> f32 {
    if std::arch::is_aarch64_feature_detected!("neon") {
        // SAFETY: we have just checked that this CPU supports NEON.
        unsafe { sum_f32_neon_inner(values) }
    } else {
        sum_f32(values)
    }
}
```

**What happens:**

1. `#[cfg(target_arch = "aarch64")]` compiles these functions only for arm64. On other CPUs they do not exist (a program for x86-64 would need its own version or none).
2. `#[target_feature(enable = "neon")]` says: "inside this function, the compiler may use NEON". Inside such a function, intrinsics that only compute with values are safe to call.
3. `vdupq_n_f32(0.0)` makes a vector with `0.0` in all 4 lanes. Its type is `float32x4_t`: four `f32` in one register.
4. `vld1q_f32(ptr)` loads 4 floats from memory. It takes a raw pointer, so it is `unsafe`: Rust cannot check that 16 bytes are readable there. The `// SAFETY:` comment explains why it is fine: the chunk is an array of exactly 4.
5. `vaddq_f32` adds 4 lanes to 4 lanes. `vaddvq_f32` is the horizontal sum of the 4 lanes.
6. The public function first asks the CPU at run time whether it has NEON (`is_aarch64_feature_detected!`). Only then does it call the inner function, which is `unsafe` to call from code without the feature: calling it on a CPU without the feature would be undefined behaviour.

On arm64 the check is always true, and the compiler knows it: in the release assembly of `sum_f32_neon` the check is gone, and the loop is just `ldr q1` (load 4 floats) and `fadd.4s v0, v0, v1`. The pattern "detect at run time, then call the `#[target_feature]` function" is still the one to learn, because on x86-64 it is needed for real: AVX2 is not on every CPU (`is_x86_feature_detected!("avx2")`).

Look at the timing in Section 4 again: the hand-written NEON version (625 ns) is **slower** than the plain Rust lane version (304 ns). It has only one vector chain, `acc`, so each `fadd.4s` waits for the previous one. Hand-written SIMD is not automatically fast; Exercise 3 fixes it.

> 🧪 **Testing:** Every SIMD function needs a test that compares it with a simple version, for **many lengths**, including 0, 1, a multiple of the width and one more than a multiple. Most SIMD bugs live in the `rest` part.

Add this test module at the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn close(a: f32, b: f32) -> bool {
        (a - b).abs() <= 1e-4 * a.abs().max(b.abs()).max(1.0)
    }

    #[test]
    fn every_length_from_0_to_40_gives_the_same_sum() {
        for len in 0..=40 {
            let values: Vec<f32> = (0..len).map(|i| i as f32 * 0.25).collect();
            let expected = sum_f32(&values);
            assert!(close(sum_f32_lanes(&values), expected), "lanes, len {len}");
            #[cfg(target_arch = "aarch64")]
            assert!(close(sum_f32_neon(&values), expected), "neon, len {len}");
        }
    }

    #[test]
    fn integer_sum_wraps_instead_of_panicking() {
        assert_eq!(sum_u32(&[u32::MAX, 2]), 1);
    }
}
```

`close` compares floats with a tolerance relative to their size (Lesson 12.3), because different summation orders may round differently. The loop covers "no full group", "exactly full groups" and every possible size of `rest`. `cargo test`:

```text
test tests::every_length_from_0_to_40_gives_the_same_sum ... ok
test tests::integer_sum_wraps_instead_of_panicking ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

### Portable SIMD is not stable yet

Intrinsics are tied to one CPU family. The standard library also has a **portable** SIMD module, `std::simd`, with types like `f32x4` that work on every CPU. In Rust 1.100 it is still **nightly only**. On stable Rust:

```rust
use std::simd::f32x4;

fn main() {
    let a = f32x4::from_array([1.0, 2.0, 3.0, 4.0]);
    println!("{:?}", a + a);
}
```

```text
error[E0658]: use of unstable library feature `portable_simd`
 --> src/main.rs:1:5
  |
1 | use std::simd::f32x4;
  |     ^^^^^^^^^^^^^^^^
  |
  = note: see issue #86656 <https://github.com/rust-lang/rust/issues/86656> for more information
```

On stable, write plain loops that the compiler can vectorize (Sections 2–3), use intrinsics behind `#[cfg]`, or use a crate that already contains SIMD code: for example `memchr` (fast byte search, used by `regex` and many others).

## 6. What stops auto-vectorization

Before you write intrinsics, check why the compiler did not vectorize. The usual reasons:

| Reason | Example | What to do |
|---|---|---|
| Float order | `total += v` on `f32`/`f64` | Write the lanes yourself (Section 3), if a different order is acceptable. |
| Early exit | `values.iter().position(\|&b\| b == 0)` | The loop may stop at any element. Use a crate (`memchr`) or intrinsics. |
| Each step needs the previous one | a running total written into an array: `out[i] = out[i - 1] + a[i]` | Usually cannot be vectorized simply; choose another algorithm. |
| A call inside the loop | calling a function that is not inlined | Make sure it can be inlined (small, in the same crate, or `#[inline]`). |
| Mixed or unknown lengths | indexing `a[i]`, `b[i]`, `out[i]` with one `i` | Iterate with `zip` or slice all to the same length first; the compiler then removes bounds checks. |

You can check every case in the assembly: look for `.4s`, `.16b`, `.2d` and `q` registers in the loop.

On x86-64 the same `sum_u32` uses 128-bit SSE2 by default (`paddd xmm0, xmm1`), and 256-bit AVX2 only if you allow the target feature. You can see both from your Mac (the x86-64 target from Lesson 12.7 is needed). `RUSTFLAGS` is an environment variable with extra compiler options (Lesson 19.8 explains it):

```bash
cargo asm --lib --target x86_64-unknown-linux-gnu p19_simd::sum_u32 | grep padd
RUSTFLAGS="-C target-cpu=x86-64-v3" cargo asm --lib --target x86_64-unknown-linux-gnu p19_simd::sum_u32 | grep padd
```

```text
	paddd xmm1, xmm2
	paddd xmm0, xmm2
	...
	vpaddd ymm0, ymm0, ymmword ptr [rdi + 4*rax]
	vpaddd ymm1, ymm1, ymmword ptr [rdi + 4*rax + 32]
	...
```

`xmm` registers are 128 bits, `ymm` registers are 256 bits. Same source, twice the width, but that program would crash with "illegal instruction" on an older x86-64 CPU without AVX2.

> 🔒 **Security:** `unsafe` SIMD code reads memory through raw pointers. A wrong length calculation reads past the end of a slice: that is undefined behaviour and can leak data. Keep pointer loads on fixed-size chunks (`as_chunks`), as in Section 5, and test every length.

## Common mistakes

> ⚠️ **Common mistake:** calling an intrinsic in a normal function. Without `#[target_feature(enable = "neon")]` on the function, every NEON intrinsic is `unsafe` to call, even though NEON is always on for your Mac:
>
> ```text
> error[E0133]: call to function `vaddq_f32` with `#[target_feature]` is unsafe and requires unsafe block
>   --> src/lib.rs:46:15
>    |
> 46 |         acc = vaddq_f32(acc, v);
>    |               ^^^^^^^^^^^^^^^^^ call to function with `#[target_feature]`
>    |
>    = help: in order for the call to be safe, the context requires the following additional target feature: neon
>    = note: the neon target feature being enabled in the build configuration does not remove the requirement to list it in `#[target_feature]`
> ```
>
> Fix: put the SIMD loop in its own function with `#[target_feature(enable = "neon")]`, as in Section 5, and call it from a wrapper after the runtime check.

> ⚠️ **Common mistake:** forgetting the `rest`. A loop over `as_chunks::<4>()` silently skips the last 1–3 values. A test with lengths 0..=40 finds this at once.

> ⚠️ **Common mistake:** expecting the exact same float result. A SIMD sum adds in a different order, so the last digits may differ. Compare with a tolerance, or keep the old order where exact output matters.

## Try it yourself

1. Build `code/p19-simd` with all four functions, the test module, `examples/order.rs` and `src/main.rs`.
2. Run `cargo test`, then `cargo run --release` three times. Write down the best time of each function.
3. In `sum_f32_lanes`, change `[0.0f32; 8]` and `as_chunks::<8>()` to 4 lanes, then to 16 lanes. Check the assembly and the time each time. How many `fadd.4s` are in the loop?
4. Change `LEN` to `64 * 1024 * 1024` (256 MiB, much larger than the caches) and `ROUNDS` to `5`. Compare the times *and* the results. (On an M4, `sum_f32_lanes` was still about 7 times faster, about 5 ms against 38 ms, because the M4 reads memory very fast. The results were no longer equal: 126015530 one by one, 148981680 with lanes; the exact sum is 150994938.)

## Exercises

### Exercise 1 — Does `max` vectorize? (easy)

Add `pub fn max_i32(values: &[i32]) -> Option<i32>` that returns `values.iter().copied().max()`, mark it `#[inline(never)]`, and look at its assembly. Is it vectorized? Which instruction finds the maximum of 4 lanes at once?

<details>
<summary>Hint</summary>

Look for an instruction with `.4s` in the main loop, and for an instruction ending in `v` after the loop (like `addv`).

</details>

<details>
<summary>Answer</summary>

```rust
#[inline(never)]
pub fn max_i32(values: &[i32]) -> Option<i32> {
    values.iter().copied().max()
}
```

Yes, it is vectorized. The main loop of `cargo asm --lib p19_simd::max_i32`:

```text
	smax.4s v0, v0, v4
	smax.4s v1, v1, v5
	smax.4s v2, v2, v6
	smax.4s v3, v3, v7
	...
	smaxv.4s s0, v0
```

Why it works: `smax.4s` ("signed maximum") takes the larger value lane by lane; `smaxv.4s` is the horizontal maximum of the 4 lanes. The maximum of integers does not depend on the order, so the compiler may vectorize it freely, like the integer sum.

</details>

### Exercise 2 — A dot product in lanes, with a test (medium)

The dot product of two slices is `a[0]*b[0] + a[1]*b[1] + …`. Write `dot(a, b)` the simple way (with `zip`, `map`, `sum`) and `dot_lanes(a, b)` with 8 lanes like `sum_f32_lanes`. Use only the first `min(a.len(), b.len())` values. Then write a test that compares them for every length from 0 to 40.

<details>
<summary>Hint</summary>

Shorten both slices to the same length first: `let (a, b) = (&a[..len], &b[..len]);`. Then call `as_chunks::<8>()` on both and `zip` the chunks.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn dot(a: &[f32], b: &[f32]) -> f32 {
    a.iter().zip(b).map(|(x, y)| x * y).sum()
}

pub fn dot_lanes(a: &[f32], b: &[f32]) -> f32 {
    let len = a.len().min(b.len());
    let (a, b) = (&a[..len], &b[..len]);
    let (a_chunks, a_rest) = a.as_chunks::<8>();
    let (b_chunks, b_rest) = b.as_chunks::<8>();
    let mut lanes = [0.0f32; 8];
    for (ca, cb) in a_chunks.iter().zip(b_chunks) {
        for ((lane, x), y) in lanes.iter_mut().zip(ca).zip(cb) {
            *lane += x * y;
        }
    }
    let mut total: f32 = lanes.iter().sum();
    for (x, y) in a_rest.iter().zip(b_rest) {
        total += x * y;
    }
    total
}

#[cfg(test)]
mod dot_tests {
    use super::*;

    #[test]
    fn dot_lanes_matches_simple_dot_for_many_lengths() {
        for len in 0..=40 {
            let a: Vec<f32> = (0..len).map(|i| i as f32 * 0.5).collect();
            let b: Vec<f32> = (0..len).map(|i| 3.0 - i as f32 * 0.25).collect();
            let expected = dot(&a, &b);
            let got = dot_lanes(&a, &b);
            assert!((got - expected).abs() <= 1e-3, "len {len}: {got} vs {expected}");
        }
    }
}
```

```text
test dot_tests::dot_lanes_matches_simple_dot_for_many_lengths ... ok
```

Why it works: both slices have the same length, so both `rest` parts have the same length too. In release mode the loop becomes `fmul.4s` and `fadd.4s` on two vector chains. The test covers every possible size of the rest.

</details>

### Exercise 3 — Two NEON chains (hard)

The NEON version in Section 5 has one chain and is slower than `sum_f32_lanes`. Write `sum_f32_neon2` that processes 8 floats per trip into two accumulators, `acc0` and `acc1`, and adds them at the end. Keep the runtime check and the `// SAFETY:` comments, add it to the test from Section 5, then time it.

<details>
<summary>Hint</summary>

Use `as_chunks::<8>()`. Load the first 4 values with `vld1q_f32(chunk.as_ptr())` and the next 4 with `vld1q_f32(chunk.as_ptr().add(4))`. At the end: `vaddvq_f32(vaddq_f32(acc0, acc1))`.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(target_arch = "aarch64")]
#[target_feature(enable = "neon")]
fn sum_neon2_inner(values: &[f32]) -> f32 {
    use std::arch::aarch64::{vaddq_f32, vaddvq_f32, vdupq_n_f32, vld1q_f32};

    let (chunks, rest) = values.as_chunks::<8>();
    let mut acc0 = vdupq_n_f32(0.0);
    let mut acc1 = vdupq_n_f32(0.0);
    for chunk in chunks {
        // SAFETY: `chunk` is a [f32; 8]; we read values 0..4 and 4..8, both in bounds.
        let (v0, v1) = unsafe { (vld1q_f32(chunk.as_ptr()), vld1q_f32(chunk.as_ptr().add(4))) };
        acc0 = vaddq_f32(acc0, v0);
        acc1 = vaddq_f32(acc1, v1);
    }
    let mut total = vaddvq_f32(vaddq_f32(acc0, acc1));
    for &v in rest {
        total += v;
    }
    total
}

#[cfg(target_arch = "aarch64")]
#[inline(never)]
pub fn sum_f32_neon2(values: &[f32]) -> f32 {
    if std::arch::is_aarch64_feature_detected!("neon") {
        // SAFETY: we have just checked that this CPU supports NEON.
        unsafe { sum_neon2_inner(values) }
    } else {
        values.iter().sum()
    }
}
```

Why it works: `acc0` and `acc1` do not depend on each other, so the CPU runs their additions side by side. The time drops to about the time of `sum_f32_lanes` (about 300 ns on an M4), because this is now the same machine code the compiler made from the plain lane version. That is the real lesson: the plain Rust version was just as fast and needs no `unsafe`.

</details>

## Quiz

1. What is a lane, and how many `u8` lanes does a 128-bit register have?

<details>
<summary>Answer</summary>

A lane is one slot of a vector register. A 128-bit register has 16 lanes of 8 bits, so 16 `u8` values.

</details>

2. Why does the compiler vectorize a `u32` sum but not an `f32` sum?

<details>
<summary>Answer</summary>

A SIMD sum adds the values in a different order. For integers (with wrapping) the result is the same in any order. For floats, rounding makes the result depend on the order, and the compiler must not change your results.

</details>

3. Why is `sum_f32_lanes` both faster and different in its last digits?

<details>
<summary>Answer</summary>

Faster: the compiler puts the 8 lanes into two vector registers, so it does 8 additions per trip in two independent chains. Different: the additions happen in a different order, so they round differently.

</details>

4. What does `#[target_feature(enable = "neon")]` allow, and why is the function then `unsafe` to call from normal code?

<details>
<summary>Answer</summary>

Inside the function, the compiler may use NEON instructions, and value-only intrinsics are safe to call. If such a function ran on a CPU without that feature, the CPU would hit an unknown instruction: undefined behaviour. So the caller must guarantee the feature exists, usually with `is_aarch64_feature_detected!` or `is_x86_feature_detected!`.

</details>

5. Why is `vld1q_f32` `unsafe` even inside a `#[target_feature]` function?

<details>
<summary>Answer</summary>

It reads 16 bytes through a raw pointer. The compiler cannot check that those bytes are inside a valid slice; the programmer must guarantee it.

</details>

6. Can you use `std::simd` on stable Rust 1.100?

<details>
<summary>Answer</summary>

No. Portable SIMD (`std::simd`, feature `portable_simd`) is still nightly only. On stable you write vectorizable loops, use `std::arch` intrinsics, or use crates such as `memchr`.

</details>

## Summary

- SIMD instructions do the same operation on several values at once. On arm64 (NEON) a vector register is 128 bits: 4 `f32`, 16 `u8`. On x86-64, SSE2 is 128 bits, AVX2 256 bits.
- In release mode the compiler **auto-vectorizes** simple loops. Check it in `cargo asm`: look for `.4s`, `.16b` and `q` registers in the loop.
- Float reductions are not vectorized, because a different order gives a different result. Write the lanes yourself (`as_chunks::<8>()` and an array of totals) if a different result is acceptable.
- Several independent chains matter as much as SIMD width: one vector chain was slower than two.
- Intrinsics from `std::arch::aarch64` go into a `#[target_feature(enable = "neon")]` function, called after a runtime check; pointer loads need `unsafe` with a `// SAFETY:` comment.
- Test every SIMD function against a simple version for many lengths, with a tolerance for floats.
- `std::simd` (portable SIMD) is still nightly only in Rust 1.100.

## Sources and further reading

- [`std::arch` module](https://doc.rust-lang.org/std/arch/index.html) — the official documentation of CPU-specific intrinsics, runtime detection and `#[target_feature]`.
- [`core::arch::aarch64`](https://doc.rust-lang.org/core/arch/aarch64/index.html) — the list of all NEON intrinsics in Rust (`std::arch::aarch64` is the same module).
- [The Rust Reference — The `target_feature` attribute](https://doc.rust-lang.org/reference/attributes/codegen.html) — the exact rules for `#[target_feature]` and when calls are safe.
- [Arm C Language Extensions — Advanced SIMD (Neon) intrinsics](https://arm-software.github.io/acle/neon_intrinsics/advsimd.html) — Arm's own specification of every NEON intrinsic and the instruction it becomes.
- [`slice::as_chunks`](https://doc.rust-lang.org/std/primitive.slice.html) — the slice method used in this lesson (search the page for `as_chunks`).
- [`std::simd` (nightly)](https://doc.rust-lang.org/std/simd/index.html) — the portable SIMD module that is not stable yet.
- [The Rust Performance Book](https://nnethercote.github.io/perf-book/) — practical advice on making Rust programs faster.

---

[🏠 Start](../START-HERE.md) · [← 19.6 Data layout for speed](06-data-layout-for-speed.md) · [📋 Phase 19](README.md) · [19.8 Build settings for speed: profiles, LTO and PGO →](08-build-settings-for-speed.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
