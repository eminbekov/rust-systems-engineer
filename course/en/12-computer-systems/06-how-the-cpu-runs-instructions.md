<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 12.5 Text encoding: ASCII, Unicode and UTF-8](05-text-encoding-ascii-and-utf8.md) · [📋 Phase 12](README.md) · [12.7 Reading assembly: arm64 and x86-64 →](07-reading-assembly.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/06-how-the-cpu-runs-instructions.md) <!-- nav -->

# Lesson 12.6 — How the CPU runs instructions

> **Phase 12 — Computer systems** · Lesson 6 of 15 · ⏱ about 120 minutes

## What you will learn

- What an instruction set architecture is, and the main facts about arm64 (your Mac) and x86-64 (most servers).
- How a pipeline lets the CPU work on many instructions at the same time, and what latency and throughput mean.
- Why independent work runs faster than a chain of dependent steps (instruction-level parallelism), shown with a real measurement.
- How branch prediction works, and how a wrong guess can make the same loop ten times slower.
- Why benchmarks on your Mac can change between runs (performance and efficiency cores, other programs).

## Before you start

- You finished [Lesson 12.5 — Text encoding](05-text-encoding-ascii-and-utf8.md).
- You remember the fetch–decode–execute cycle, registers and the program counter from [Lesson 00.3 — How a computer runs a program](../00-getting-started/03-how-a-computer-runs-a-program.md). That was the simple model; this lesson shows what a modern CPU really does.
- You know how to time code with `std::time::Instant` and `std::hint::black_box`, and that measurements need `--release` ([Lesson 05.12 — Benchmarking basics](../05-rust-intermediate/12-benchmarking-basics.md)).
- Create the package: `cargo new code/p12-cpu`. All examples here are measurements, so always run them with `cargo run --release`.

## New words

| Word | Meaning |
|---|---|
| **ISA** (Instruction Set Architecture) | The list of instructions a CPU family understands, its registers and its rules. It is the contract between compiled programs and the hardware. |
| **arm64 / AArch64** | The 64-bit ISA of Arm processors, used by Apple M-series Macs, phones and many cloud servers. The two names mean the same thing. |
| **x86-64** | The 64-bit ISA of Intel and AMD (Advanced Micro Devices) processors, used by most PCs (personal computers) and servers. Also called amd64. |
| **general-purpose register** | A register for ordinary integer values and addresses. arm64 has 31 of them (`x0`–`x30`), x86-64 has 16. |
| **load / store** | An instruction that copies data from memory into a register / from a register into memory. |
| **clock cycle** | One tick of the CPU's clock. Several billion per second; the basic unit of time inside a CPU. |
| **pipeline** | Splitting the work on an instruction into stages, so that different instructions can be in different stages at once. |
| **latency** | How long one operation takes from start to result. |
| **throughput** | How many operations finish per unit of time when many are in flight. |
| **dependency chain** | A series of instructions where each one needs the result of the one before. It cannot be sped up by parallel work. |
| **ILP** (Instruction-Level Parallelism) | Running several independent instructions of one program at the same time. |
| **out-of-order execution** | The CPU runs instructions as soon as their inputs are ready, not strictly in program order, and then makes the results appear in order. |
| **branch** | An instruction that may jump to another place in the code, for example for an `if` or a loop. |
| **branch prediction** | The CPU guesses which way a branch goes before it knows, and continues working on the guess. |
| **misprediction** | A wrong guess: the CPU throws away the work done on the wrong path and starts again. It costs many cycles. |

## 1. The instruction set: the CPU's language

In Lesson 00.3 you met a toy computer with four instructions. A real CPU family has hundreds, and their list, together with the registers and the rules, is the **ISA** (Instruction Set Architecture). Machine code built for one ISA does not run on another one. Two ISAs matter for you:

