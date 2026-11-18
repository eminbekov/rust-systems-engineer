<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 19.5 Memory profiling and fewer allocations](05-memory-profiling-and-allocations.md) · [📋 Phase 19](README.md) · [19.7 SIMD: one instruction, many data →](07-simd.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/19-performance-and-profiling/06-data-layout-for-speed.md) <!-- nav -->

# Lesson 19.6 — Data layout for speed

> **Phase 19 — Performance and profiling** · Lesson 6 of 11 · ⏱ about 135 minutes

## What you will learn

- Why the *shape* of your data often matters more than the code that walks over it.
- How one rare, big enum variant makes every value big, and how a `Box` fixes it.
- How to store a graph in two flat arrays (CSR) instead of a `Vec` per vertex, and when that helps.
- How to choose between `HashMap`, a faster hasher and a plain `Vec` for counting by key.
- How to protect a layout decision with a compile-time check and a test.

## Before you start

- You finished [Lesson 19.5 — Memory profiling and fewer allocations](05-memory-profiling-and-allocations.md).
- You know caches and cache lines ([Lesson 12.9 — The memory hierarchy and caches](../12-computer-systems/09-memory-hierarchy-and-caches.md)), the four rules of [Lesson 12.10 — Writing cache-friendly code](../12-computer-systems/10-cache-friendly-code.md), and sizes, padding and niches from [Lesson 12.11 — Memory layout of Rust types](../12-computer-systems/11-memory-layout-of-rust-types.md).
- You know adjacency lists and breadth-first search ([Lesson 07.13 — Graphs: BFS and DFS](../07-algorithms-and-data-structures/13-graphs-bfs-and-dfs.md)) and hash maps ([Lesson 07.6 — How hash tables work](../07-algorithms-and-data-structures/06-hash-tables.md)).

## New words

| Word | Meaning |
|---|---|
| **data layout** | How values are arranged in memory: their sizes, their order, and whether they sit next to each other or behind pointers. |
| **hot data** | The data that the busy loop of a program reads again and again. Its layout decides the speed. |
| **memory bandwidth** | How many bytes per second can move between RAM (Random Access Memory, the main memory) and the CPU (Central Processing Unit). A loop that reads more bytes than it needs wastes bandwidth. |
| **CSR** (Compressed Sparse Row) | A layout for graphs and sparse matrices: all neighbour lists stored one after another in one array, plus an array of start positions. |
| **offset** | A position inside an array, counted from its start. In CSR, `offsets[v]` says where the neighbours of `v` begin. |
| **dense keys** | Keys that are small integers without big gaps, such as `0..10_000`. They can be used directly as indexes into a `Vec`. |
| **hasher** | The algorithm that turns a key into a hash number for a `HashMap`. The default one, SipHash, is safe against attacks but not the fastest. |
| **FxHash** | A very simple, very fast hash function from the Rust compiler, available as the `rustc-hash` crate. Not safe against deliberate attacks. |
| **hash flooding** | An attack that sends many keys with the same hash, so that a hash table becomes very slow (Lesson 04.9). |

## 1. The shape of the data

Lessons 12.9 to 12.11 taught the hardware facts: the CPU loads whole cache lines of 128 bytes on your Mac, walking memory in order is fast, jumping through pointers is slow, and every type has a size and padding. Those lessons ended with four rules. This lesson adds three layout changes that profiles of real programs often point to:

```text
1. Keep the hot data small           a rare big variant makes every value big
2. Flat arrays instead of many Vecs   one allocation and one scan instead of millions
3. Choose the map for your keys      dense keys -> Vec; trusted keys -> faster hasher
```

Each one is measured below. The package for this lesson:

```bash
cd ~/projects/rust/code
cargo new --lib p19-layout
cd p19-layout
mkdir src/bin
cargo add rustc-hash@2.1
```

`src/lib.rs` holds the timing helper from Lesson 12.10 and a small random number generator, so that every example uses the same data on every run:

```rust
//! Helpers shared by the examples of Lesson 19.6.
use std::hint::black_box;
use std::time::{Duration, Instant};

/// Runs `work` five times and returns the fastest time and the result.
/// (The same helper as in Lesson 12.10.)
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

/// A small, repeatable pseudo-random number generator (xorshift).
pub struct Rng(pub u64);

impl Rng {
    pub fn next_u64(&mut self) -> u64 {
        self.0 ^= self.0 << 13;
        self.0 ^= self.0 >> 7;
        self.0 ^= self.0 << 17;
        self.0
    }

    /// A number in `0..n`.
    pub fn below(&mut self, n: u64) -> u64 {
        self.next_u64() % n
    }
}
```

