<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 12.8 From Rust to machine code](08-from-rust-to-machine-code.md) · [📋 Phase 12](README.md) · [12.10 Writing cache-friendly code →](10-cache-friendly-code.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/09-memory-hierarchy-and-caches.md) <!-- nav -->

# Lesson 12.9 — The memory hierarchy and caches

> **Phase 12 — Computer systems** · Lesson 9 of 15 · ⏱ about 135 minutes

## What you will learn

- Why a computer has several kinds of memory (registers, caches, RAM, disk) instead of one.
- What a cache and a cache line are, and what happens on a cache hit and a cache miss.
- How to read the cache sizes of your own Mac and of a Linux machine.
- How to measure the speed of each cache level yourself with a small Rust program.
- What locality and prefetching are, and why the *order* in which you touch memory matters so much.

## Before you start

- You finished [Lesson 12.8 — From Rust to machine code](08-from-rust-to-machine-code.md).
- You remember the table "how long things take" from [Lesson 00.3 — How a computer runs a program](../00-getting-started/03-how-a-computer-runs-a-program.md), and the stack and the heap from [Lesson 03.1](../03-ownership-and-borrowing/01-memory-stack-and-heap.md).
- You know how to time code honestly: release mode and `std::hint::black_box` from [Lesson 05.12 — Benchmarking basics](../05-rust-intermediate/12-benchmarking-basics.md).
- Create the package for this lesson and the next one. Every example is one file in `src/bin/`, and you run it with `cargo run --release --bin <name>`:

```bash
cd ~/projects/rust
cargo new code/p12-caches
mkdir code/p12-caches/src/bin
```

## New words

| Word | Meaning |
|---|---|
| **memory hierarchy** | The layers of memory in a computer, from tiny and very fast (registers) to huge and slow (disk). |
| **register** | A storage place inside the CPU (Central Processing Unit) core itself. Instructions work on registers directly. |
| **cache** | A small, fast memory that keeps copies of recently used data from a bigger, slower memory. |
| **L1, L2, L3** (Level 1, 2, 3) | The cache levels. L1 is the smallest and fastest and sits closest to the core; each next level is bigger and slower. |
| **SLC** (System Level Cache) | Apple's last cache level, shared by the CPU cores, the graphics chip and other parts of the chip. |
| **RAM** (Random Access Memory) | The main working memory of the computer (Lesson 00.3). Much bigger and much slower than the caches. |
| **SSD** (Solid-State Drive) | The disk of a modern computer, built from flash memory chips. It keeps data without power. |
| **SRAM / DRAM** (Static / Dynamic Random Access Memory) | Two kinds of memory chips. SRAM is fast and expensive and is used for caches; DRAM is cheaper and denser and is used for RAM. |
| **cache line** | The fixed-size block (64 or 128 bytes) that caches copy at once. A cache never copies a single byte. |
| **cache hit / cache miss** | A hit: the data is already in the cache. A miss: it is not, and must be fetched from a slower level. |
| **latency** | How long one access takes, from asking to getting the answer. |
| **bandwidth** | How many bytes per second can flow, when many accesses are in progress. |
| **locality** | The habit of programs to use the same data again soon (*temporal locality*) and data next to it (*spatial locality*). Caches work because of it. |
| **working set** | The amount of memory a piece of code touches during some period. If it fits in a cache, the code is fast. |
| **prefetching** | The CPU guessing which memory you will need next and loading it before you ask. |
| **pointer chasing** | Following a chain where each address is stored in the previous element, so each load must wait for the one before. |
| **KiB / MiB / GiB** (kibibyte, mebibyte, gibibyte) | 1 KiB = 1,024 bytes, 1 MiB = 1,024 KiB, 1 GiB = 1,024 MiB. |
| **ns** (nanosecond) | One billionth of a second. A modern core does about 4–5 clock cycles in 1 ns. |

## 1. Why there is a hierarchy

A memory can be fast, big or cheap, but not all three at once. Fast memory (SRAM) needs several transistors per bit and must sit very close to the core, so there is room for only a little of it. Big memory (DRAM) stores a bit in one tiny capacitor, so it is dense and cheap, but it sits on separate chips and reading it takes a long time. A disk is bigger and cheaper still, and much slower.