| | arm64 (AArch64): your Mac | x86-64: most Linux servers |
|---|---|---|
| General-purpose registers | 31 of 64 bits: `x0`–`x30` (their lower 32 bits are called `w0`–`w30`) | 16 of 64 bits: `rax`, `rbx`, `rcx`, `rdx`, `rsi`, `rdi`, `rbp`, `rsp`, `r8`–`r15` |
| Special registers | `sp` (stack pointer), `pc` (program counter), flags `N Z C V` | `rsp` (stack pointer), `rip` (instruction pointer), flags |
| Instruction length | always 4 bytes | 1 to 15 bytes |
| Memory access | only special load/store instructions (`ldr`, `str`) touch memory; arithmetic works on registers | many instructions can read or write memory directly |

arm64 is a **load/store** design: to add a number from memory, the CPU first loads it into a register, then adds registers, then stores the result if needed. All instructions have the same length, which makes them easy to decode quickly. Lesson 12.7 teaches you to read both kinds of assembly.

The **clock** of a CPU ticks a few billion times per second (a few gigahertz, GHz). One tick is a **clock cycle**, a fraction of a nanosecond. In the simple model of Lesson 00.3, one instruction took one full fetch–decode–execute round. A modern core does far better than "one instruction per cycle", and the rest of this lesson explains how.

## 2. The pipeline: an assembly line for instructions

Think of washing clothes: washing takes an hour, drying an hour, folding an hour. One load takes three hours. But you do not wait three hours before you start the next load: while load 1 is in the dryer, load 2 is already in the washer. With three loads in different stages, a finished load comes out **every hour**, even though each load still takes three hours.

A CPU does the same with instructions. Here is a simple 4-stage pipeline (real cores have more stages):

```text
cycle:          1      2      3      4      5      6      7
instruction 1   fetch  decode exec   write
instruction 2          fetch  decode exec   write
instruction 3                 fetch  decode exec   write
instruction 4                        fetch  decode exec   write
```

Each instruction still needs 4 cycles: that is its **latency**. But from cycle 4 on, one instruction finishes every cycle: that is the **throughput**. Modern cores go further. They are **superscalar**: they fetch and execute **several** instructions per cycle, with several adders, multipliers and load units working side by side. And they execute **out of order**: an instruction runs as soon as its inputs are ready, even if an earlier instruction is still waiting for memory. The results are then put back in program order, so the program cannot tell the difference.

All this has one condition: the instructions must be **independent**. If instruction 2 needs the result of instruction 1, it has to wait, no matter how many adders the CPU has.

## 3. Dependency chains: a measurement

Adding numbers with one running sum is a **dependency chain**: each `sum += v` needs the previous `sum`. You learned in Lesson 12.3 that the compiler may not reorder float additions, so it must do them one after another, exactly as written. If we keep **four** separate sums, the four chains are independent, and the CPU can run them side by side. Put this into `src/main.rs`:

```rust
use std::hint::black_box;
use std::time::Instant;

fn sum_one_chain(values: &[f64]) -> f64 {
    let mut sum = 0.0;
    for &v in values {
        sum += v;
    }
    sum
}

fn sum_four_chains(values: &[f64]) -> f64 {
    let mut s = [0.0; 4];
    let chunks = values.chunks_exact(4);
    let rest = chunks.remainder();
    for c in chunks {
        s[0] += c[0];
        s[1] += c[1];
        s[2] += c[2];
        s[3] += c[3];
    }
    let mut sum = (s[0] + s[1]) + (s[2] + s[3]);
    for &v in rest {
        sum += v;
    }
    sum
}

fn main() {
    let values: Vec<f64> = (0..10_000_000).map(|i| (i % 100) as f64 * 0.5).collect();
    for round in 1..=3 {
        let start = Instant::now();
        let a = sum_one_chain(black_box(&values));
        let one = start.elapsed();
        let start = Instant::now();
        let b = sum_four_chains(black_box(&values));
        let four = start.elapsed();
        println!("round {round}: one chain {one:>9.2?} (sum {a}), four chains {four:>9.2?} (sum {b})");
    }
}
```