## 2. Keep the hot data small

An enum is as big as its biggest variant plus the tag (Lesson 12.11). That is fine when all variants are similar. It hurts when one variant is rare and huge: then *every* value pays for it. Imagine a program that processes user interface events. Almost all are clicks; once in a while a user pastes a block of text, kept inline in a 256-byte buffer. `src/bin/events.rs`:

```rust
use p19_layout::{Rng, best_of_5};
use std::hint::black_box;

/// One rare variant with a big inline buffer makes every value big.
#[allow(dead_code)] // this example reads only the clicks
enum Event {
    Click { x: u32, y: u32 },
    Key(u32),
    Paste([u8; 256]),
}

/// The same events, but the rare big payload lives on the heap.
#[allow(dead_code)]
enum SmallEvent {
    Click { x: u32, y: u32 },
    Key(u32),
    Paste(Box<[u8; 256]>),
}

fn sum_clicks(events: &[Event]) -> u64 {
    let mut sum = 0;
    for e in events {
        if let Event::Click { x, y } = e {
            sum += u64::from(*x + *y);
        }
    }
    sum
}

fn sum_clicks_small(events: &[SmallEvent]) -> u64 {
    let mut sum = 0;
    for e in events {
        if let SmallEvent::Click { x, y } = e {
            sum += u64::from(*x + *y);
        }
    }
    sum
}

fn main() {
    println!("size_of::<Event>()      = {} bytes", size_of::<Event>());
    println!(
        "size_of::<SmallEvent>() = {} bytes",
        size_of::<SmallEvent>()
    );

    let n = 1_000_000;
    let mut rng = Rng(42);
    let mut big = Vec::with_capacity(n);
    let mut small = Vec::with_capacity(n);
    for _ in 0..n {
        match rng.below(100) {
            0 => {
                big.push(Event::Paste([7; 256]));
                small.push(SmallEvent::Paste(Box::new([7; 256])));
            }
            1 => {
                big.push(Event::Key(rng.below(128) as u32));
                small.push(SmallEvent::Key(1));
            }
            _ => {
                let (x, y) = (rng.below(1920) as u32, rng.below(1080) as u32);
                big.push(Event::Click { x, y });
                small.push(SmallEvent::Click { x, y });
            }
        }
    }
    let (t, s) = best_of_5(|| sum_clicks(black_box(&big)));
    println!("Vec<Event>:      {t:>9.2?}  (sum {s})");
    let (t, s) = best_of_5(|| sum_clicks_small(black_box(&small)));
    println!("Vec<SmallEvent>: {t:>9.2?}  (sum {s})");
}
```

**What happens:**

1. `Event` stores the paste buffer inline: 256 bytes, plus the tag, rounded to the alignment of `u32`. `SmallEvent` stores only a `Box`, an 8-byte pointer, and the buffer lives on the heap.
2. The loop makes one million events: 98% clicks, 1% keys, 1% pastes. The same random numbers go into both vectors, so the sums must be equal.
3. `sum_clicks` reads only the tag and the two coordinates of each event. Everything else in the value is never needed in this loop, but it still sits between the useful bytes.

`cargo run --release --bin events`:

```text
size_of::<Event>()      = 260 bytes
size_of::<SmallEvent>() = 16 bytes
Vec<Event>:         4.94ms  (sum 1468695352)
Vec<SmallEvent>:  423.08µs  (sum 1468695352)
```

The same answer, **11 times faster**. The vector of `Event` is 260 MB (megabytes): for each click, the CPU loads a whole 128-byte cache line (the events are 260 bytes apart, so no two share one) and uses only 12 bytes of it. The vector of `SmallEvent` is 16 MB, eight events per cache line. The rare paste now costs one extra pointer jump, but only for 1% of the events, and the loop above never touches it.

`cargo clippy` knows this problem and warns about it:

```text
warning: large size difference between variants
  --> src/bin/events.rs:6:1
   |
 6 | / enum Event {
 7 | |     Click { x: u32, y: u32 },
   | |     ------------------------ the second-largest variant contains at least 8 bytes
 8 | |     Key(u32),
 9 | |     Paste([u8; 256]),
   | |     ---------------- the largest variant contains at least 256 bytes
10 | | }
   | |_^ the entire enum is at least 260 bytes
   |
   = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.100.0/index.html#large_enum_variant
   = note: `#[warn(clippy::large_enum_variant)]` on by default
