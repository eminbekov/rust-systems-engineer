<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 12.9 The memory hierarchy and caches](09-memory-hierarchy-and-caches.md) · [📋 Phase 12](README.md) · [12.11 Memory layout of Rust types →](11-memory-layout-of-rust-types.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/12-computer-systems/10-cache-friendly-code.md) <!-- nav -->

# Lesson 12.10 — Writing cache-friendly code

> **Phase 12 — Computer systems** · Lesson 10 of 15 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- Four practical rules that make code friendly to the cache, each with a measurement.
- Why the order of two nested loops can change the speed 35 times.
- Why `Vec<T>` beats `LinkedList<T>` and `Vec<Box<T>>` when you walk through the data.
- What "array of structs" and "struct of arrays" mean, and when to split hot and cold data.
- What false sharing is, and how to stop two threads from fighting over one cache line.

## Before you start

- You finished [Lesson 12.9 — The memory hierarchy and caches](09-memory-hierarchy-and-caches.md) and have the package `code/p12-caches`.
- You know threads, `thread::scope` and atomic counters from [Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md), and linked lists from [Lesson 07.4](../07-algorithms-and-data-structures/04-linked-lists.md).
- As before, every example is a file in `src/bin/`, run with `cargo run --release --bin <name>`.

## New words

| Word | Meaning |
|---|---|
| **cache-friendly** | Code that touches memory in an order the caches handle well: in order, close together, with a small working set. |
| **row-major order** | Storing a grid row by row in one flat buffer: all of row 0, then all of row 1, and so on. Element `(row, col)` is at index `row * width + col`. |
| **indirection** | Reaching data through a pointer instead of directly. Each indirection can be one more cache miss. |
| **AoS** (Array of Structs) | A `Vec` of whole records, such as `Vec<Particle>`. All fields of one record sit together. |
| **SoA** (Struct of Arrays) | One `Vec` per field, such as `Particles { mass: Vec<f64>, charge: Vec<f64>, … }`. All values of one field sit together. |
| **hot / cold data** | Hot data is used on every pass of a frequent loop; cold data is used rarely. Keeping them apart keeps the working set small. |
| **cache coherence** | The hardware rule that all cores see one consistent value for each memory address, even though each core has its own cache. |
| **false sharing** | Two threads write *different* variables that happen to lie in the *same* cache line, so the line moves between their cores all the time. |
| **SIMD** (Single Instruction, Multiple Data) | CPU (Central Processing Unit) instructions that process several numbers at once. The compiler uses them for simple loops over contiguous data (Phase 19 goes deeper). |
| **L1, L2, RAM, KiB, MiB** | Reminder from Lesson 12.9: L1 and L2 (Level 1 and Level 2) are the cache levels, RAM (Random Access Memory) is the main memory; 1 KiB (kibibyte) = 1,024 bytes, 1 MiB (mebibyte) = 1,024 KiB. |
| **MB** (megabyte) | One million bytes, a little less than 1 MiB. Used here for sizes computed in round numbers, such as 10,000,000 × 8 bytes = 80 MB. |
| **µs / ms** (microsecond / millisecond) | A millionth / a thousandth of a second. 1 ms = 1,000 µs; 1 µs = 1,000 ns (nanoseconds). |

## 1. A helper for honest timing

Single measurements are noisy: other programs, the operating system and the CPU clock change the result from run to run. A simple fix is to run the work several times and keep the **fastest** run, which is the one least disturbed. Put this helper in `src/lib.rs` of `code/p12-caches`, so every program in `src/bin/` can use it as `p12_caches::best_of_5`:

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