**What happens:**

1. The vector has 10 million `f64` values. They are all multiples of 0.5 and the sums stay small, so every addition is exact and both functions give the same result. (With other data, the two results could differ slightly, because the order of additions is different — Lesson 12.3.)
2. `chunks_exact(4)` gives slices of exactly 4 values; `remainder()` is what is left at the end (0 to 3 values). Each of the four positions goes into its own sum `s[0]`…`s[3]`.
3. `black_box` hides the input from the optimiser, so the compiler cannot compute the sum at compile time.
4. We measure three rounds. The first round is often slower (memory is touched for the first time, the core may still be waking up), so we look at the later rounds.

Run `cargo run --release`. Real output from an Apple M4 Pro (your numbers will be different, but the ratio should be similar):

```text
round 1: one chain    8.39ms (sum 247500000), four chains    2.90ms (sum 247500000)
round 2: one chain    5.98ms (sum 247500000), four chains    1.48ms (sum 247500000)
round 3: one chain    5.34ms (sum 247500000), four chains    1.47ms (sum 247500000)
```

The four-chain version is about **3.6 times faster**, though it does exactly the same number of additions. With one chain, every addition waits for the previous one: 10 million additions in about 5.5 ms is about 0.55 ns each, which is the **latency** of one float addition on this core. With four chains, four additions are in flight at once, and the speed approaches the **throughput** of the adders. This is **instruction-level parallelism** (ILP).

> ⚡ **Performance:** You rarely write four accumulators by hand. For integers the compiler does it for you (integer addition can be reordered without changing the result), and it even uses SIMD (Single Instruction, Multiple Data) instructions that add several numbers in one instruction. For floats you must allow a different order yourself, as here, or use a library that does. Lesson 19.7 covers SIMD.

## 4. Branch prediction

A pipeline needs to know which instruction comes next **before** the current one is finished. After a **branch** (an `if`, the end of a loop) the next instruction depends on a result that is not ready yet. Waiting would stop the pipeline. So the CPU **predicts**: it guesses the direction from the history of this branch, and continues on the guess. If the guess was right, no time is lost. If it was wrong (a **misprediction**), the CPU throws away all the work on the wrong path and starts again from the right instruction. That costs many cycles.

Branch predictors are very good at regular patterns ("this loop goes back 999 times, then exits") and helpless with random data. Let us measure: we add up all bytes that are at least 128, once in random order and once sorted.

```rust
use std::hint::black_box;
use std::time::Instant;

fn random_bytes(n: usize) -> Vec<u8> {
    let mut state: u64 = 0x2545_f491_4f6c_dd1d;
    (0..n)
        .map(|_| {
            state ^= state << 13;
            state ^= state >> 7;
            state ^= state << 17;
            (state >> 56) as u8
        })
        .collect()
}

fn sum_big(values: &[u8]) -> u64 {
    let mut sum = 0u64;
    for &v in values {
        if v >= 128 {
            sum += v as u64;
        }
    }
    sum
}

fn sum_big_branchy(values: &[u8]) -> u64 {
    let mut sum = 0u64;
    for &v in values {
        if v >= 128 {
            sum += black_box(v) as u64;
        }
    }
    sum
}

fn main() {
    let random = random_bytes(10_000_000);
    let mut sorted = random.clone();
    sorted.sort_unstable();
    for round in 1..=3 {
        let start = Instant::now();
        let a = sum_big(black_box(&random));
        let t_random = start.elapsed();
        let start = Instant::now();
        let b = sum_big(black_box(&sorted));
        let t_sorted = start.elapsed();
        println!("round {round}: random {t_random:>9.2?}, sorted {t_sorted:>9.2?} (sums {a} {b})");
        let start = Instant::now();
        let c = sum_big_branchy(black_box(&random));
        let t_random = start.elapsed();
        let start = Instant::now();
        let d = sum_big_branchy(black_box(&sorted));
        let t_sorted = start.elapsed();
        println!("   branchy: random {t_random:>9.2?}, sorted {t_sorted:>9.2?} (sums {c} {d})");
    }
}
```