So a computer uses all of them, stacked by speed. Think of a kitchen: the knife in your hand (a register), the cutting board (L1), the counter (L2), the fridge (RAM), and the shop down the street (the disk). You keep what you use right now as close as possible.

```text
        faster, smaller, more expensive per byte
                        ▲
   registers   ~ 1 KiB per core        < 1 ns
   L1 cache    64–128 KiB per core     ~ 1 ns
   L2 cache    4–16 MiB per cluster    ~ 4–20 ns
   SLC / L3    tens of MiB             ~ 60 ns
   RAM         8–128 GiB               ~ 100–120 ns
   SSD disk    hundreds of GiB         ~ 20,000–100,000 ns
                        ▼
        slower, bigger, cheaper per byte
```

The numbers are typical values for an Apple M4 Pro; you will measure most of them yourself in section 4. The important part is the ratio: a miss that goes all the way to RAM costs about **100 times** more than a hit in L1. During those ~120 ns a core could have run several hundred instructions.

The caches are managed **by the hardware**, not by your program and not by the operating system. You never write "put this in L1". You only control *which addresses you touch and in which order*. That is exactly what this lesson and the next one are about.

## 2. Caches and cache lines

When an instruction loads a value from an address, the core asks its L1 cache first.

1. If L1 has the data, that is a **hit**: the value arrives in about 1 ns.
2. If not, that is a **miss**: L1 asks L2. If L2 misses too, it asks the next level, and finally RAM.
3. The answer travels back, and **every level on the way keeps a copy**, so the next access is a hit.

The cache does not copy just the value you asked for. It copies the whole **cache line** that contains it: an aligned block of 64 or 128 bytes. If you load the `u64` at address 1000, the cache brings in all the bytes of its line, including the neighbours.

```text
   RAM, cut into 128-byte lines

   line 6: addresses  768 ..  895   [........................]
   line 7: addresses  896 .. 1023   [.............X..........]  <- you load address 1000
   line 8: addresses 1024 .. 1151   [........................]

   after the miss, the L1 cache holds all of line 7 (128 bytes),
   so loading address 1008 or 1016 next is a hit
```

This is why caches work at all. Programs have **locality**:

- **Temporal locality** (in time): data used now is likely to be used again soon, like a loop counter or a value you read twice.
- **Spatial locality** (in space): data next to what you use now is likely to be used soon, like the next element of a `Vec`.

A cache is small, so when it is full and a new line arrives, an old line must leave (it is *evicted*). Usually the line that has not been used for the longest time goes. If your **working set** (the memory your loop keeps touching) is larger than a cache level, lines are evicted before you use them again, and you pay the miss over and over.

> ⚡ **Performance:** A `Vec<u64>` with 16 elements fills exactly one 128-byte line. Reading all 16 costs about one miss plus 15 hits. A linked list of 16 nodes scattered over the heap can cost 16 misses. That difference is measured in the next lesson.

## 3. The caches on your Mac

macOS reports the cache sizes through `sysctl`, a command that reads system information:

```bash
sysctl hw.cachelinesize hw.l1dcachesize hw.l2cachesize
sysctl hw.perflevel0.l1dcachesize hw.perflevel0.l2cachesize hw.perflevel1.l1dcachesize hw.perflevel1.l2cachesize
```

On the Mac used for this lesson (Apple M4 Pro) the output was:

```text
hw.cachelinesize: 128
hw.l1dcachesize: 65536
hw.l2cachesize: 4194304
hw.perflevel0.l1dcachesize: 131072
hw.perflevel0.l2cachesize: 16777216
hw.perflevel1.l1dcachesize: 65536
hw.perflevel1.l2cachesize: 4194304
```

**What happens:**