/// Runs `work` five times and returns the fastest time and the result.
pub fn best_of_5<T>(mut work: impl FnMut() -> T) -> (Duration, T) {
    let mut best = Duration::MAX;
    let mut result = None;
    for _ in 0..5 {
        let start = Instant::now();
        let value = black_box(work());
        best = best.min(start.elapsed());
        result = Some(value);
    }
    (best, result.expect("the loop runs five times"))
}
```

**What happens:**

1. `work` is a closure that does the job and returns its result (Lesson 05.4). `FnMut` allows it to change captured variables.
2. Each run is timed; `best` keeps the smallest `Duration`.
3. `black_box(work())` makes sure the result counts as used, so the compiler cannot skip the work.
4. The result of the last run is returned, so the caller can print it and prove the work was done.

> 🧪 **Testing:** Even a timing helper deserves a test: does it really run the work five times, and does it ignore one slow run? Exercise 1 asks you to write it.

## 2. Rule 1 — walk memory in the order it is stored

A grid of 4096 × 4096 numbers is usually stored **row-major**: one flat `Vec<f64>`, row after row. Walking it row by row reads memory in order. Walking it column by column jumps 4096 × 8 = 32 KiB forward on every step, so every access lands in a different cache line.

```text
   grid (4 x 4) stored row-major in one Vec:

   index:   0  1  2  3 | 4  5  6  7 | 8  9 10 11 | 12 13 14 15
            row 0      | row 1      | row 2      | row 3

   by rows:     0 → 1 → 2 → 3 → 4 → …      (next element is the neighbour)
   by columns:  0 → 4 → 8 → 12 → 1 → …    (next element is one whole row away)
```

Create `src/bin/grid.rs`:

```rust
use p12_caches::best_of_5;
use std::hint::black_box;

const N: usize = 4096; // a 4096 x 4096 grid of f64 = 128 MiB

fn sum_by_rows(grid: &[f64]) -> f64 {
    let mut total = 0.0;
    for row in 0..N {
        for col in 0..N {
            total += grid[row * N + col];
        }
    }
    total
}

fn sum_by_columns(grid: &[f64]) -> f64 {
    let mut total = 0.0;
    for col in 0..N {
        for row in 0..N {
            total += grid[row * N + col];
        }
    }
    total
}

fn main() {
    let grid = vec![1.0; N * N];
    let (t, sum) = best_of_5(|| sum_by_rows(black_box(&grid)));
    println!("by rows:    {t:>9.2?}  (sum {sum})");
    let (t, sum) = best_of_5(|| sum_by_columns(black_box(&grid)));
    println!("by columns: {t:>9.2?}  (sum {sum})");
}
```

```text
by rows:       8.96ms  (sum 16777216)
by columns:   48.12ms  (sum 16777216)
```

**What happens:**

1. Both functions add the same 16,777,216 numbers; only the order of the two loops differs.
2. By rows, one 128-byte line gives 16 numbers, and the prefetcher sees a simple forward pattern.
3. By columns, each line gives one number before the loop moves away. By the time the loop comes back for the next column, 4096 lines later, the line has long been evicted.
4. The row loop is not even limited by memory here: it waits for its own additions, because every `total += …` needs the previous total. Exercise 2 shows a version that is 35 times faster than the column order.

The size of the grid decides how much this matters. The same program with other values of `N`:

| `N` | Grid size | Fits in | By rows | By columns |
|---|---|---|---|---|
| 64 | 32 KiB | L1 | 3.71 µs | 3.50 µs |
| 256 | 512 KiB | L2 | 35.75 µs | 38.67 µs |
| 1024 | 8 MiB | L2 | 618.88 µs | 1.18 ms |
| 4096 | 128 MiB | RAM | 8.96 ms | 48.12 ms |

While the whole grid fits in L1 or comfortably in L2, the order hardly matters. When it does not fit, the column order pays for a miss on nearly every element.

> ⚠️ **Common mistake:** A `Vec<Vec<f64>>` is **not** one flat grid. Each inner `Vec` is a separate heap allocation, and the outer `Vec` holds pointers to them. Walking one row is still in order, but moving to the next row follows a pointer to somewhere else. For grids, prefer one flat `Vec` and the index `row * width + col`.

## 3. Rule 2 — avoid pointer jumping

Each `Box`, `Rc` or linked-list node is a separate allocation, reached through a pointer. Walking such data means **indirection**: load the pointer, then load the data it points to. If the targets are scattered, every step is a miss.

Create `src/bin/list.rs`. It sums ten million numbers stored in four ways:

```rust
use p12_caches::best_of_5;
use std::collections::LinkedList;
use std::hint::black_box;