**What happens:**

1. `random_bytes` makes pseudo-random bytes with a tiny generator called *xorshift*: it mixes a 64-bit state with shifts and XOR (exclusive OR, the `^` operator from Lesson 12.4) and takes the top 8 bits. It is not good for security, but it is fast, needs no crate and gives the same bytes on every run.
2. In the random data, `v >= 128` is true about half of the time, with no pattern. In the sorted data, it is false for the first half and true for the second half: a perfect pattern.
3. `sum_big` is the natural code. `sum_big_branchy` is the same, but the `black_box` inside the `if` forces the compiler to keep a real branch (it must not call `black_box` when the condition is false).

Real output (your numbers will differ):

```text
round 1: random    1.15ms, sorted    1.00ms (sums 957728160 957728160)
   branchy: random   30.95ms, sorted    2.78ms (sums 957728160 957728160)
round 2: random  735.00µs, sorted  635.83µs (sums 957728160 957728160)
   branchy: random   31.60ms, sorted    3.34ms (sums 957728160 957728160)
round 3: random  887.88µs, sorted    1.20ms (sums 957728160 957728160)
   branchy: random   37.84ms, sorted    2.75ms (sums 957728160 957728160)
```

Two lessons are hidden in these numbers:

- **With a real branch, random data is about 11 times slower.** About 5 million guesses are wrong, and each costs a few nanoseconds. On sorted data the predictor is almost always right.
- **The natural code has no branch at all.** `sum_big` is fast for both inputs. The compiler replaced the `if` with branch-free instructions ("add `v` or add 0") and even processed many bytes per instruction with SIMD. It is also 30 to 40 times faster than the branchy version on random data. Lesson 12.7 shows the instructions it used, and Lesson 12.8 shows how to check what the compiler did.

> 💡 **Tip:** Do not remove branches by hand "just in case". First measure; then look at the assembly. The compiler often removes branches itself, and a branch that is predictable costs almost nothing.

> 🔒 **Security:** Prediction has a dark side. In 2018, the attacks called Spectre and Meltdown showed that work done on a wrongly predicted path, although thrown away, leaves traces in the caches, and an attacker can measure those traces to read secret data. Operating systems, browsers and compilers added protections. The lesson for you: code that handles secrets (passwords, keys) should not branch on the secret or use it as an index (Phase 10 called this *constant-time* code).

## 5. Why your measurements jump around

Look at the numbers above again: the same function took 735 µs in one round and 888 µs in the next. Many things change timing:

```bash
sysctl hw.perflevel0.name hw.perflevel0.physicalcpu hw.perflevel1.name hw.perflevel1.physicalcpu
```

```text
hw.perflevel0.name: Performance
hw.perflevel0.physicalcpu: 8
hw.perflevel1.name: Efficiency
hw.perflevel1.physicalcpu: 4
```

- An M4 Pro has **two kinds of cores**: 8 performance cores and 4 efficiency cores (yours may have a different number). The efficiency cores are slower but use less power. macOS decides where your program runs, and it can move it.
- The clock speed changes with load and temperature.
- Other programs share the cores, the caches and the memory.

So: run several rounds, ignore the first, compare **ratios** inside one run rather than absolute times between runs, and repeat a measurement before you believe it. Lesson 09.11 introduces tools that do this statistics for you.

> 🧪 **Testing:** Before you measure a faster version of a function, write a test that checks that it gives the same result as the simple version, for many inputs (and for edge cases such as an empty slice or a length that is not a multiple of 4). A fast wrong answer is worthless. Exercises 2 and 3 do this.

