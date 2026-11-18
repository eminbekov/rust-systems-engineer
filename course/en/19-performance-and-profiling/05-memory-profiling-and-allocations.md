<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 19.4 Profiling on Linux with perf](04-profiling-on-linux-with-perf.md) · [📋 Phase 19](README.md) · [19.6 Data layout for speed →](06-data-layout-for-speed.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/19-performance-and-profiling/05-memory-profiling-and-allocations.md) <!-- nav -->

# Lesson 19.5 — Memory profiling and fewer allocations

> **Phase 19 — Performance and profiling** · Lesson 5 of 11 · ⏱ about 105 minutes

## What you will learn

- Why heap allocations cost time, and the difference between **how many** allocations a program makes and **how much** memory it holds at its **peak**.
- How to measure both with the heap profiler `dhat`, switched on by a Cargo feature of your own.
- How to write a test that fails when code allocates more than it should.
- Practical ways to allocate less: reuse a buffer, look up with `&str`, borrow instead of cloning, and read input in pieces.

## Before you start

- You finished [Lesson 19.4 — Profiling on Linux with perf](04-profiling-on-linux-with-perf.md). We continue with `code/p19-wordfreq`.
- You know how an allocator hands out heap blocks and what `#[global_allocator]` does ([Lesson 12.13 — How memory allocators work](../12-computer-systems/13-how-memory-allocators-work.md)).
- You know crate features from [Lesson 05.8 — Cargo: dependencies and versions](../05-rust-intermediate/08-cargo-dependencies-and-versions.md). Here you define a feature in your own package.

## New words

| Word | Meaning |
|---|---|
| **allocation** | Asking the allocator for a block of heap memory (`malloc` underneath). Every `String`, `Vec` or `Box` with content owns one. |
| **block** | One piece of memory handed out by one allocation. |
| **allocation rate** (turnover) | How many blocks a program allocates (and frees) in total. Many short-lived blocks cost time. |
| **peak memory** | The largest amount of heap memory the program holds at one moment. It decides how much RAM (Random Access Memory) the program needs. |
| **heap profiler** | A tool that records every allocation: where in the code it happened, how big it was, and how long it lived. |
| **dhat** | A heap profiler for Rust, published as a crate. It replaces the global allocator with a counting one. Its name and its viewer come from **DHAT** (Dynamic Heap Analysis Tool), a heap profiler that is part of the Valgrind tool set. |
| **t-gmax** | dhat's name for the moment of the global maximum: the time when the heap was biggest. |
| **t-end** | dhat's name for the end of the program. Blocks still alive at t-end were never freed. |
| **program point** (PP) | In dhat's report, one place in the code (with its call stack) that allocates. |
| **optional dependency** | A dependency that is compiled only when a feature of your package switches it on. |
| **buffer reuse** | Keeping one `String` or `Vec` and clearing it for each new item, instead of creating a new one every time. |

## 1. Two questions about memory

In Lesson 19.3 almost half of the time on macOS went into `malloc`, `free` and growing strings. An allocation is not free: the allocator must find a block of the right size, keep its bookkeeping in order, and later take the block back (Lesson 12.13). One allocation costs tens of nanoseconds; 25 million of them cost a second.

There are two different questions, and they need different fixes:

```text
How MANY blocks? (allocation rate)        How MUCH at once? (peak memory)
  25 million tiny Strings, each freed        one 130 MB String with the whole file
  right away                                 held until the end
  -> costs CPU time in malloc/free           -> costs RAM; can get the process killed
  -> fix: reuse, borrow, avoid copies        -> fix: process data in pieces
```

A sampling profiler shows the first problem as wide `malloc` and `free` boxes, but it cannot tell you *which line* allocated, or how much memory is held at the peak. A **heap profiler** can.

## 2. Setting up dhat

`dhat` works by replacing the global allocator with one that records every allocation. That slows the program down a lot, so we switch it on only when we ask for it, with a feature of our own package. Add the crate as an **optional** dependency:

```bash
cd ~/projects/rust/code/p19-wordfreq
cargo add dhat@0.3 --optional
```

Then make sure `Cargo.toml` has this `[features]` section (Cargo may have added a feature named `dhat`; rename it to `dhat-heap`, the name the `dhat` documentation uses):

```toml
[dependencies]
dhat = { version = "0.3", optional = true }

[features]
dhat-heap = ["dep:dhat"]
```

**What happens:**

1. `optional = true`: the `dhat` crate is not compiled in a normal build.
2. `[features]` defines switches for *your* package. `dhat-heap = ["dep:dhat"]` means: "when the feature `dhat-heap` is on, enable the dependency `dhat`".
3. `cargo build --features dhat-heap` turns it on; plain `cargo build` does not.

Now two small additions to `src/main.rs`. At the top, after the `use` lines:

```rust
// With `--features dhat-heap`, every allocation goes through dhat's counting allocator.
#[cfg(feature = "dhat-heap")]
#[global_allocator]
static ALLOC: dhat::Alloc = dhat::Alloc;
```

And as the first lines of `main`:

```rust
    // Starts heap profiling; the report is written when `_profiler` is dropped at the end of main.
    #[cfg(feature = "dhat-heap")]
    let _profiler = dhat::Profiler::new_heap();
```

**What happens:**

1. `#[cfg(feature = "dhat-heap")]` compiles the next item only when the feature is on. Without the feature, the program is exactly as before.
2. `#[global_allocator]` makes `dhat::Alloc` the allocator for every `Box`, `Vec` and `String` in the program (Lesson 12.13 wrote its own).
3. `Profiler::new_heap()` starts recording. The variable must have a name (`_profiler`, not `_`), so that it lives until the end of `main`; when it is dropped, dhat prints a summary and writes `dhat-heap.json`.

dhat makes the program many times slower, so use a small input. Generate one million words (5 MB, megabytes):

```bash
cargo run --release --bin gen -- small.txt 1000000
echo "small.txt" >> .gitignore
echo "dhat-heap.json" >> .gitignore
cargo run --profile profiling --features dhat-heap --bin p19-wordfreq -- small.txt
```

```text
22576 different words
   52463 ka
   52370 ru
   52280 mi
   52077 ing
   52068 lo
   51825 ne
   51661 pre
   51482 sta
     958 theing
     943 ruis
dhat: Total:     13,246,059 bytes in 1,022,599 blocks
dhat: At t-gmax: 7,001,579 bytes in 14,342 blocks
dhat: At t-end:  1,088 bytes in 2 blocks
dhat: The data has been saved to dhat-heap.json, and is viewable with dhat/dh_view.html
```

The program ran for about 12.6 seconds instead of 0.2 seconds without dhat. **Read the three dhat lines:**

1. **Total:** during the whole run, 1,022,599 blocks were allocated, 13 MB in total. That is about **one allocation per word**: `normalize` still returns a new `String` for every word (after the Lesson 19.3 fix; the original version made 2,127,791 blocks, about two per word).
2. **At t-gmax:** at the peak, the program held 7 MB in 14,342 blocks: the 5.2 MB text of the file, the `HashMap` and the words in it.
3. **At t-end:** 1,088 bytes in 2 blocks were still alive at the very end: small buffers of the standard library (for example the one behind `println!`). A number that grows with the input here would mean a leak.

We use the `profiling` profile from Lesson 19.3, because dhat needs debug info to show file names and line numbers in its report.

## 3. Looking inside: the DHAT viewer

`dhat-heap.json` holds the details. Open the DHAT viewer at `https://nnethercote.github.io/dh_view/dh_view.html` in your browser and load the file with its *Load…* button. The page reads the file inside your browser; it does not upload it.

What you see, in words:

- At the top, the same totals as in the terminal, plus the time of t-gmax and t-end in microseconds.
- A **sort metric** menu. The default, "Total (blocks)", finds the places that allocate most *often*; "Total (bytes)", "At t-gmax (bytes)" and "At t-end (bytes)" sort by size, by memory held at the peak and by memory never freed.
- Below, a tree of **program points** (PPs). Each one shows *Total*, *At t-gmax* and *At t-end* for the blocks allocated there, and an "Allocated at" call stack. You can open a PP to see its children (the different callers).

For our program, sorted by the default metric, the list starts like this (call stacks shortened to our own functions):

```text
PP 1: 1,000,000 blocks, 4,182,316 bytes   <- p19_wordfreq::normalize (src/main.rs:23)  <- count_words
PP 2:    22,575 blocks,   205,947 bytes   <- p19_wordfreq::top::{closure#0} (src/main.rs:54)
PP 3:        14 blocks, 2,162,668 bytes   <- count_words (src/main.rs:49): HashMap growing (reserve_rehash)
PP 4:         1 block,  5,248,982 bytes   <- main (src/main.rs:70): fs::read_to_string
```

**What happens:**

1. PP 1 is the `String::with_capacity` inside `normalize`: one block per word that is not skipped by `split_whitespace`. It is 98% of all blocks.
2. PP 2 is `w.clone()` in `top`: every one of the 22,575 different words is copied once, only to be sorted.
3. PP 3: the `HashMap` grew 14 times. Each time it allocates a bigger table and moves everything; the last table holds 1.6 MB at t-gmax.
4. PP 4: the whole file as one block, the biggest single part of the peak.

So the allocation *rate* is about `normalize`, and the *peak* is about reading the whole file at once. We fix the first one in this lesson's main part and the second one in Exercise 3.

> 🔒 **Security:** a profile file contains paths and function names of your program. The DHAT viewer works locally, but do not attach `dhat-heap.json` of private code to public issues.

## 4. A test that counts allocations

Before changing the code, let us write down the goal as a test: **counting words that are already in the map should not allocate per word.** `dhat` has a testing mode for exactly this. Add to the `tests` module in `src/main.rs`:

```rust
    #[cfg(feature = "dhat-heap")]
    #[test]
    fn known_words_do_not_allocate() {
        let mut counts = HashMap::new();
        count_words("x y z", &mut counts); // the first time, new words need Strings
        let text = "x Y z, ".repeat(1000); // made before profiling starts

        let _profiler = dhat::Profiler::builder().testing().build();
        count_words(&text, &mut counts);
        let stats = dhat::HeapStats::get();

        // Only the reusable `word` buffer, however many words there are.
        dhat::assert_eq!(stats.total_blocks, 1);
        assert_eq!(counts["x"], 1001);
    }
```

**What happens:**

1. `#[cfg(feature = "dhat-heap")]`: the test exists only when the feature is on, because it needs dhat's global allocator.
2. The first `count_words` call puts `x`, `y` and `z` into the map. The test text is built *before* profiling starts, so its allocation is not counted.
3. `Profiler::builder().testing().build()` starts dhat in testing mode: no file is written unless an assertion fails.
4. `HeapStats::get()` returns the numbers since the profiler started: `total_blocks`, `total_bytes`, `curr_blocks`, `max_bytes` and more.
5. `dhat::assert_eq!` works like `assert_eq!`, but on failure it also saves the profile, so you can see *where* the unexpected blocks came from.
6. We expect exactly 1 block: the one buffer the new code will reuse for all 3000 words.

Run only this test (dhat counts allocations of *all* threads, so other tests running at the same time would disturb the numbers):

```bash
cargo test --features dhat-heap known_words
```

```text
---- tests::known_words_do_not_allocate stdout ----
dhat: Total:     4,000 bytes in 3,000 blocks
dhat: At t-gmax: 2 bytes in 1 blocks
dhat: At t-end:  0 bytes in 0 blocks
dhat: The data has been saved to dhat-heap.json, and is viewable with dhat/dh_view.html

thread 'tests::known_words_do_not_allocate' (6374792) panicked at src/main.rs:138:9:
dhat: assertion failed: `(left == right)`
  left: `3000`,
 right: `1`
```