```

The same idea works for structs and integers. A `usize` ID is 8 bytes; if you will never have more than four billion items, a `u32` ID is 4 bytes and twice as many fit in a cache line. A rarely used `String` field that is usually empty can become `Option<Box<str>>`: 16 bytes instead of 24, and `None` costs nothing extra thanks to the niche from Lesson 12.11.

> ⚡ **Performance:** a smaller type helps only if it is *hot*: read in a busy loop over many values. Boxing a variant adds an allocation when the value is created and a pointer jump when the big variant is used. Measure both sides; for a type that lives in a handful of variables, size does not matter.

## 3. Flat arrays instead of many small vectors

In Lesson 07.13 you stored graphs as **adjacency lists**, `Vec<Vec<usize>>`: one vector of neighbours per vertex. It is simple and correct. For a big graph, it means one allocation per vertex, 24 bytes of `Vec` header per vertex, 8-byte `usize` targets, and neighbour lists scattered over the heap.

**CSR** (Compressed Sparse Row) stores the same graph in two flat arrays:

```text
adjacency lists              CSR
0: [1, 2]                    targets: [1, 2, | 0, 0, 3, | 2]     (vertex 1 has none)
1: []                        offsets: [0, 2, 2, 5, 6]
2: [0, 0, 3]                 neighbours of v = targets[offsets[v] .. offsets[v+1]]
3: [2]                       neighbours of 2 = targets[2..5] = [0, 0, 3]
```

`targets` holds all neighbour lists one after another; `offsets[v]` says where the list of `v` starts, and `offsets[v + 1]` where it ends. With `u32` instead of `usize`, every number is half as big. `src/bin/graph.rs` builds a random graph with two million vertices and eight edges each, in both layouts, and runs two tasks: a full scan of all edges, and a breadth-first search (BFS) from vertex 0:

```rust
use p19_layout::{Rng, best_of_5};
use std::collections::VecDeque;
use std::hint::black_box;

/// The graph as in Lesson 07.13: one Vec of neighbours per vertex.
fn build_lists(n: usize, degree: usize, rng: &mut Rng) -> Vec<Vec<usize>> {
    (0..n)
        .map(|_| (0..degree).map(|_| rng.below(n as u64) as usize).collect())
        .collect()
}

/// The same graph in two flat arrays (CSR, "compressed sparse row"):
/// the neighbours of `v` are `targets[offsets[v]..offsets[v + 1]]`.
struct Csr {
    offsets: Vec<u32>,
    targets: Vec<u32>,
}

impl Csr {
    fn from_lists(lists: &[Vec<usize>]) -> Csr {
        let mut offsets = Vec::with_capacity(lists.len() + 1);
        let mut targets = Vec::new();
        offsets.push(0);
        for list in lists {
            targets.extend(list.iter().map(|&t| t as u32));
            offsets.push(targets.len() as u32);
        }
        Csr { offsets, targets }
    }

    fn neighbours(&self, v: usize) -> &[u32] {
        &self.targets[self.offsets[v] as usize..self.offsets[v + 1] as usize]
    }
}

/// Breadth-first search from vertex 0; returns how many vertices were reached.
fn bfs_lists(lists: &[Vec<usize>]) -> usize {
    let mut seen = vec![false; lists.len()];
    let mut queue = VecDeque::from([0]);
    seen[0] = true;
    let mut reached = 0;
    while let Some(v) = queue.pop_front() {
        reached += 1;
        for &u in &lists[v] {
            if !seen[u] {
                seen[u] = true;
                queue.push_back(u);
            }
        }
    }
    reached
}

fn bfs_csr(g: &Csr) -> usize {
    let mut seen = vec![false; g.offsets.len() - 1];
    let mut queue = VecDeque::from([0u32]);
    seen[0] = true;
    let mut reached = 0;
    while let Some(v) = queue.pop_front() {
        reached += 1;
        for &u in g.neighbours(v as usize) {
            if !seen[u as usize] {
                seen[u as usize] = true;
                queue.push_back(u);
            }
        }
    }
    reached
}

/// Sum of all edge targets: a pure scan, no random access.
fn sum_lists(lists: &[Vec<usize>]) -> u64 {
    lists.iter().flatten().map(|&u| u as u64).sum()
}