fn main() {
    let n: u64 = 10_000_000;
    let vector: Vec<u64> = (0..n).collect();
    let mut boxed: Vec<Box<u64>> = (0..n).map(Box::new).collect();
    let list: LinkedList<u64> = (0..n).collect();

    let (t, sum) = best_of_5(|| black_box(&vector).iter().sum::<u64>());
    println!("Vec<u64>:               {t:>9.2?}  (sum {sum})");
    let (t, sum) = best_of_5(|| black_box(&boxed).iter().map(|b| **b).sum::<u64>());
    println!("Vec<Box<u64>>:          {t:>9.2?}  (sum {sum})");
    let (t, sum) = best_of_5(|| black_box(&list).iter().sum::<u64>());
    println!("LinkedList<u64>:        {t:>9.2?}  (sum {sum})");

    // Mix the boxes: same values, but now the pointers jump around the heap.
    let mut seed: u64 = 0x2545_F491_4F6C_DD1D;
    for i in (1..boxed.len()).rev() {
        seed ^= seed << 13;
        seed ^= seed >> 7;
        seed ^= seed << 17;
        boxed.swap(i, (seed % (i as u64 + 1)) as usize);
    }
    let (t, sum) = best_of_5(|| black_box(&boxed).iter().map(|b| **b).sum::<u64>());
    println!("Vec<Box<u64>> shuffled: {t:>9.2?}  (sum {sum})");
}
```

```text
Vec<u64>:                878.25µs  (sum 49999995000000)
Vec<Box<u64>>:             2.20ms  (sum 49999995000000)
LinkedList<u64>:           8.16ms  (sum 49999995000000)
Vec<Box<u64>> shuffled:   30.90ms  (sum 49999995000000)
```

**What happens:**

1. `Vec<u64>` stores the numbers next to each other: 80 MB read in order, and the compiler can use SIMD instructions that add several numbers at once.
2. `Vec<Box<u64>>` stores ten million pointers; each one points to an 8-byte allocation. The allocator handed out the boxes one after another, so they still lie almost in order, but every number costs an extra load and the data takes more space (each box also has allocator overhead).
3. `LinkedList<u64>` is a chain of nodes, each with the value and two pointers. Here each step must wait for the previous node to tell it where the next one is: pointer chasing (Lesson 12.9). Its nodes also happen to be in order, which is the best case for a list.
4. After the shuffle, the pointers in `boxed` point all over the heap. The work is the same as in point 2, but it takes **35 times longer** than the plain `Vec`. Real programs that allocate and free for a long time get close to this case, for lists too.

> ⚡ **Performance:** Choose `Vec<T>` by default. A linked list wins in theory for inserting in the middle, but finding the middle means walking the chain, and on modern hardware a `Vec` that shifts elements with one fast memory copy usually wins anyway. Reach for `Box` per element only when you need it (recursive types, trait objects).

## 4. Rule 3 — keep together the data you use together

A cache line brings in all the bytes around the value you asked for. If your loop needs only one field of a big struct, most of every line is wasted.

Consider particles with eight numbers of 8 bytes each (a position and a velocity of three numbers each, a mass and a charge). You can store them as an **array of structs** (AoS), `Vec<Particle>`, or as a **struct of arrays** (SoA), one `Vec` per field. Create `src/bin/aos_soa.rs`:

```rust
use p12_caches::best_of_5;
use std::hint::black_box;

/// One particle, stored as a whole: 8 numbers of 8 bytes = 64 bytes.
#[allow(dead_code)] // this example reads only `mass`
#[derive(Clone)]
struct Particle {
    position: [f64; 3],
    velocity: [f64; 3],
    mass: f64,
    charge: f64,
}