Red, as expected: 3000 words, 3000 blocks.

## 5. Reuse a buffer, look up with &str

Two changes remove the per-word allocation:

- `normalize` becomes `normalize_into`: it writes into a `String` that the caller owns and clears it first. `count_words` creates that buffer once.
- The `HashMap` lookup uses `get_mut(word.as_str())`. A `HashMap<String, usize>` can be searched with a `&str`, because `String` implements `Borrow<str>`. Only a word that is not yet in the map is copied into a new `String` with `clone()`.

Replace `normalize` and `count_words`:

```rust
/// Lowercase letters and digits only: "The," -> "the".
/// Writes into `out`, which is cleared first, so one buffer can be reused.
#[inline(never)]
fn normalize_into(word: &str, out: &mut String) {
    out.clear();
    if word.is_ascii() {
        for byte in word.bytes().filter(u8::is_ascii_alphanumeric) {
            out.push(char::from(byte.to_ascii_lowercase()));
        }
    } else {
        // Rare path: the full Unicode rules, exactly as before.
        out.extend(word.to_lowercase().chars().filter(|c| c.is_alphanumeric()));
    }
}
```

```rust
#[inline(never)]
fn count_words(text: &str, counts: &mut HashMap<String, usize>) {
    let mut word = String::new(); // one buffer for all words
    for raw in text.split_whitespace() {
        normalize_into(raw, &mut word);
        if word.is_empty() || is_stop_word(&word) {
            continue;
        }
        // Look up with a &str: no new String for a word we already know.
        if let Some(count) = counts.get_mut(word.as_str()) {
            *count += 1;
        } else {
            counts.insert(word.clone(), 1);
        }
    }
}
```

**What happens:**

1. `out.clear()` sets the length to 0 but keeps the capacity. After the first few words the buffer is big enough for every word, and `push` never allocates again.
2. `String::new()` does not allocate at all; the first `push` does, once.
3. `get_mut(word.as_str())` hashes the text and finds the entry without creating anything. `insert(word.clone(), 1)` runs only 22,576 times, once per different word.
4. The test from Lesson 19.3 still calls `normalize`. Give the `tests` module a small helper with the old interface:

```rust
    /// Test helper: the old interface, built on `normalize_into`.
    fn normalize(word: &str) -> String {
        let mut out = String::new();
        normalize_into(word, &mut out);
        out
    }
```

Now the tests:

```bash
cargo test --features dhat-heap known_words
cargo test
```

```text
test tests::known_words_do_not_allocate ... ok
test tests::counts_and_top_on_a_small_text ... ok
test tests::fast_normalize_matches_the_reference ... ok
```

Green. The heap profile of the whole program:

```bash
cargo run --profile profiling --features dhat-heap --bin p19-wordfreq -- small.txt
```

```text
dhat: Total:     9,269,722 bytes in 45,177 blocks
dhat: At t-gmax: 7,000,205 bytes in 14,343 blocks
dhat: At t-end:  1,088 bytes in 2 blocks
```

From 1,022,599 blocks to 45,177: 22,576 for new words in the map, 22,575 for the copies in `top`, and a few others. The peak did not change (7 MB), because we did not change how the file is read. And the speed, on the big file, after checking that the output is identical (`before-buffer` is a copy of the binary from before this change):

```text
Benchmark 1: ./before-buffer words.txt (10 runs)
                mean     ±       σ          min     …     max
  Wall Time    1.187 s   ±   0.202 s      1.077 s   …   1.753 s
  Memory       131.1 MiB ±     0.6 MiB    129.6 MiB …   131.5 MiB
 
Benchmark 2: ./target/release/p19-wordfreq words.txt (10 runs)
                mean     ±       σ          min     …     max
  Wall Time    0.768 s   ±   0.084 s      0.714 s   …   0.953 s            -35.3%  (1.5x faster)
  Memory       131.0 MiB ±     0.6 MiB    129.6 MiB …   131.5 MiB            0.0%
```