fn sum_csr(g: &Csr) -> u64 {
    g.targets.iter().map(|&u| u64::from(u)).sum()
}

fn main() {
    let (n, degree) = (2_000_000, 8);
    let mut rng = Rng(7);
    let lists = build_lists(n, degree, &mut rng);
    let csr = Csr::from_lists(&lists);

    let list_bytes = n * size_of::<Vec<usize>>() + n * degree * size_of::<usize>();
    let csr_bytes = (csr.offsets.len() + csr.targets.len()) * size_of::<u32>();
    println!("Vec<Vec<usize>>: {} MB", list_bytes / 1_000_000);
    println!("CSR with u32:    {} MB", csr_bytes / 1_000_000);

    let (t, r) = best_of_5(|| sum_lists(black_box(&lists)));
    println!("sum Vec<Vec<usize>>: {t:>9.2?}  ({r})");
    let (t, r) = best_of_5(|| sum_csr(black_box(&csr)));
    println!("sum CSR:             {t:>9.2?}  ({r})");
    let (t, r) = best_of_5(|| bfs_lists(black_box(&lists)));
    println!("BFS Vec<Vec<usize>>: {t:>9.2?}  ({r} reached)");
    let (t, r) = best_of_5(|| bfs_csr(black_box(&csr)));
    println!("BFS CSR:             {t:>9.2?}  ({r} reached)");
}
```

**What happens:**

1. `from_lists` copies every list into `targets` and records where each one ends. `offsets` has one more entry than there are vertices, so `offsets[v + 1]` always exists.
2. `neighbours(v)` returns a slice of `targets`: no copy, no allocation.
3. The two BFS functions are the algorithm from Lesson 07.13, once per layout. The memory estimate counts the `Vec` headers (24 bytes each) and the targets; the real `Vec<Vec<usize>>` also has allocator overhead per list.

`cargo run --release --bin graph`:

```text
Vec<Vec<usize>>: 176 MB
CSR with u32:    72 MB
sum Vec<Vec<usize>>:   10.17ms  (15999321722498)
sum CSR:              785.50µs  (15999321722498)
BFS Vec<Vec<usize>>:  110.10ms  (1999337 reached)
BFS CSR:               97.02ms  (1999337 reached)
```

Read the result carefully, because it is not "CSR is always 13 times faster":

- **Memory:** 72 MB instead of 176 MB, and 2 allocations instead of 2 million (building and dropping the lists takes time too).
- **Scanning all edges:** 13 times faster. CSR is one contiguous array of 4-byte numbers that the CPU reads in order (and the compiler can add several numbers at once); the lists need a pointer jump per vertex and twice as many bytes.
- **BFS:** only about 10% faster. BFS jumps to random vertices: `seen[u]` and the neighbour list of each new vertex are somewhere else in memory every time. Those random accesses dominate, and both layouts pay for them.

That is the honest rule: a compact layout helps most where the loop walks memory in order, and gives smaller gains where the algorithm itself jumps around. Graph databases, routing engines and sparse matrix code use CSR because it wins on memory and scans, and never loses.

> 🧪 **Testing:** two layouts of the same data must give the same answers. Exercise 2 asks you to test that `Csr` returns exactly the neighbours of the lists, including a vertex without neighbours and a repeated neighbour.

## 4. Choose the map for your keys

Counting things by key is everywhere: requests per user, words per text, packets per address. The usual tool is a `HashMap`. The standard one uses **SipHash** with a random key (Lesson 07.6), which protects against **hash flooding** but costs time for every lookup. If the keys are small integers without big gaps, you do not need hashing at all. `src/bin/maps.rs` counts ten million random IDs between 0 and 9 999 three ways:

```rust
use p19_layout::{Rng, best_of_5};
use rustc_hash::FxHashMap;
use std::collections::HashMap;
use std::hint::black_box;

/// Counts how often each id appears, with the standard HashMap (SipHash).
fn count_std(ids: &[u32]) -> HashMap<u32, u32> {
    let mut counts = HashMap::new();
    for &id in ids {
        *counts.entry(id).or_insert(0) += 1;
    }
    counts
}

/// The same with FxHashMap: a much simpler and faster hash function.
fn count_fx(ids: &[u32]) -> FxHashMap<u32, u32> {
    let mut counts = FxHashMap::default();
    for &id in ids {
        *counts.entry(id).or_insert(0) += 1;
    }
    counts
}