/// The same data, one Vec per field.
#[allow(dead_code)]
struct Particles {
    position: Vec<[f64; 3]>,
    velocity: Vec<[f64; 3]>,
    mass: Vec<f64>,
    charge: Vec<f64>,
}

fn heavy_aos(particles: &[Particle]) -> usize {
    particles.iter().filter(|p| p.mass > 1.0).count()
}

fn heavy_soa(particles: &Particles) -> usize {
    particles.mass.iter().filter(|&&m| m > 1.0).count()
}

fn main() {
    let n = 4_000_000;
    let one = Particle {
        position: [0.0; 3],
        velocity: [0.0; 3],
        mass: 2.0,
        charge: 0.0,
    };
    let aos = vec![one; n];
    let soa = Particles {
        position: vec![[0.0; 3]; n],
        velocity: vec![[0.0; 3]; n],
        mass: vec![2.0; n],
        charge: vec![0.0; n],
    };

    let (t, heavy) = best_of_5(|| heavy_aos(black_box(&aos)));
    println!("array of structs: {t:>8.2?}  ({heavy} heavy particles)");
    let (t, heavy) = best_of_5(|| heavy_soa(black_box(&soa)));
    println!("struct of arrays: {t:>8.2?}  ({heavy} heavy particles)");
}
```

```text
array of structs:   2.57ms  (4000000 heavy particles)
struct of arrays: 345.62µs  (4000000 heavy particles)
```

**What happens:**

1. Both functions count particles whose mass is above 1.0. They give the same answer.
2. In the AoS version, every 128-byte line holds two particles, so the loop reads 256 MB from memory to use 32 MB of masses. Seven eighths of the traffic is wasted.
3. In the SoA version, `mass` is one contiguous `Vec<f64>` of 32 MB: every byte that arrives is used, and the compiler can compare several masses with one SIMD instruction. The result is about **8 times faster**.
4. `#[allow(dead_code)]` switches off the compiler warning "field is never read": this small example reads only `mass`.

SoA is not always better. If your loop uses *all* fields of a particle (for example, to move it: add `velocity` to `position`), AoS brings in everything you need with one line, while SoA must stream from several different `Vec`s. Choose the layout by the loops that matter most.

A lighter version of the same idea is **hot/cold splitting**: keep the few fields a frequent loop reads (hot) in one struct, and move big, rarely used fields (cold), such as a long description, into another struct or behind a `Box`. Exercise 3 measures it.

## 5. Rule 4 — do not let threads write to the same line

Each core has its own L1 and L2 caches. When one core writes to a cache line, the hardware must make sure that no other core keeps an old copy. This is **cache coherence**: before the write, the line is taken away from all other caches; a core that wants it back must fetch the new version.

That is fine when threads really share data. It is a disaster when two threads write **different** variables that happen to sit in the **same** line. The line bounces between the two cores on every write, even though the threads never touch each other's data. This is **false sharing**.

Create `src/bin/false_sharing.rs`:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread;
use std::time::Instant;

const ADDS: u64 = 50_000_000;

/// Two counters next to each other: they share one cache line.
struct Close {
    a: AtomicU64,
    b: AtomicU64,
}

/// A counter that fills a whole 128-byte cache line by itself.
#[repr(align(128))]
struct Padded(AtomicU64);

/// Two padded counters: each one has its own cache line.
struct Apart {
    a: Padded,
    b: Padded,
}

fn count(counter: &AtomicU64) {
    for _ in 0..ADDS {
        counter.fetch_add(1, Ordering::Relaxed);
    }
}