## Common mistakes

> ⚠️ **Common mistake: measuring a debug build.** `cargo run` without `--release` compiles without optimisation. The branch experiment then shows completely different numbers, because the compiler keeps every branch and adds overflow checks. Always measure release builds.

> ⚠️ **Common mistake: letting the compiler delete the work.** If a result is never used, the optimiser may remove the whole computation, and you measure an empty loop. Use the result (print it) and pass inputs through `black_box`.

> ⚠️ **Common mistake: trusting one run.** One measurement can be off by 30% or more on a busy machine. Repeat, and look for a stable pattern.

## Try it yourself

1. Change `sum_four_chains` into an eight-chain version. Is it faster than four chains on your Mac? (At some point there are more chains than adders, and the speed stops growing.)
2. In the branch program, make the data "almost sorted": sort it, then swap 1% of random pairs. How much slower is the branchy version than on fully sorted data?
3. Run the branch program with `cargo run` (debug). Compare all four numbers with the release run.

## Exercises

### Exercise 1 — Count the cycles (easy)

A CPU has a 4-stage pipeline like the one in Section 2, finishes one instruction per cycle once it is full, and never stalls. How many cycles do 6 independent instructions take? And 1000?

<details>
<summary>Hint</summary>

Draw the table for 6 instructions. The first instruction finishes after 4 cycles; after that, one more finishes in every cycle.

</details>

<details>
<summary>Answer</summary>

6 instructions take 4 + 5 = **9 cycles**; 1000 take 4 + 999 = **1003 cycles**. The general rule is *stages + instructions − 1*. Without a pipeline they would take 6 × 4 = 24 and 4000 cycles.

Why it works: filling the pipeline costs `stages − 1` extra cycles once; after that, the throughput is one instruction per cycle.

</details>

### Exercise 2 — Two chains, with a test (medium)

In `src/lib.rs`, write `pub fn sum_one_chain(values: &[f64]) -> f64` and `pub fn sum_two_chains(values: &[f64]) -> f64`. Write a test that checks that both give the same result for every length from 0 to 20 (use values `i as f64 * 0.5`, which add up exactly). Then time both in `src/main.rs` with 10 million values.

<details>
<summary>Hint</summary>

Use `chunks_exact(2)` and its `remainder()`. In `src/main.rs` you can call the library functions as `p12_cpu::sum_two_chains` (the package name with `_`).

</details>

<details>
<summary>Answer</summary>

```rust
/// Adds the values with one running sum (one dependency chain).
pub fn sum_one_chain(values: &[f64]) -> f64 {
    let mut sum = 0.0;
    for &v in values {
        sum += v;
    }
    sum
}

/// Adds the values with two independent running sums.
pub fn sum_two_chains(values: &[f64]) -> f64 {
    let mut a = 0.0;
    let mut b = 0.0;
    let pairs = values.chunks_exact(2);
    let rest = pairs.remainder();
    for p in pairs {
        a += p[0];
        b += p[1];
    }
    let mut sum = a + b;
    for &v in rest {
        sum += v;
    }
    sum
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn two_chains_match_one_chain_for_every_length() {
        for len in 0..=20 {
            let values: Vec<f64> = (0..len).map(|i| i as f64 * 0.5).collect();
            assert_eq!(sum_two_chains(&values), sum_one_chain(&values), "len {len}");
        }
    }
}
```

Example timing on an M4 Pro (three rounds):

```text
one   6.75ms  two   2.66ms  (247500000 247500000)
one   5.53ms  two   2.72ms  (247500000 247500000)
one   5.60ms  two   2.69ms  (247500000 247500000)
```

Why it works: two independent chains let two additions be in flight at once, so the time roughly halves. The test covers every remainder case (odd and even lengths, and the empty slice); the values are chosen so that the sums are exact, which makes `assert_eq!` on floats safe here.

</details>

### Exercise 3 — Remove the branch yourself (medium)