1.5 times faster again. Together with Lesson 19.3, the program went from about 1.75 s to 0.77 s on this Mac, with the same output. Note the *Memory* line: 131 MiB in both versions. The peak is the next thing to fix (Exercise 3).

## 6. A toolbox for fewer allocations

| Pattern | Instead of | Do this |
|---|---|---|
| Reuse a buffer | `let s = make_string(x);` in a loop | one `String`/`Vec` outside the loop, `clear()` and fill it |
| Look up by borrowed key | `map.entry(key.to_string())` for every item | `map.get_mut(key)` first, allocate only for new keys |
| Borrow instead of clone | `Vec<(String, usize)>` copied from a map | `Vec<(&str, usize)>` that points into the map (Exercise 2) |
| Reserve once | `Vec::new()` and many `push` | `Vec::with_capacity(n)` when you know `n` |
| Format into a buffer | `format!(...)` or `to_string()` per item | `write!(buffer, ...)` into a reused `String` (`use std::fmt::Write`) |
| Stream the input | `fs::read_to_string` of a huge file | `BufReader` and `read_line` into a reused `String` (Exercise 3) |

> ⚡ **Performance:** do not apply these everywhere "just in case". They make code a little harder to read. Use them where a profiler shows `malloc`/`free` or where dhat shows a program point with millions of blocks, and keep a test that protects the result.

> 📏 **Convention:** functions that write into a buffer passed by the caller are often named `..._into` (`normalize_into`), like `read_to_string(&mut buf)` and `extend_from_slice` in the standard library. The caller owns the buffer and decides when to reuse it.

## Common mistakes

1. **Running without the feature.** `cargo run --profile profiling --bin p19-wordfreq -- small.txt` prints no dhat lines: the feature `dhat-heap` is off, so neither the allocator nor the profiler exists. Add `--features dhat-heap`.
2. **Naming the profiler `_`.** `let _ = dhat::Profiler::new_heap();` drops the profiler immediately, so it records nothing useful. Use a real name such as `_profiler`.
3. **Shipping with dhat on.** dhat's allocator made our program about 60 times slower. Keep it behind the feature and never enable the feature in production builds.
4. **Allocation tests next to other tests.** dhat counts the allocations of all threads. If other tests run in parallel, `total_blocks` includes their blocks and the test fails randomly. Run allocation tests alone, with a name filter as above, or with `cargo test --features dhat-heap -- --test-threads=1`.
5. **Huge `with_capacity`.** Reserving "enough for anything" (`String::with_capacity(1 << 20)` per item) trades allocation count for peak memory. Reserve what you need.

## Try it yourself

1. Open `dhat-heap.json` of the new version in the DHAT viewer (sorted by "Total (blocks)"). Which two program points are left at the top? (The `clone()` in `count_words` and the one in `top`.)
2. Sort by "At t-gmax (bytes)". Which program point holds the most memory at the peak? (Reading the file.)
3. Change the test to use `"x Y z, ".repeat(100_000)`. Does it still pass with `total_blocks == 1`?

## Exercises

### Exercise 1 — Read the numbers (easy)

The original version (Lesson 19.3) gave `Total: 22,929,129 bytes in 2,127,791 blocks` on `small.txt`; the version after Lesson 19.3 gave `1,022,599 blocks`; after this lesson, `45,177 blocks`. The file has 1,000,000 words. (a) How many blocks per word did each version allocate? (b) Why was the original version above 2?

<details>
<summary>Hint</summary>

Divide by one million. Then remember what the original `normalize` did: `to_lowercase` and then `collect` into a `String` that started empty.

</details>

<details>
<summary>Answer</summary>