fn main() {
    let close = Close {
        a: AtomicU64::new(0),
        b: AtomicU64::new(0),
    };
    let start = Instant::now();
    thread::scope(|s| {
        s.spawn(|| count(&close.a));
        s.spawn(|| count(&close.b));
    });
    println!(
        "same line:      {:>9.2?}  (a = {}, b = {})",
        start.elapsed(),
        close.a.load(Ordering::Relaxed),
        close.b.load(Ordering::Relaxed)
    );

    let apart = Apart {
        a: Padded(AtomicU64::new(0)),
        b: Padded(AtomicU64::new(0)),
    };
    let start = Instant::now();
    thread::scope(|s| {
        s.spawn(|| count(&apart.a.0));
        s.spawn(|| count(&apart.b.0));
    });
    println!(
        "separate lines: {:>9.2?}  (a = {}, b = {})",
        start.elapsed(),
        apart.a.0.load(Ordering::Relaxed),
        apart.b.0.load(Ordering::Relaxed)
    );

    println!(
        "size of Close: {} bytes, size of Apart: {} bytes",
        size_of::<Close>(),
        size_of::<Apart>()
    );
}
```

```text
same line:       425.32ms  (a = 50000000, b = 50000000)
separate lines:   86.73ms  (a = 50000000, b = 50000000)
size of Close: 16 bytes, size of Apart: 256 bytes
```

**What happens:**

1. Each thread adds 1 to its **own** counter fifty million times. No data is shared, and both runs do exactly the same work.
2. In `Close`, the two 8-byte counters sit next to each other inside one 16-byte struct, so they are in one cache line. Every `fetch_add` of one thread steals the line from the other core.
3. `#[repr(align(128))]` tells the compiler to place every `Padded` value at an address that is a multiple of 128 and to round its size up to 128 bytes. (Lesson 12.11 explains alignment and `repr` in detail.) Now each counter owns a whole line, and `Apart` is 256 bytes.
4. With separate lines, each core keeps its line in its own L1 and the same work runs **5 times faster**.

> 💡 **Tip:** You do not need to write `Padded` yourself in real projects: the crate `crossbeam-utils` (version 0.8) has `CachePadded<T>`, which uses the right size for each kind of CPU. Pad only data that different threads **write** often, such as per-thread counters or the two ends of a queue. Padding everything only wastes memory and cache.

## 6. When to care

Cache-friendly code matters only when the working set is bigger than the caches and the loop is hot. So, before you rearrange data: **measure first** (Lesson 05.12; most code is not limited by memory), **check the size** (if everything fits in L2, order hardly matters), **find the access pattern** (do I walk data in stored order, follow pointers, read fields I do not need, let threads write to neighbouring variables?), then **change one thing and measure again**.

> 📏 **Convention:** Start with the simple, clear layout: a `Vec` of plain structs. Switch to SoA, hot/cold splitting or padding only when a measurement shows a hot loop that suffers, and write a comment that explains why the unusual layout exists.

## Common mistakes

**1. Measuring a working set that fits in the cache.** With `N = 64` the grid is 32 KiB and both loop orders take about 3.5 µs. A benchmark with small inputs can say "order does not matter" while production data says the opposite. Test with realistic sizes.

**2. Pointers by habit.** A `Vec<Vec<T>>` grid, a `LinkedList`, or a `Vec<Box<T>>` where `Vec<T>` would do, each add a pointer jump. In section 3, scattered boxes were 35 times slower than a plain `Vec`.

**3. Padding with the wrong size.** `#[repr(align(64))]` is enough on most x86-64 chips, but on Apple chips with 128-byte lines two such values can still share a line. Use 128, or `CachePadded` from `crossbeam-utils`.

## Try it yourself

1. Create `src/lib.rs` with `best_of_5` and `src/bin/grid.rs`. Run `cargo run --release --bin grid`.
2. Change `const N` to 64, 256 and 1024 and run again each time. Fill in your own version of the table from section 2. Where does the column order start to hurt on your Mac?
3. Create `false_sharing.rs` and run it. Change `#[repr(align(128))]` to `#[repr(align(8))]`, run again, and explain the result. Change it back.

## Exercises

### Exercise 1 — Test the timing helper (easy)

Write two tests for `best_of_5` in `src/lib.rs`: one checks that the closure runs exactly five times and that the result of the last run is returned; the other checks that one slow run (sleep 50 ms in the first call only) does not count.

<details>
<summary>Hint</summary>