/// Ids are small and dense (0..max), so a plain Vec indexed by id is enough.
fn count_vec(ids: &[u32], max: u32) -> Vec<u32> {
    let mut counts = vec![0; max as usize];
    for &id in ids {
        counts[id as usize] += 1;
    }
    counts
}

fn main() {
    let max = 10_000;
    let mut rng = Rng(1);
    let ids: Vec<u32> = (0..10_000_000).map(|_| rng.below(max as u64) as u32).collect();

    let (t, c) = best_of_5(|| count_std(black_box(&ids)));
    println!("HashMap (SipHash): {t:>9.2?}  (id 7 seen {} times)", c[&7]);
    let (t, c) = best_of_5(|| count_fx(black_box(&ids)));
    println!("FxHashMap:         {t:>9.2?}  (id 7 seen {} times)", c[&7]);
    let (t, c) = best_of_5(|| count_vec(black_box(&ids), max));
    println!("Vec by index:      {t:>9.2?}  (id 7 seen {} times)", c[7]);
}
```

**What happens:**

1. `FxHashMap<K, V>` is the standard `HashMap` with a different hasher: `HashMap<K, V, FxBuildHasher>`. It is created with `default()` instead of `new()`, because `new()` exists only for the default hasher. Everything else (`entry`, `get`, `insert`) is the same.
2. `count_vec` uses the ID itself as the index. There is no hashing, no comparison and no probing: one memory access per ID, into a 40 KB (kilobyte) array that stays in the cache.
3. `c[&7]` reads a map by key; `c[7]` reads a vector by index. All three print the same count, which shows the three versions agree.

`cargo run --release --bin maps`:

```text
HashMap (SipHash):   48.51ms  (id 7 seen 1049 times)
FxHashMap:           10.96ms  (id 7 seen 1049 times)
Vec by index:         2.82ms  (id 7 seen 1049 times)
```

FxHash is 4.4 times faster than SipHash here, and the `Vec` is 17 times faster. A short guide:

| Keys | Choice |
|---|---|
| small integers in a known range (`0..n`, with few gaps) | `Vec<V>` indexed by key |
| your own data, not chosen by an outsider (IDs from your database, words from your own files) | `FxHashMap` (or another fast hasher) |
| anything that comes from users or the network (headers of HTTP, the HyperText Transfer Protocol; user names; keys in JSON, the JavaScript Object Notation) | the standard `HashMap` |

> 🔒 **Security:** FxHash has no secret key. An attacker who can choose keys can easily produce thousands that land in the same bucket, and every lookup then becomes a slow search: a DoS (Denial of Service) attack called hash flooding (Lesson 04.9). Never use a fast unkeyed hasher for keys from untrusted input. When in doubt, keep the default.

## Common mistakes

1. **Changing the layout without a profile.** A layout change makes code harder to read. Do it for data that a profile shows as hot (Lessons 19.3 and 19.4), and measure before and after.
2. **Boxing everything.** `Box` helps a *rare* big variant. Boxing a variant that is common adds an allocation and a pointer jump to most values and makes the program slower.
3. **`u32` IDs that can overflow.** Converting with `as u32` silently cuts off large values (Lesson 12.2). If more than four billion items are possible, use `u64`, or convert with `u32::try_from(x)` and handle the error.
4. **A fast hasher on network input.** See the security note above. `FxHashMap` belongs to internal, trusted data.
5. **Comparing layouts at a size that fits in the cache.** With 1,000 events, both enum layouts fit in the caches and run at almost the same speed. Layout effects appear when the hot data is bigger than the caches (Lesson 12.10). Test with realistic sizes.

## Try it yourself

1. In `events.rs`, change the event mix to 50% clicks and 49% keys (`1..=49 =>`). The difference between the layouts shrinks. Why? (Hint: Lesson 12.6. The `if let` is now a branch that the CPU cannot predict, and the cost of wrong guesses hides the memory cost.)
2. In `graph.rs`, change `degree` to 2 and then to 32. How does the gap between the two scans change?
3. In `maps.rs`, change `max` to 10 000 000. The `Vec` is now 40 MB and no longer fits in the caches. Which version wins now?

## Exercises

### Exercise 1 — Guard the size at compile time (easy)

Add a check to `events.rs` that makes the build fail if `SmallEvent` ever becomes bigger than 16 bytes. Then, to see it work, change `Paste(Box<[u8; 256]>)` back to `Paste([u8; 256])` in `SmallEvent` (and the line that creates it) and build.

<details>
<summary>Hint</summary>

Lesson 12.11 used `const _: () = assert!(...);`. A `const` is evaluated by the compiler, so a failing `assert!` is a compile error.

</details>

<details>
<summary>Answer</summary>

After the `SmallEvent` definition:

```rust
// Fails to compile if someone makes SmallEvent big again.
const _: () = assert!(size_of::<SmallEvent>() <= 16);
```

With the inline buffer, `cargo build --release --bin events` fails:

```text
error[E0080]: evaluation panicked: assertion failed: size_of::<SmallEvent>() <= 16
  --> src/bin/events.rs:21:15
   |