Write `pub fn sum_big_branchless(values: &[u8]) -> u64` that sums the bytes ≥ 128 **without** an `if` in the loop. Test it against a simple iterator version for all 256 byte values. Then time it on random and on sorted data: is there still a difference?

<details>
<summary>Hint</summary>

A `bool` converts to an integer with `as`: `true as u64` is 1, `false as u64` is 0. Multiply the value by that.

</details>

<details>
<summary>Answer</summary>

```rust
/// Sums the bytes that are at least 128, without an `if` in the loop.
pub fn sum_big_branchless(values: &[u8]) -> u64 {
    let mut sum = 0u64;
    for &v in values {
        let keep = (v >= 128) as u64; // 1 or 0
        sum += keep * v as u64;
    }
    sum
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn branchless_matches_simple_version() {
        let values: Vec<u8> = (0..=255u8).chain([0, 128, 127, 255]).collect();
        let expected: u64 = values.iter().filter(|&&v| v >= 128).map(|&v| v as u64).sum();
        assert_eq!(sum_big_branchless(&values), expected);
        assert_eq!(sum_big_branchless(&[]), 0);
    }
}
```

Example timing (10 million bytes, three rounds):

```text
branchless random 557.62µs sorted 509.58µs (957728160 957728160)
branchless random 508.92µs sorted 509.12µs (957728160 957728160)
branchless random 509.42µs sorted 509.12µs (957728160 957728160)
```

Why it works: there is no branch, so there is nothing to predict, and the order of the data no longer matters. The comparison becomes an instruction that produces 0 or 1. The speed is the same as the compiler's own version of `sum_big` from Section 4, because the compiler had already done this transformation.

</details>

### Exercise 4 — How much can the predictor learn? (hard)

Branch predictors remember patterns. Build data in which the branch pattern repeats: take a block of `period` random bytes and repeat it until you have 10 million bytes. Time `sum_big_branchy` for periods 2, 16, 256, 4096 and 65536, and for fully random data. Where does the predictor stop helping?

<details>
<summary>Hint</summary>

`(0..n).map(|i| random[i % period]).collect()` repeats the first `period` bytes of `random`. Reuse `random_bytes` and `sum_big_branchy` from Section 4.

</details>

<details>
<summary>Answer</summary>

```rust
use std::hint::black_box;
use std::time::Instant;

fn random_bytes(n: usize) -> Vec<u8> {
    let mut state: u64 = 0x2545_f491_4f6c_dd1d;
    (0..n)
        .map(|_| {
            state ^= state << 13;
            state ^= state >> 7;
            state ^= state << 17;
            (state >> 56) as u8
        })
        .collect()
}

fn sum_big_branchy(values: &[u8]) -> u64 {
    let mut sum = 0u64;
    for &v in values {
        if v >= 128 {
            sum += black_box(v) as u64;
        }
    }
    sum
}

fn main() {
    let n = 10_000_000;
    let random = random_bytes(n);
    for period in [2usize, 16, 256, 4096, 65536] {
        // A random block of `period` bytes, repeated: the branch pattern repeats every `period` steps.
        let data: Vec<u8> = (0..n).map(|i| random[i % period]).collect();
        let start = Instant::now();
        let s = sum_big_branchy(black_box(&data));
        println!("period {period:>6}: {:>8.2?} (sum {s})", start.elapsed());
    }
    let start = Instant::now();
    let s = sum_big_branchy(black_box(&random));
    println!("no period   : {:>8.2?} (sum {s})", start.elapsed());
}
```

Example output on an M4 Pro:

```text
period      2:   2.50ms (sum 1140000000)
period     16:   2.79ms (sum 592500000)
period    256:   3.97ms (sum 898944401)
period   4096:   6.55ms (sum 940508798)
period  65536:  25.13ms (sum 953348632)
no period   :  26.13ms (sum 957728160)
```