1. `hw.cachelinesize: 128` — the cache line is 128 bytes.
2. Apple chips have two kinds of cores. **Performance cores** (`perflevel0`) are fast and big; **efficiency cores** (`perflevel1`) are slower and use less power. They have different caches.
3. A performance core has 128 KiB of L1 data cache (`l1dcachesize`, the `d` means data; there is a separate cache for instructions). Its L2 cache is 16 MiB, shared by a group (cluster) of cores.
4. An efficiency core has 64 KiB of L1 data cache and shares a 4 MiB L2 cache. The short names `hw.l1dcachesize` and `hw.l2cachesize` show these smaller values.
5. There is no L3 line. Apple's last level is the SLC, and macOS does not report its size.

Your numbers may differ: an M1, an M3 Max or an Intel Mac print other values. That is fine; use your own numbers for the rest of the lesson.

On Linux, the command `getconf` prints the same kind of information, and `lscpu --caches` prints a table. Inside a container on this Mac:

```bash
docker run --rm ubuntu:26.04 getconf LEVEL1_DCACHE_LINESIZE
```

```text
64
```

The same chip reports 64 here. Linux on arm64 shows the smallest line size that the CPU announces about itself, while macOS reports 128, the line size of the L2 cache and of the blocks that move between L2 and memory. Different levels may use different line sizes, and Apple does not publish every detail. Section 5 shows that, for memory far away, 128 is the number that matters on this Mac. Most x86-64 chips use 64 bytes everywhere.

> 💡 **Tip:** The Linux VM (Virtual Machine) of Docker Desktop does not pass all cache sizes through, so `lscpu --caches` may print empty sizes in a container. On a real Linux server, or in your Lima VM `dev` from Phase 11, it prints the full table.

## 4. Measuring each level: pointer chasing

You can *see* the hierarchy with a program that loads from memory blocks of growing size. The trick is to make every load depend on the one before, so the CPU cannot start the next load early. The array holds a random cycle: `next[i]` is the index to visit after `i`. Create `src/bin/latency.rs`:

```rust
use std::hint::black_box;
use std::time::Instant;

/// A tiny pseudo-random number generator (xorshift), good enough for shuffling.
fn next_random(state: &mut u64) -> u64 {
    *state ^= *state << 13;
    *state ^= *state >> 7;
    *state ^= *state << 17;
    *state
}

/// Builds one big random cycle: next[i] is the index to visit after i.
fn random_cycle(len: usize) -> Vec<usize> {
    let mut order: Vec<usize> = (0..len).collect();
    let mut seed = 0x2545_F491_4F6C_DD1D;
    // Sattolo's shuffle: the result is a single cycle through all indexes.
    for i in (1..len).rev() {
        let j = (next_random(&mut seed) % i as u64) as usize;
        order.swap(i, j);
    }
    let mut next = vec![0; len];
    for k in 0..len {
        next[order[k]] = order[(k + 1) % len];
    }
    next
}

fn main() {
    const STEPS: usize = 20_000_000;
    println!("{:>10}  {:>8}", "size", "ns/load");
    for kib in [16, 64, 256, 1024, 8 * 1024, 32 * 1024, 128 * 1024, 512 * 1024] {
        let len = kib * 1024 / size_of::<usize>();
        let next = random_cycle(len);
        let mut i = 0;
        let start = Instant::now();
        for _ in 0..STEPS {
            i = next[i]; // each load depends on the previous one
        }
        let elapsed = start.elapsed();
        black_box(i);
        let ns = elapsed.as_nanos() as f64 / STEPS as f64;
        println!("{:>7} KiB  {:>8.1}", kib, ns);
    }
}
```

**What happens:**

1. `next_random` is xorshift, a very small pseudo-random generator: three shifts and XORs (exclusive OR, the `^` operator from Lesson 12.4) mix the bits of the state. It is not good for security, but it is fast and repeatable.
2. `random_cycle` shuffles the indexes `0..len` with *Sattolo's shuffle*: like an ordinary shuffle, but `j` is always smaller than `i`. This small change guarantees that following `next` visits **every** index once before it comes back to 0. Then `next[order[k]] = order[k + 1]` links the shuffled order into a ring.
3. For each size, `len` is the number of `usize` values (8 bytes each) that fill that many KiB.
4. The timed loop does `i = next[i]` twenty million times. The address of the next load is the *result* of this load, so the core must wait for each one. This is **pointer chasing**, and it measures pure latency.
5. Because the cycle is random, the next index is usually far away, in another cache line. Spatial locality does not help, so we see the level where the whole array fits.
6. `black_box(i)` makes the compiler believe the result is used, so it cannot delete the loop (Lesson 05.12).