21 | const _: () = assert!(size_of::<SmallEvent>() <= 16);
   |               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ evaluation of `_` failed here
```

Why it works: the check runs during compilation and costs nothing at run time. It turns a performance decision into something the compiler enforces, so a colleague who adds a big field gets an error with a clear reason instead of a slowdown nobody notices. Put the box back afterwards.

</details>

### Exercise 2 — Test the CSR layout (medium)

Write two tests in `graph.rs`: (a) for the small graph `[[1, 2], [], [0, 0, 3], [2]]`, `Csr::from_lists` gives `offsets` `[0, 2, 2, 5, 6]` and `neighbours(v)` equal to each list; (b) for a random graph with 1000 vertices and degree 2, both BFS functions reach the same number of vertices. Run them with `cargo test --bin graph`.

<details>
<summary>Hint</summary>

The fields of `Csr` are private but the test module is inside the same file, so it can read them. To compare `&[u32]` with a `Vec<usize>`, convert the list first: `list.iter().map(|&u| u as u32).collect::<Vec<u32>>()`.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn csr_has_the_same_neighbours() {
        // Vertex 1 has no neighbours; vertex 2 has a repeated one.
        let lists = vec![vec![1, 2], vec![], vec![0, 0, 3], vec![2]];
        let csr = Csr::from_lists(&lists);
        assert_eq!(csr.offsets, [0, 2, 2, 5, 6]);
        for (v, list) in lists.iter().enumerate() {
            let expected: Vec<u32> = list.iter().map(|&u| u as u32).collect();
            assert_eq!(csr.neighbours(v), expected, "vertex {v}");
        }
    }

    #[test]
    fn both_searches_reach_the_same_vertices() {
        let lists = build_lists(1000, 2, &mut Rng(3));
        assert_eq!(bfs_lists(&lists), bfs_csr(&Csr::from_lists(&lists)));
    }
}
```

```text
test tests::csr_has_the_same_neighbours ... ok
test tests::both_searches_reach_the_same_vertices ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the first test covers the edge cases where offset code usually breaks (an empty list makes two equal offsets in a row; a repeated neighbour must be kept). The second compares the two layouts on bigger data through the algorithm that uses them.

</details>

### Exercise 3 — A faster hasher for the word counter (hard)

In `code/p19-wordfreq` (Lesson 19.5), the `HashMap<String, usize>` was one of the biggest parts of the Linux profile. Replace it with `FxHashMap` everywhere (the program, `count_words`, `top` and the tests), check that the output on `words.txt` is identical, and measure with `hyperfine`. Is this safe for this program?

<details>
<summary>Hint</summary>

`cargo add rustc-hash@2.1`, then `use rustc_hash::FxHashMap;` and replace `HashMap::new()` with `FxHashMap::default()`. Keep a copy of the old binary for the comparison.

</details>

<details>
<summary>Answer</summary>

The changed lines in `src/main.rs`:

```text
use rustc_hash::FxHashMap;

