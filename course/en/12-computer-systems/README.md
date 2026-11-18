<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 11.16 Checkpoint: Stage 2 complete](../11-linux-shell-and-containers/16-checkpoint-stage-2-complete.md) · [12.1 Bits, bytes and number systems →](01-bits-bytes-and-number-systems.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/README.md) <!-- nav -->

# Phase 12 — Computer systems

> **Stage 3 — How computers really work** · ⏱ about 4 weeks at ~10 hours per week

## What this phase is about

Until now you have written Rust and trusted the compiler and the computer to do the right thing. In this phase you open the box. You learn how numbers, floats and text really look as bits and bytes, how the CPU (Central Processing Unit) runs instructions, and how to read the assembly that the compiler produces for your functions, both on your Mac (arm64) and for x86-64 Linux servers.

Then you follow your data into memory: why caches make some loops ten times faster than others, how the compiler lays out a struct or an enum in memory, how a program is linked and loaded before `main` starts, and how a memory allocator hands out the heap memory that `Box`, `Vec` and `String` use. You finish by writing your own allocator.

Why does a systems engineer need this? Because the reasons for slow services, strange crashes and wrong numbers are often below the language: an integer that wrapped, a float that lost precision, a branch the CPU could not predict, a cache miss in a hot loop, a library loaded in the wrong version. After this phase you can look at the machine code and the memory and find such causes yourself, instead of guessing.

Every lesson has small, real experiments that you run on your own computer. The numbers you measure will differ from the ones in the lessons; what matters is that you can explain them.

## What you will be able to do

- [ ] Convert between binary, hex and decimal, explain two's complement, and choose the right overflow behaviour (`checked_*`, `wrapping_*`, `saturating_*`).
- [ ] Decode a float into sign, exponent and fraction, and compare, sort and sum floats correctly.
- [ ] Use masks, shifts and the built-in bit methods, and encode and decode UTF-8 by hand.
- [ ] Explain pipelines, instruction-level parallelism and branch prediction, and show their effect with a measurement.
- [ ] Read short arm64 and x86-64 assembly listings, including the calling convention and stack frames, and check what the compiler did with your code.
- [ ] Explain the memory hierarchy and write cache-friendly code; predict the size and alignment of Rust types.
- [ ] Inspect binaries on macOS and Linux and explain static and dynamic linking.
- [ ] Implement the `GlobalAlloc` trait with your own allocator and test it under Miri.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 12.1 | [Bits, bytes and number systems](01-bits-bytes-and-number-systems.md) | 120 min |
| 12.2 | [Integers in memory: two's complement and overflow](02-integers-in-memory.md) | 135 min |
| 12.3 | [Floating-point numbers](03-floating-point-numbers.md) | 120 min |
| 12.4 | [Bitwise operations and bit tricks](04-bitwise-operations.md) | 135 min |
| 12.5 | [Text encoding: ASCII, Unicode and UTF-8](05-text-encoding-ascii-and-utf8.md) | 120 min |
| 12.6 | [How the CPU runs instructions](06-how-the-cpu-runs-instructions.md) | 120 min |
| 12.7 | [Reading assembly: arm64 and x86-64](07-reading-assembly.md) | 135 min |
| 12.8 | [From Rust to machine code](08-from-rust-to-machine-code.md) | 135 min |
| 12.9 | [The memory hierarchy and caches](09-memory-hierarchy-and-caches.md) | 135 min |
| 12.10 | [Writing cache-friendly code](10-cache-friendly-code.md) | 2.5 h |
| 12.11 | [Memory layout of Rust types](11-memory-layout-of-rust-types.md) | 2.5 h |
| 12.12 | [Linking and loading programs](12-linking-and-loading.md) | 135 min |
| 12.13 | [How memory allocators work](13-how-memory-allocators-work.md) | 135 min |
| 12.14 | [Project: your own memory allocator](14-project-your-own-allocator.md) | 8 h |
| 12.15 | [Checkpoint](15-checkpoint.md) | 105 min |

## Projects in this phase

- **Practice packages** `code/p12-<topic>`, one per topic: `p12-bits` (Lessons 12.1–12.4), `p12-text` (12.5), `p12-cpu` (12.6) and `p12-asm` (12.7–12.8), and further packages in the later lessons. Each has tests for the exercises.
- **Project: your own memory allocator** (Lesson 12.14, package `code/p12-allocator`) — a bump allocator and a free-list allocator behind the `GlobalAlloc` trait, with tests that also run under Miri.
- **Checkpoint** (Lesson 12.15) — a self-test quiz and practical tasks before you move on to operating systems.

## How to study this phase

1. **Predict first, then run.** Before you run an example, write down what you expect: the bits, the output, the assembly, which version is faster. The moments when you were wrong are the ones you learn most from.
2. **Do the conversions by hand a few times.** Binary, hex, two's complement, a float, a UTF-8 character: work each one out on paper once or twice, then let Rust check you. After that, use the computer.
3. **Always measure release builds**, run every measurement several times, and compare ratios rather than exact times. Your Mac has performance and efficiency cores, and other programs run at the same time.
4. **Keep the assembly short.** Look at one small function at a time with `cargo asm`, and mark it `#[inline(never)]`. Use the tables in Lesson 12.7 as a cheat sheet until the common instructions are familiar.
5. **Keep the tests next to the tricks.** Every bit trick, faster loop or `unsafe` allocator in this phase has a simple, obvious version; test the clever one against it on many inputs.

---

[🏠 Start](../START-HERE.md) · [← 11.16 Checkpoint: Stage 2 complete](../11-linux-shell-and-containers/16-checkpoint-stage-2-complete.md) · [12.1 Bits, bytes and number systems →](01-bits-bytes-and-number-systems.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