Run it in release mode (it takes about ten seconds):

```bash
cd ~/projects/rust/code/p12-caches
cargo run --release --bin latency
```

```text
      size   ns/load
     16 KiB       1.0
     64 KiB       1.0
    256 KiB       4.1
   1024 KiB       6.3
   8192 KiB      10.9
  32768 KiB      67.5
 131072 KiB     112.3
 524288 KiB     123.1
```

Read the table next to the sizes from section 3 (the program ran on a performance core):

| Array size | Fits in | Time per load |
|---|---|---|
| 16 KiB, 64 KiB | L1 (128 KiB) | about 1 ns (4–5 cycles) |
| 256 KiB – 8 MiB | L2 (16 MiB) | 4–11 ns |
| 32 MiB | SLC, not L2 | about 65 ns |
| 128 MiB, 512 MiB | only RAM | 110–125 ns |

Every time the array outgrows a level, the time jumps. Inside L2 the time also grows slowly with size; part of that comes from translating addresses (the TLB, Translation Lookaside Buffer, a cache for address translations, which you meet in Phase 13).

> ⚠️ **Common mistake:** Expecting the same numbers on every run or every computer. A second run on the same Mac printed 1.8, 1.5, 4.6, 5.7, 25.6, 65.3, 112.4, 120.6. The shape is stable (small, then jumps at the cache sizes); single values move by tens of percent. Run more than once and look at the steps, not the decimals.

## 5. Locality: order matters

The pointer-chasing program was built to defeat the cache. Normal code is kinder, but the order of accesses still decides a lot. This program sums the same 128 MiB of numbers twice: once in order, once in a shuffled order. Create `src/bin/order.rs`:

```rust
use std::hint::black_box;
use std::time::Instant;

fn next_random(state: &mut u64) -> u64 {
    *state ^= *state << 13;
    *state ^= *state >> 7;
    *state ^= *state << 17;
    *state
}

fn sum_in_order(data: &[u64], order: &[usize]) -> u64 {
    let mut total = 0u64;
    for &i in order {
        total = total.wrapping_add(data[i]);
    }
    total
}

fn main() {
    let len = 16 * 1024 * 1024; // 16 Mi numbers of 8 bytes = 128 MiB
    let data: Vec<u64> = (0..len as u64).collect();

    let sequential: Vec<usize> = (0..len).collect();
    let mut shuffled = sequential.clone();
    let mut seed = 0x9E37_79B9_7F4A_7C15;
    for i in (1..len).rev() {
        let j = (next_random(&mut seed) % (i as u64 + 1)) as usize;
        shuffled.swap(i, j);
    }

    for (name, order) in [("sequential", &sequential), ("shuffled", &shuffled)] {
        let start = Instant::now();
        let total = sum_in_order(black_box(&data), black_box(order));
        let elapsed = start.elapsed();
        let ns = elapsed.as_nanos() as f64 / len as f64;
        println!("{name:>10}: {elapsed:>10.2?}  ({ns:.2} ns per number, sum {total})");
    }
}
```

**What happens:**

1. `sum_in_order` adds `data[i]` for every index in `order`. The work is identical in both runs; only the order of the indexes differs.
2. `shuffled` is an ordinary Fisher–Yates shuffle (`% (i + 1)`), because here we do not need a single cycle.
3. Both sums print the same total, which proves both runs did the same work.

```text
sequential:     4.63ms  (0.28 ns per number, sum 140737479966720)
  shuffled:    52.05ms  (3.10 ns per number, sum 140737479966720)
```

The shuffled order is about **11 times slower**. In order, one 128-byte line gives 16 numbers, so only one load in 16 can miss. Shuffled, almost every load misses.

The cache line itself becomes visible when you touch memory with a growing step. Create `src/bin/stride.rs`:

```rust
use std::hint::black_box;
use std::time::Instant;

fn main() {
    let len = 32 * 1024 * 1024; // 32 Mi numbers of 8 bytes = 256 MiB
    let mut data = vec![1u64; len];
    println!("{:>5}  {:>11}  {:>9}  {:>8}", "step", "bytes apart", "touched", "ns/touch");
    for step in [1, 2, 4, 8, 16, 32, 64, 128] {
        let start = Instant::now();
        let mut i = 0;
        while i < len {
            data[i] = data[i].wrapping_mul(3);
            i += step;
        }
        let elapsed = start.elapsed();
        black_box(&mut data);
        let touched = len / step;
        let ns = elapsed.as_nanos() as f64 / touched as f64;
        println!("{:>5}  {:>11}  {:>9}  {:>8.2}", step, step * 8, touched, ns);
    }
}
```

```text
 step  bytes apart    touched  ns/touch
    1            8   33554432      0.47
    2           16   16777216      0.47
    4           32    8388608      0.61
    8           64    4194304      1.02
   16          128    2097152      3.50
   32          256    1048576      3.79
   64          512     524288      3.91
  128         1024     262144      4.01
```

**What happens:**

1. With a step of 1, 2, 4 or 8 numbers, several touches fall into the same 128-byte line. They share one fetch from memory, so each touch is cheap.
2. From a step of 16 numbers (128 bytes) on, **every touch lands in a new line**. The cost per touch jumps to about 3.5 ns and then stays almost flat: you now pay for one whole line per touch, and a bigger step does not make a line more expensive.
3. The jump sits exactly at 128 bytes, the line size that macOS reported.

## 6. Prefetching and many loads at once

Something does not add up. A miss to RAM costs about 120 ns (section 4), but the stride program paid only about 3.5 ns per new line, and the shuffled sum only 3.1 ns per number. Two hardware features explain it.

**Prefetching.** The CPU watches the addresses you load. When it sees a regular pattern (address 0, 128, 256, 384, …), it starts fetching the next lines *before* you ask. When your loop gets there, the data is already in the cache. Walking forward through a `Vec` is the easiest pattern of all.

**Many loads at once.** A core does not have to wait for one load before starting the next, as long as it already knows the next address. In the shuffled sum, the indexes come from the `shuffled` array, so the core can send dozens of loads to memory at the same time, and their waiting times overlap. Total time ≈ latency ÷ number of loads in flight.

Pointer chasing defeats both: the next address is unknown until the current load finishes, and it is random. That is why it shows the real latency, and why data structures built from pointers (linked lists, trees of `Box`) can be slow, which is the subject of the next lesson.

```text
   independent loads (shuffled sum)       dependent loads (pointer chasing)

   load A ──────────────►                 load A ──────────────►
    load B ──────────────►                                       load B ──────────────►
     load C ──────────────►                                                            load C ──►
   time ─────────────────────►            time ─────────────────────────────────────────────►
```

> ⚡ **Performance:** The fastest memory access pattern is "forward, in order, through contiguous memory". The slowest is "follow a pointer to a random place, then follow the next one". Most cache-friendly code is about moving from the second pattern to the first.

> 🔒 **Security:** Caches are shared by programs, and their timing can leak information: an attacker measures how long a load takes to learn which lines another program used. Spectre and similar attacks work like this. Operating systems and browsers contain defences, which is one reason why precise timers are sometimes made less precise.

## Common mistakes

**1. Measuring in debug mode.** Running `cargo run --bin latency` without `--release` printed:

```text
      size   ns/load
     16 KiB       8.1
     64 KiB       7.8
    256 KiB       8.0
   1024 KiB       8.6
   8192 KiB      17.9
  32768 KiB      62.3
 131072 KiB     103.0
 524288 KiB     392.6
```

The L1 and L2 steps have vanished: the unoptimized loop itself takes 8 ns per step, which hides the 1 ns and 4 ns caches. Always measure memory effects with `--release`.