fn count_words(text: &str, counts: &mut FxHashMap<String, usize>) {
fn top(counts: &FxHashMap<String, usize>, n: usize) -> Vec<(String, usize)> {
    let mut counts = FxHashMap::default();
```

(The `count_words` and `top` lines show only the new signatures; the bodies stay the same. Replace every `HashMap::new()` in `main` and in the tests with `FxHashMap::default()`.) `std-wordfreq` is a copy of the binary from before the change:

```text
identical output
Benchmark 1: ./std-wordfreq words.txt (10 runs)
                mean     ±       σ          min     …     max
  Wall Time    714.8 ms  ±     6.5 ms     706.9 ms  …   725.1 ms
  Memory       130.9 MiB ±     0.4 MiB    130.4 MiB …   131.4 MiB
 
Benchmark 2: ./target/release/p19-wordfreq words.txt (10 runs)
                mean     ±       σ          min     …     max
  Wall Time    629.3 ms  ±    13.8 ms     613.1 ms  …   663.5 ms           -12.0%
  Memory       130.8 MiB ±     0.4 MiB    130.4 MiB …   131.4 MiB           -0.1%
```

Why it works: hashing a short word with FxHash takes a few instructions instead of the SipHash rounds that showed up in the profiles. The output stays identical because `top` sorts by count and then by word, so the iteration order of the map does not matter. Is it safe? For a command-line tool that counts words in your own files, yes. If this code ever ran in a service on text sent by strangers, an attacker could send words chosen to collide and make it very slow; there, keep the standard `HashMap`.

</details>

## Quiz

1. An enum has variants of 8, 4 and 256 bytes. How big is a `Vec` of one million such values, roughly, and why?

<details>
<summary>Answer</summary>

About 260 MB: every value takes the size of the largest variant plus the tag (260 bytes here), whichever variant it actually holds.

</details>

2. When does boxing a big variant make things *slower*?

<details>
<summary>Answer</summary>

When the big variant is common or is read in the hot loop: then most values need an extra allocation when created and an extra pointer jump when read.

</details>

3. What two arrays make up a CSR graph, and how do you find the neighbours of vertex `v`?

<details>
<summary>Answer</summary>

`targets` (all neighbour lists one after another) and `offsets` (where each list starts, with one extra entry at the end). The neighbours of `v` are `targets[offsets[v]..offsets[v + 1]]`.

</details>

4. Why did CSR make the full scan 13 times faster but BFS only about 10% faster?

<details>
<summary>Answer</summary>

The scan reads one contiguous array in order, so the compact layout (fewer bytes, no pointer per vertex) pays off fully. BFS jumps to random vertices and random positions in `seen`; those random memory accesses dominate its time and cost the same in both layouts.

</details>

5. When should you *not* use `FxHashMap`?

<details>
<summary>Answer</summary>

When the keys can be chosen by someone outside your control (users, network input). Without a secret key, an attacker can create many colliding keys and slow the map down (hash flooding).

</details>

6. You count events per hour of the day. Which map do you choose?

<details>
<summary>Answer</summary>

A plain array or `Vec` with 24 entries, indexed by the hour. The keys are small and dense; no hashing is needed.

</details>

## Summary

- The layout of the hot data often decides the speed of a loop. Change it where a profile points, and measure.
- An enum is as big as its biggest variant: box a rare big variant (clippy's `large_enum_variant` warns you); use `u32` IDs where the range allows.
- Many small `Vec`s cost an allocation and a pointer each; a CSR graph stores everything in two flat arrays: 2.4 times less memory and 13 times faster scans here, but only about 10% for BFS.
- Dense integer keys belong in a `Vec`; trusted keys can use a fast hasher such as `FxHashMap`; keys from untrusted input keep the standard `HashMap`.
- Protect layout decisions with `const` size assertions and tests that compare the old and new layouts.

## Sources and further reading

- [The Rust Performance Book — Type Sizes](https://nnethercote.github.io/perf-book/type-sizes.html) — measuring and shrinking types, including boxing large variants.
- [The Rust Performance Book — Hashing](https://nnethercote.github.io/perf-book/hashing.html) — alternative hashers and when they are appropriate.
- [Clippy lints](https://rust-lang.github.io/rust-clippy/stable/index.html) — search for `large_enum_variant` to read the lint's explanation.
- [rustc-hash documentation](https://docs.rs/rustc-hash/latest/rustc_hash/) — `FxHashMap`, `FxHashSet` and `FxBuildHasher`.
- [`std::collections::HashMap`](https://doc.rust-lang.org/std/collections/struct.HashMap.html) — the default hasher, `with_hasher`, and the note on hash flooding.
- [Wikipedia — Sparse matrix](https://en.wikipedia.org/wiki/Sparse_matrix) — the compressed sparse row (CSR) format explained with pictures.

---

[🏠 Start](../START-HERE.md) · [← 19.5 Memory profiling and fewer allocations](05-memory-profiling-and-allocations.md) · [📋 Phase 19](README.md) · [19.7 SIMD: one instruction, many data →](07-simd.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