A closure can change a captured `let mut calls = 0;` because `best_of_5` takes `FnMut`. Use `std::thread::sleep(Duration::from_millis(50))` for the slow run. Run `cargo test -q --lib`.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn runs_the_work_five_times_and_returns_its_result() {
        let mut calls = 0;
        let (_, result) = best_of_5(|| {
            calls += 1;
            calls * 10
        });
        assert_eq!(calls, 5);
        assert_eq!(result, 50, "the result of the last run is returned");
    }

    #[test]
    fn reports_the_fastest_run() {
        let mut calls = 0;
        let (time, ()) = best_of_5(|| {
            calls += 1;
            if calls == 1 {
                std::thread::sleep(Duration::from_millis(50));
            }
        });
        assert!(
            time < Duration::from_millis(50),
            "the slow first run must not count: {time:?}"
        );
    }
}
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.05s
```

Why it works: the closure counts its own calls, and the fifth call returns 50. In the second test four of the five runs take almost no time, so the fastest one is far below 50 ms. Both tests are deterministic: they do not depend on how fast the computer is, only on "50 ms is longer than an empty closure".

</details>

### Exercise 2 — Column sums, the fast way (medium)

Write `fn column_sums_slow(grid: &[f64], n: usize) -> Vec<f64>` that returns the sum of each column of an `n × n` row-major grid by walking down each column. Then write `column_sums_fast` with the same result that walks the grid row by row. Measure both with `n = 4096`, and add a test that both return `[12.0, 15.0, 18.0]` for the 3 × 3 grid `1..=9`.

<details>
<summary>Hint</summary>

In the fast version, keep the `sums` vector and, for each row, add the row's values to `sums` element by element. `grid.chunks_exact(n)` gives you the rows one by one, and `sums.iter_mut().zip(row)` pairs each sum with its value.

</details>

<details>
<summary>Answer</summary>

`src/bin/col_sums.rs`:

```rust
use p12_caches::best_of_5;
use std::hint::black_box;

/// sums[c] = the sum of column c of an n x n grid. Walks down each column.
fn column_sums_slow(grid: &[f64], n: usize) -> Vec<f64> {
    let mut sums = vec![0.0; n];
    for (col, sum) in sums.iter_mut().enumerate() {
        for row in 0..n {
            *sum += grid[row * n + col];
        }
    }
    sums
}

/// The same result, but walks the grid row by row.
fn column_sums_fast(grid: &[f64], n: usize) -> Vec<f64> {
    let mut sums = vec![0.0; n];
    for row in grid.chunks_exact(n) {
        for (sum, value) in sums.iter_mut().zip(row) {
            *sum += value;
        }
    }
    sums
}

fn main() {
    let n = 4096;
    let grid: Vec<f64> = (0..n * n).map(|i| (i % 7) as f64).collect();
    let (t, slow) = best_of_5(|| column_sums_slow(black_box(&grid), n));
    println!("down the columns: {t:>9.2?}");
    let (t, fast) = best_of_5(|| column_sums_fast(black_box(&grid), n));
    println!("row by row:       {t:>9.2?}");
    println!("same sums: {}", slow == fast);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn both_versions_sum_the_columns() {
        let grid: Vec<f64> = (1..=9).map(f64::from).collect();
        assert_eq!(column_sums_slow(&grid, 3), [12.0, 15.0, 18.0]);
        assert_eq!(column_sums_fast(&grid, 3), [12.0, 15.0, 18.0]);
    }
}
```

```text
down the columns:   73.42ms
row by row:          2.12ms
same sums: true
```

Why it works: the fast version reads the grid strictly in order. Its inner loop also adds into `n` *independent* sums, so it does not wait for one long chain of additions, and the compiler can use SIMD. The result is exactly the same, because each column is still added from the top row to the bottom row. Here it is 35 times faster.

</details>

### Exercise 3 — Hot and cold fields (medium)

An `Order` has an `id: u64`, a `paid: bool` and a big `note: [u8; 112]`. Print `size_of::<Order>()`. Count the paid orders among two million orders in two ways: over `&[Order]`, and over a separate `Vec<bool>` that holds only the `paid` flags. Measure both and explain the difference.

<details>
<summary>Hint</summary>

Build `let paid: Vec<bool> = orders.iter().map(|o| o.paid).collect();` once, outside the timed part. Both counts are `.iter().filter(…).count()`.

</details>

<details>
<summary>Answer</summary>

`src/bin/hot_cold.rs`:

```rust
use p12_caches::best_of_5;
use std::hint::black_box;