(a) 2.13, 1.02 and 0.045 blocks per word. (b) The original made one block in `to_lowercase` and at least one in `collect`. `collect` starts with an empty `String` (the `filter` cannot know how many characters will pass), so for longer words the string had to grow, and growing means a new, bigger block (dhat counts a `realloc` as a new block). That gave about 0.1 extra blocks per word.

</details>

### Exercise 2 — Borrow instead of clone in top (medium)

Change `top` so that it returns `Vec<(&str, usize)>`, with string slices that point into the map instead of cloned `String`s. Update the test `counts_and_top_on_a_small_text` and measure with dhat. How many blocks are left?

<details>
<summary>Hint</summary>

`counts.iter().map(|(w, c)| (w.as_str(), *c))`. When you sort, `a.0` and `b.0` are already `&str`, so compare `a.0.cmp(b.0)`. In the test, the expected value becomes `vec![("x", 2), ("y", 2)]`.

</details>

<details>
<summary>Answer</summary>

```rust
fn top(counts: &HashMap<String, usize>, n: usize) -> Vec<(&str, usize)> {
    let mut all: Vec<(&str, usize)> = counts.iter().map(|(w, c)| (w.as_str(), *c)).collect();
    // Most frequent first; equal counts in alphabetical order.
    all.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(b.0)));
    all.truncate(n);
    all
}
```

In the test: `let expected = vec![("x", 2), ("y", 2)];`. The `println!` loop in `main` works unchanged. dhat now reports:

```text
dhat: Total:     8,702,549 bytes in 22,601 blocks
dhat: At t-gmax: 7,000,205 bytes in 14,343 blocks
dhat: At t-end:  1,088 bytes in 2 blocks
```

Why it works: the result borrows from `counts` (the compiler connects the lifetime of the returned `&str` to the `&HashMap` parameter by lifetime elision, Lesson 05.3), so no text is copied. 22,575 fewer blocks; what is left is essentially one block per different word, which the map really needs.

</details>

### Exercise 3 — Lower the peak: read in lines (hard)

The peak is the whole file in one `String`. Change `main` to read every file with `BufReader` and `read_line` into one reused `String`, calling `count_words` for each line. Check that the output on `words.txt` is identical, then measure with dhat (blocks and t-gmax) and with `hyperfine` (time and memory). Explain the trade-off.

<details>
<summary>Hint</summary>

`use std::fs::File; use std::io::{BufRead, BufReader};`. `read_line` *appends* to the string and returns the number of bytes read, 0 at the end of the file. Clear the line after each call.

</details>

<details>
<summary>Answer</summary>

```rust
    let mut line = String::new(); // one buffer for all lines
    for path in &paths {
        let mut reader = BufReader::new(File::open(path).expect("cannot open the file"));
        while reader.read_line(&mut line).expect("cannot read the file") > 0 {
            count_words(&line, &mut counts);
            line.clear();
        }
    }
```

dhat on `small.txt` (66,666 lines):

```text
dhat: Total:     4,860,756 bytes in 143,357 blocks
dhat: At t-gmax: 2,372,324 bytes in 22,584 blocks
dhat: At t-end:  1,088 bytes in 2 blocks
```

`hyperfine` on `words.txt` (`lines-wordfreq` is the new binary):

```text
Benchmark 1: ./target/release/p19-wordfreq words.txt (5 runs)
                mean     ±       σ          min     …     max
  Wall Time    791.1 ms  ±    89.2 ms     727.0 ms  …   944.1 ms
  Memory       130.0 MiB ±     0.6 MiB    129.2 MiB …   130.8 MiB
 
Benchmark 2: ./lines-wordfreq words.txt (5 runs)
                mean     ±       σ          min     …     max
  Wall Time    892.7 ms  ±    55.3 ms     842.4 ms  …   971.3 ms           +12.8%
  Memory         4.9 MiB ±     0.6 MiB      4.5 MiB …     5.6 MiB          -96.2%  (26.3x less)
```