**2. A working set that is not what you think.** If you test "a big array" of 1 MiB, it still fits in L2, and you measure L2, not RAM. Compare your sizes with your own `sysctl` numbers before you draw conclusions.

**3. Letting the prefetcher fool you.** If you build the chain in order (`next[i] = i + 1`), the addresses are regular and the CPU predicts them. You measure the prefetcher, not the latency. Exercise 4 shows how big the difference is.

**4. Forgetting `black_box`.** Without `black_box(i)`, the compiler sees that `i` is never used after the loop and is allowed to delete the loop; the program would then report nonsense like 0.0 ns. In this exact program the loop happens to survive (the index check in `next[i]` could panic, and a panic is a visible effect), but a small change to the loop can remove that reason. `black_box` keeps the measurement honest no matter what.

## Try it yourself

1. Create `src/bin/latency.rs` from section 4 and run it twice with `cargo run --release --bin latency`. Write down where your jumps are and compare them with your `sysctl` sizes.
2. Now run the same program on the efficiency cores. macOS has a command, `taskpolicy`, that starts a program with a lower priority; such programs run on the efficiency cores:

```bash
cargo build --release --bin latency
taskpolicy -c background ./target/release/latency
```

On the Mac used for this lesson:

```text
      size   ns/load
     16 KiB       2.6
     64 KiB       2.6
    256 KiB       7.9
   1024 KiB      10.3
   8192 KiB     140.0
  32768 KiB     181.5
 131072 KiB     201.0
 524288 KiB     243.0
```

3. Explain the difference: the efficiency cores run at a lower clock speed (every level is slower), and their L2 is only 4 MiB, so the 8 MiB array already falls out to memory. This matches `hw.perflevel1.l2cachesize: 4194304` exactly.
4. Create `order.rs` and `stride.rs` and run them. Find the step where the cost per touch jumps on your Mac.

## Exercises

### Exercise 1 — Lines and numbers (easy)

Answer with arithmetic, then check the first answer with a tiny program that prints `128 / size_of::<u64>()`:

1. How many `u64` values fit in one 128-byte cache line? How many `u8`?
2. A `Vec<u32>` has 1,000,000 elements. How many 128-byte lines does its buffer cover, at least?
3. Which cache level of a performance core can hold that whole buffer?

<details>
<summary>Hint</summary>

A `u64` is 8 bytes, a `u32` is 4 bytes. Divide the total bytes by the line size and round up.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    println!("{}", 128 / size_of::<u64>());
    println!("{}", 128 / size_of::<u8>());
    println!("{}", (1_000_000 * size_of::<u32>()).div_ceil(128));
}
```

```text
16
128
31250
```

1. 16 `u64` values, 128 `u8` values.
2. 4,000,000 bytes ÷ 128 = 31,250 lines (more by one if the buffer does not start at a line border).
3. 4,000,000 bytes is about 3.8 MiB: too big for L1 (128 KiB), but it fits in the 16 MiB L2.

</details>

### Exercise 2 — Name the level (medium)

Write a function `fn level_for(bytes: usize) -> &'static str` that returns `"L1"`, `"L2"` or `"beyond L2"` for a working set of `bytes`, using the performance-core sizes 128 KiB and 16 MiB. Put it in `src/bin/levels.rs` with a `main` that prints the level for 16 KiB, 256 KiB and 32 MiB. Add tests for the borders: exactly 128 KiB is still L1, one byte more is L2.

<details>
<summary>Hint</summary>

Use constants such as `const L1: usize = 128 * 1024;` and `if bytes <= L1 { … } else if … `. Put the tests in a `#[cfg(test)] mod tests` at the bottom of the file and run `cargo test --bin levels`.

</details>

<details>
<summary>Answer</summary>

```rust
const L1: usize = 128 * 1024;
const L2: usize = 16 * 1024 * 1024;

fn level_for(bytes: usize) -> &'static str {
    if bytes <= L1 {
        "L1"
    } else if bytes <= L2 {
        "L2"
    } else {
        "beyond L2"
    }
}

fn main() {
    for kib in [16, 256, 32 * 1024] {
        println!("{kib} KiB -> {}", level_for(kib * 1024));
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn borders_belong_to_the_smaller_level() {
        assert_eq!(level_for(L1), "L1");
        assert_eq!(level_for(L1 + 1), "L2");
        assert_eq!(level_for(L2), "L2");
        assert_eq!(level_for(L2 + 1), "beyond L2");
    }
}
```