/// An order with a rarely used, big field: 128 bytes in total.
#[derive(Clone)]
struct Order {
    id: u64,
    paid: bool,
    note: [u8; 112],
}

fn count_paid(orders: &[Order]) -> usize {
    orders.iter().filter(|o| o.paid).count()
}

fn count_paid_hot(paid: &[bool]) -> usize {
    paid.iter().filter(|&&p| p).count()
}

fn main() {
    let n = 2_000_000;
    let orders = vec![
        Order {
            id: 7,
            paid: true,
            note: [0; 112]
        };
        n
    ];
    let paid: Vec<bool> = orders.iter().map(|o| o.paid).collect();
    println!("size of Order: {} bytes", size_of::<Order>());

    let (t, count) = best_of_5(|| count_paid(black_box(&orders)));
    println!("whole orders: {t:>9.2?}  ({count} paid)");
    let (t, count) = best_of_5(|| count_paid_hot(black_box(&paid)));
    println!("hot field:    {t:>9.2?}  ({count} paid)");
    black_box((orders[0].id, orders[0].note[0]));
}
```

```text
size of Order: 128 bytes
whole orders:    4.96ms  (2000000 paid)
hot field:     108.17µs  (2000000 paid)
```

Why it works: each `Order` fills a whole 128-byte cache line, so counting over orders loads 256 MB to read 2 MB of flags. The separate `Vec<bool>` is 2 MB, fits in L2, and is read in order with SIMD. Same answer, 45 times faster. In a real program you would keep `paid` in a small hot struct and move `note` to a cold one.

</details>

### Exercise 4 — False sharing with four threads (hard)

Make an array of four `AtomicU64` counters and start four threads with `thread::scope`, each adding 1 to its own counter twenty million times. Then do the same with an array of four `Padded` counters. Print both times and the total of the padded counters.

<details>
<summary>Hint</summary>

`std::array::from_fn(|_| AtomicU64::new(0))` creates the array. Inside the scope, `for counter in &close { s.spawn(move || count(counter)); }` starts one thread per counter; `move` copies the reference into each thread.

</details>

<details>
<summary>Answer</summary>

Create `src/bin/four_threads.rs`. Copy `Padded` and `count` from `false_sharing.rs` (shown as `// …`); only `ADDS` changes:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread;
use std::time::Instant;

const ADDS: u64 = 20_000_000;

// … `Padded` and `count` exactly as in false_sharing.rs …