Why it works: the peak no longer contains the file, so the heap peak fell from 7.0 to 2.4 MB on the small file, and the whole process needs 4.9 MiB instead of 130 MiB on the big one; memory no longer grows with the size of the file. The price: `count_words` creates its `word` buffer once per *call*, now once per line, so blocks went up from 22,601 to 143,357, and the program is about 13% slower. Fewer allocations and lower peak memory are different goals. (To get both, move the `word` buffer out of `count_words` and pass it in, or keep it in a struct together with the map.)

</details>

## Quiz

1. What is the difference between the allocation rate and peak memory? Give one fix for each.

<details>
<summary>Answer</summary>

The allocation rate is how many blocks are allocated and freed in total; it costs CPU (Central Processing Unit) time. Peak memory is the most memory held at one moment; it costs RAM. Reusing buffers lowers the rate; processing input in pieces lowers the peak.

</details>

2. Why is `dhat` put behind an optional feature instead of being always on?

<details>
<summary>Answer</summary>

It replaces the global allocator with a slow, recording one (about 60 times slower here). Normal builds and production must not pay that price or depend on the crate.

</details>

3. What do "At t-gmax" and "At t-end" tell you?

<details>
<summary>Answer</summary>

At t-gmax: how much memory was held at the moment of the biggest heap (the peak). At t-end: what was still allocated when the program ended; a value that grows with the input points to a leak.

</details>

4. Why can a `HashMap<String, usize>` be searched with a `&str`?

<details>
<summary>Answer</summary>

Because `String` implements `Borrow<str>`, and `HashMap::get` and `get_mut` accept any borrowed form of the key whose hash and equality match. So no temporary `String` is needed for a lookup.

</details>

5. Why must an allocation-counting test run alone?

<details>
<summary>Answer</summary>

dhat counts allocations of the whole process, from all threads. Tests run in parallel by default, so their allocations would be counted too and the numbers would change from run to run.

</details>

## Summary

- Allocations cost CPU time (allocation rate) and memory (peak); measure both before you fix either.
- `dhat` behind an optional feature (`dhat-heap = ["dep:dhat"]`) plus `#[global_allocator]` and `Profiler::new_heap()` gives Total, At t-gmax and At t-end, and a `dhat-heap.json` for the DHAT viewer.
- The viewer's program points, sorted by blocks or bytes, show exactly which line allocates.
- `Profiler::builder().testing()` and `HeapStats` turn "this code should not allocate" into a test.
- Reuse buffers (`..._into` functions), look up with `&str`, and borrow instead of cloning: our program went from 2.1 million to 23 thousand blocks and became faster.
- Streaming the input lowers the peak from the size of the file to a few megabytes, at a small cost in time.

## Sources and further reading

- [dhat crate documentation](https://docs.rs/dhat/latest/dhat/) — setup, the `dhat-heap` feature pattern, testing mode and `HeapStats`.
- [DHAT viewer](https://nnethercote.github.io/dh_view/dh_view.html) — the web page that opens `dhat-heap.json`.
- [Valgrind DHAT manual](https://valgrind.org/docs/manual/dh-manual.html) — the original tool; explains t-gmax, t-end, program points and sort metrics.
- [The Rust Performance Book — Heap Allocations](https://nnethercote.github.io/perf-book/heap-allocations.html) — many more patterns for allocating less in Rust.
- [The Cargo Book — Features](https://doc.rust-lang.org/cargo/reference/features.html) — `[features]`, optional dependencies and `dep:`.
- [`std::io::BufRead`](https://doc.rust-lang.org/std/io/trait.BufRead.html) — `read_line` and reading input in pieces.

---

[🏠 Start](../START-HERE.md) · [← 19.4 Profiling on Linux with perf](04-profiling-on-linux-with-perf.md) · [📋 Phase 19](README.md) · [19.6 Data layout for speed →](06-data-layout-for-speed.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