```text
16 KiB -> L1
256 KiB -> L2
32768 KiB -> beyond L2
```

Why it works: a working set that is exactly as big as the cache fits in it, so `<=` is right. In practice the real border is a little lower, because the cache also holds other data (the stack, the loop's own variables).

</details>

### Exercise 3 — Test the cycle (medium)

The latency program is only correct if `random_cycle` really makes **one** cycle through all indexes. If it made several small cycles, the walk would stay in a small part of the array and measure the wrong level. Write a test that, for the lengths 1, 2, 3, 10 and 1000, walks `len` steps from index 0 and checks that every index is visited exactly once and that the walk returns to 0.

<details>
<summary>Hint</summary>

Keep a `vec![false; len]` of visited flags. Before setting a flag, assert that it is still `false`.

</details>

<details>
<summary>Answer</summary>

Add this at the bottom of `src/bin/latency.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn random_cycle_visits_every_index_once() {
        for len in [1, 2, 3, 10, 1000] {
            let next = random_cycle(len);
            let mut seen = vec![false; len];
            let mut i = 0;
            for _ in 0..len {
                assert!(!seen[i], "index {i} visited twice (len {len})");
                seen[i] = true;
                i = next[i];
            }
            assert_eq!(i, 0, "the walk must come back to the start (len {len})");
            assert!(seen.iter().all(|&s| s), "some index was never visited (len {len})");
        }
    }
}
```

```text
running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(That is `cargo test -q --bin latency`.) Why it works: a walk of `len` steps that never repeats an index must have visited all `len` indexes, and arriving back at 0 closes the ring. Try changing `% i` to `% (i + 1)` in `random_cycle`: that is the ordinary shuffle, which can produce several small cycles, and the test fails.

</details>

### Exercise 4 — Beat the latency with order (hard)

Add a function `sequential_cycle(len)` that returns `next[i] = (i + 1) % len`. Chase both cycles over 512 MiB with the same timed loop and print both numbers. Explain the result.

<details>
<summary>Hint</summary>

Move the timed loop into `fn chase(next: &[usize], steps: usize) -> f64` that returns nanoseconds per load, and call it twice.

</details>

<details>
<summary>Answer</summary>

Create `src/bin/seq_chase.rs`. Copy `next_random` and `random_cycle` from `latency.rs` unchanged (shown here as `// …`), then add:

```rust
use std::hint::black_box;
use std::time::Instant;

// … next_random and random_cycle from latency.rs, unchanged …

fn sequential_cycle(len: usize) -> Vec<usize> {
    (0..len).map(|i| (i + 1) % len).collect()
}

fn chase(next: &[usize], steps: usize) -> f64 {
    let mut i = 0;
    let start = Instant::now();
    for _ in 0..steps {
        i = next[i];
    }
    let elapsed = start.elapsed();
    black_box(i);
    elapsed.as_nanos() as f64 / steps as f64
}

fn main() {
    const STEPS: usize = 20_000_000;
    let len = 512 * 1024 * 1024 / size_of::<usize>();
    let random = chase(&random_cycle(len), STEPS);
    let sequential = chase(&sequential_cycle(len), STEPS);
    println!("random cycle:     {random:.1} ns/load");
    println!("sequential cycle: {sequential:.1} ns/load");
}
```

```text
random cycle:     122.5 ns/load
sequential cycle: 0.3 ns/load
```

Why it works: the loads are still dependent, but the addresses are perfectly regular. The prefetcher streams the lines in long before they are needed, and recent Apple cores even predict the next load address before the previous load has finished. The same 512 MiB costs 400 times less per load. Order beats size.

</details>

## Quiz

1. Why does a computer not use only one fast kind of memory?

<details>
<summary>Answer</summary>

Fast memory (SRAM) is expensive and takes much space per bit, and it must sit close to the core. You cannot build gigabytes of it next to every core. So a small fast memory is combined with large slow ones.

</details>

2. What is a cache line, and how big is it on your Mac?

<details>
<summary>Answer</summary>

The fixed-size, aligned block that caches copy as one unit. macOS on Apple chips reports 128 bytes (`sysctl hw.cachelinesize`); most x86-64 chips use 64 bytes.

</details>

3. Your loop touches 2 MiB of data again and again on a performance core of an M4 Pro. Where do the accesses mostly hit?

<details>
<summary>Answer</summary>

In L2. 2 MiB is too big for the 128 KiB L1 but fits in the 16 MiB L2.

</details>

4. What is the difference between temporal and spatial locality? Give an example of each.

<details>
<summary>Answer</summary>

Temporal: the same data is used again soon (a counter in a loop, a lookup table used every iteration). Spatial: nearby data is used soon (the next element of an array or a `Vec`).

</details>

5. Why does the pointer-chasing program show about 120 ns per load for big arrays, while summing a shuffled array costs only about 3 ns per number?

<details>
<summary>Answer</summary>

In pointer chasing each address comes out of the previous load, so the loads happen one after another and each pays the full latency. In the shuffled sum, the addresses are known in advance (they are in the index array), so the core keeps many loads in flight at once and their waiting times overlap.

</details>

6. Who decides which data is in the L1 cache: your program, the operating system or the hardware?

<details>
<summary>Answer</summary>

The hardware. Your program only decides which addresses it touches and in which order; the cache keeps recently used lines automatically.

</details>

## Summary

- Memory is a hierarchy: registers, L1, L2, SLC or L3, RAM, disk. Each level is bigger and slower than the one before; an L1 hit takes about 1 ns, a trip to RAM about 120 ns.
- Caches copy whole **cache lines** (128 bytes on Apple chips, 64 on most x86-64 chips), never single bytes.
- Read your sizes with `sysctl hw.cachelinesize hw.l1dcachesize hw.l2cachesize` (and the `perflevel0`/`perflevel1` keys) on macOS, with `getconf` or `lscpu --caches` on Linux.
- Pointer chasing over a random cycle measures pure latency and shows a jump each time the working set outgrows a level.
- Locality makes caches work: in-order access to contiguous memory was 11 times faster than shuffled access to the same data.
- Prefetching and many loads in flight hide latency for regular or independent accesses; dependent, random accesses get no help.
- Measure memory effects in release mode, use `black_box`, run more than once, and expect other numbers on other machines.

## Sources and further reading

- [Ulrich Drepper — What Every Programmer Should Know About Memory](https://www.akkadia.org/drepper/cpumemory.pdf) — the classic long paper on RAM, caches and how to program for them.
- [Igor Ostrovsky — Gallery of Processor Cache Effects](https://igoro.com/archive/gallery-of-processor-cache-effects/) — short experiments similar to the ones in this lesson.
- [Apple — Apple Silicon CPU Optimization Guide](https://developer.apple.com/documentation/apple-silicon/cpu-optimization-guide) — Apple's official guide to the cores and caches of its chips.
- [Computer Systems: A Programmer's Perspective (CS:APP)](https://csapp.cs.cmu.edu/) — chapter 6 explains the memory hierarchy in depth.
- [std::hint::black_box](https://doc.rust-lang.org/std/hint/fn.black_box.html) and [std::time::Instant](https://doc.rust-lang.org/std/time/struct.Instant.html) — the two tools used for every measurement here.
- [getconf(1p)](https://man7.org/linux/man-pages/man1/getconf.1p.html) and [lscpu(1)](https://man7.org/linux/man-pages/man1/lscpu.1.html) — reading cache information on Linux.
- [Fisher–Yates shuffle (Wikipedia)](https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle) — the shuffle used here, and Sattolo's variant that makes a single cycle.

---

[🏠 Start](../START-HERE.md) · [← 12.8 From Rust to machine code](08-from-rust-to-machine-code.md) · [📋 Phase 12](README.md) · [12.10 Writing cache-friendly code →](10-cache-friendly-code.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