fn main() {
    let close: [AtomicU64; 4] = std::array::from_fn(|_| AtomicU64::new(0));
    let start = Instant::now();
    thread::scope(|s| {
        for counter in &close {
            s.spawn(move || count(counter));
        }
    });
    println!("4 counters in one line: {:>9.2?}", start.elapsed());

    let apart: [Padded; 4] = std::array::from_fn(|_| Padded(AtomicU64::new(0)));
    let start = Instant::now();
    thread::scope(|s| {
        for counter in &apart {
            s.spawn(move || count(&counter.0));
        }
    });
    println!("4 padded counters:      {:>9.2?}", start.elapsed());

    let total: u64 = apart.iter().map(|c| c.0.load(Ordering::Relaxed)).sum();
    println!("total of padded counters: {total}");
}
```

```text
4 counters in one line:  897.44ms
4 padded counters:        36.75ms
total of padded counters: 80000000
```

Why it works: the four plain counters take 32 bytes, all inside one cache line, so four cores fight over it: every write takes the line away from the other three. With padding, each core keeps its own line and the threads really run in parallel. With more threads, false sharing gets worse: here it is 24 times slower.

</details>

## Quiz

1. A grid is stored row-major in one `Vec`. Which loop should be the inner one, the row loop or the column loop?

<details>
<summary>Answer</summary>

The column loop. Then consecutive steps visit neighbouring elements of one row, which are next to each other in memory.

</details>

2. Why can summing a `Vec<Box<u64>>` be much slower than summing a `Vec<u64>` with the same numbers?

<details>
<summary>Answer</summary>

Every element is reached through a pointer (indirection): one load for the pointer and one for the value. The values are separate allocations, which take more space and, after the program has allocated and freed for a while, lie scattered over the heap, so many loads miss the cache.

</details>

3. Your hot loop reads only `mass` from a `Vec<Particle>` with 64-byte particles. Which layout change helps, and why?

<details>
<summary>Answer</summary>

Store the masses in their own `Vec<f64>` (struct of arrays, or a hot/cold split). Then every cache line that arrives contains only masses, instead of one mass per 64 bytes.

</details>

4. What is false sharing, and how do you fix it?

<details>
<summary>Answer</summary>

Two threads write different variables that lie in the same cache line, so cache coherence moves the line between the cores on every write. Fix it by placing each variable in its own line, for example with `#[repr(align(128))]` or `CachePadded` from `crossbeam-utils`.

</details>

5. A benchmark with a 100 × 100 grid shows no difference between row and column order. Can you conclude that the order does not matter for your 10,000 × 10,000 production grid?

<details>
<summary>Answer</summary>

No. The small grid (80,000 bytes) fits in the cache, so every order is fast. The big grid (800 MB) does not, and the column order will cause a miss on almost every element. Benchmark with realistic sizes.

</details>

## Summary

- Rule 1: walk memory in the order it is stored. For a row-major grid, the column index belongs in the inner loop; the wrong order was 5 to 35 times slower on a 128 MiB grid.
- Rule 2: avoid pointer jumping. `Vec<T>` beat `Vec<Box<T>>` and `LinkedList<T>`; scattered boxes were 35 times slower.
- Rule 3: keep together the data you use together. Struct of arrays or hot/cold splitting makes every loaded byte useful (8 to 45 times faster in our tests).
- Rule 4: threads that write often should not share a cache line. Padding to 128 bytes removed false sharing and made two threads 5 times faster.
- These effects appear only when the working set is bigger than the caches. Measure with realistic sizes before and after every change.
- Start with simple layouts; switch to special ones only for measured hot loops, and explain why in a comment.

## Sources and further reading

- [The Rust Performance Book](https://nnethercote.github.io/perf-book/) — practical advice on making Rust code fast, including data layout.
- [std::collections](https://doc.rust-lang.org/std/collections/index.html) — the standard library's own advice on choosing a collection ("use `Vec` or `VecDeque`") and the cost of each operation.
- [std::collections::LinkedList](https://doc.rust-lang.org/std/collections/struct.LinkedList.html) — its documentation also says that `Vec` or `VecDeque` is almost always better.
- [crossbeam-utils — CachePadded](https://docs.rs/crossbeam-utils/latest/crossbeam_utils/struct.CachePadded.html) — a ready-made padding type, with a list of line sizes per CPU family.
- [std::sync::atomic::AtomicU64](https://doc.rust-lang.org/std/sync/atomic/type.AtomicU64.html) and [std::thread::scope](https://doc.rust-lang.org/std/thread/fn.scope.html) — the counters and threads used in the false-sharing example.
- [The Rust Reference — Type layout](https://doc.rust-lang.org/reference/type-layout.html) — what `#[repr(align(N))]` guarantees; the next lesson covers it.
- [Ulrich Drepper — What Every Programmer Should Know About Memory](https://www.akkadia.org/drepper/cpumemory.pdf) — section 6 is about writing code for the cache.

---

[🏠 Start](../START-HERE.md) · [← 12.9 The memory hierarchy and caches](09-memory-hierarchy-and-caches.md) · [📋 Phase 12](README.md) · [12.11 Memory layout of Rust types →](11-memory-layout-of-rust-types.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