Why it works: the predictor keeps a history of recent branch directions and learns which direction follows which history. Short patterns are learned completely; a pattern of 4096 steps is learned partly; a pattern of 65536 steps is too long to remember and looks random. The exact limits are secret details of each CPU, which is why you measure instead of guessing.

</details>

## Quiz

1. What is an ISA, and why can an arm64 program not run on an x86-64 CPU?

<details>
<summary>Answer</summary>

The Instruction Set Architecture is the set of instructions, registers and rules of a CPU family. The two ISAs encode instructions differently, so machine code for one means nothing (or something else) to the other.

</details>

2. What is the difference between latency and throughput?

<details>
<summary>Answer</summary>

Latency is how long one operation takes from start to result. Throughput is how many operations finish per unit of time when many are in flight. A pipeline improves throughput, not latency.

</details>

3. Why is the one-chain sum slower than the four-chain sum, although both do the same number of additions?

<details>
<summary>Answer</summary>

In one chain, each addition must wait for the result of the previous one, so only one is in flight at a time. Four chains are independent, so the CPU runs four additions at once.

</details>

4. Why does the compiler not turn the one-chain float sum into four chains by itself?

<details>
<summary>Answer</summary>

Float addition rounds, so a different order can give a different result. The compiler must keep the order you wrote. (For integers it may reorder, and it does.)

</details>

5. What happens on a branch misprediction?

<details>
<summary>Answer</summary>

The CPU had continued on the guessed path. It discards that work and restarts from the correct instruction, which costs many cycles.

</details>

6. Why did `sum_big` run equally fast on random and on sorted data?

<details>
<summary>Answer</summary>

The compiler replaced the `if` with branch-free instructions (and SIMD), so there was nothing to predict.

</details>

7. Name two reasons why the same benchmark gives different times on your Mac.

<details>
<summary>Answer</summary>

For example: the program may run on a performance core or an efficiency core; the clock speed changes with load and temperature; other programs compete for the cores, caches and memory; the first round touches memory for the first time.

</details>

## Summary

- The ISA is the CPU's language. arm64 has 31 general-purpose registers, fixed 4-byte instructions and separate load/store instructions; x86-64 has 16 registers and variable-length instructions.
- A pipeline overlaps the stages of many instructions; superscalar, out-of-order cores run several independent instructions per cycle.
- Latency is the time of one operation; throughput is the rate of many. A dependency chain is limited by latency; independent work reaches throughput.
- Splitting a float sum into four independent sums made it about 3.6 times faster on an M4 Pro.
- The CPU predicts branches. Unpredictable branches can make a loop about 11 times slower; the compiler often removes branches itself.
- Measure release builds, keep the work alive with `black_box`, repeat runs, compare ratios, and test that a faster version gives the same results.

## Sources and further reading

- [Putting the "You" in CPU](https://cpu.land/) — a friendly, free online explanation of how a CPU runs programs.
- [Arm: Learn the architecture — AArch64 Instruction Set Architecture](https://developer.arm.com/documentation/102374/latest/) — the official beginner guide to arm64 registers and instructions.
- [Rust standard library: `std::hint::black_box`](https://doc.rust-lang.org/std/hint/fn.black_box.html) — what `black_box` does and does not guarantee.
- [Dan Luu: Branch prediction](https://danluu.com/branch-prediction/) — a clear article on how branch predictors work, from simple to modern designs.
- [Computer Systems: A Programmer's Perspective (CS:APP)](https://csapp.cs.cmu.edu/) — chapter 4 (pipelining) and chapter 5.7 (understanding modern processors).

---

[🏠 Start](../START-HERE.md) · [← 12.5 Text encoding: ASCII, Unicode and UTF-8](05-text-encoding-ascii-and-utf8.md) · [📋 Phase 12](README.md) · [12.7 Reading assembly: arm64 and x86-64 →](07-reading-assembly.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
