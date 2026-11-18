<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 19.2 Load testing services](02-load-testing.md) · [📋 Phase 19](README.md) · [19.4 Profiling on Linux with perf →](04-profiling-on-linux-with-perf.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/19-performance-and-profiling/03-profilers-and-flame-graphs.md) <!-- nav -->

# Lesson 19.3 — Profilers and flame graphs

> **Phase 19 — Performance and profiling** · Lesson 3 of 11 · ⏱ about 135 minutes

## What you will learn

- What a **sampling profiler** does, and the difference between **self time** and **total time**.
- How to build a Rust program so that a profiler can name every function.
- How to record a profile on macOS with `samply` and explore it in the browser.
- How to read a **flame graph**, and how to make a flame graph file with `sample` and `inferno`.
- The full loop: profile, find the hot spot, change one thing, check the output is identical, measure again.

## Before you start

- You finished [Lesson 19.2 — Load testing services](02-load-testing.md). A load test tells you *that* something is slow; a profiler tells you *where*.
- You know `hyperfine` ([Lesson 09.11 — Benchmarking in depth](../09-testing-debugging-benchmarking/11-benchmarking-in-depth.md)) and Amdahl's law from the same lesson.
- You remember the call stack ([Lesson 07.7 — Recursion and the call stack](../07-algorithms-and-data-structures/07-recursion.md)) and how the heap allocator works ([Lesson 12.13 — How memory allocators work](../12-computer-systems/13-how-memory-allocators-work.md)).

## New words

| Word | Meaning |
|---|---|
| **profiler** | A tool that shows where a running program spends its time. |
| **sampling profiler** | A profiler that stops the program many times per second (for example 1000 times) and writes down the call stack each time. Functions that appear in many samples are where the time goes. |
| **sample** | One snapshot: "at this moment, this thread was running this chain of functions". |
| **call stack** | The chain of functions that are running right now: `main` called `count_words`, which called `normalize`, which called `to_lowercase`. |
| **self time** | Samples in which a function itself was running (it was at the top of the stack). |
| **total time** | Samples in which a function was anywhere on the stack: its own work plus everything it called. Also called *running time* or *inclusive time*. |
| **hot spot** | A small part of the code that takes a big share of the time. |
| **flame graph** | A picture of many stacks: each function is a box, its width shows its share of samples, the boxes above it are the functions it called. |
| **debug info** | Extra data in a binary that maps machine code back to function names, files and line numbers. |
| **symbol** / **symbolication** | A symbol is the name of a function in a binary. Symbolication turns raw code addresses from a profile into those names. |
| **inlining** | An optimization: the compiler copies a small function into its caller, so at run time there is no separate call. |
| **mangled name** | The encoded form of a Rust function name inside a binary, such as `_RNvCs7wEbHSVyFlW_12p19_wordfreq9normalize`. **Demangling** turns it back into `p19_wordfreq::normalize`. |

## 1. How a sampling profiler works

Imagine you want to know how a colleague spends the working day. You could follow them with a stopwatch and record every task (that is **instrumentation**, like the spans from Lesson 09.10). Or you could look at their screen once every ten minutes and write down what is on it. After a week, if "email" is on 40% of your notes, they spend about 40% of the day on email. That second method is **sampling**: cheap, and accurate enough when there are many samples.

A sampling profiler does the same with your program. About 1000 times per second it stops the thread for a moment and records the whole **call stack**:

```text
sample 1:  main > count_words > normalize > to_lowercase
sample 2:  main > count_words > normalize > collect > grow > malloc
sample 3:  main > count_words > free
sample 4:  main > count_words > normalize > to_lowercase
...
```

From thousands of such lines the profiler computes two numbers per function:

- **Self time:** in how many samples the function was at the *end* of the line (doing its own work). In the four samples above, `to_lowercase` has self time 2.
- **Total time:** in how many samples the function was *anywhere* in the line. `normalize` has total time 3, `count_words` has 4.

A function with a big total time but small self time is a "manager": it is slow because of what it calls. A function with big self time is doing the expensive work itself.

> ⚡ **Performance:** a profiler measures *where* time goes, not *how fast* the program is. Profiling slows the program down a little, and the numbers are approximate (a 2% function might show as 1% or 3%). For before/after speed, use `hyperfine` or `criterion`; use the profiler to decide *what* to change.

## 2. Building for profiling

A profiler needs two things from your binary:

1. **Optimized code.** A debug build spends its time in places a release build never does. Profile what you ship.
2. **Debug info**, so that the profiler can turn code addresses into function names, file names and line numbers, including functions that the optimizer **inlined** into their callers.

A release build has no debug info by default. Instead of changing `release`, add a separate **profile** to `Cargo.toml` that inherits everything from `release` and adds debug info:

```toml
[profile.profiling]
inherits = "release"
debug = true
```

Build with `cargo build --profile profiling`; the binaries go to `target/profiling/`. The machine code is the same as in `target/release/`, only the debug info is added.

> 💡 **Tip:** to walk the call stack quickly, profilers follow a chain of **frame pointers** that every function keeps on the stack. On Apple Silicon Macs the platform requires frame pointers, so this works out of the box. On Linux they are often missing; Lesson 19.4 shows what to do there.

## 3. A program to profile

We profile a word-frequency counter: it reads a big text file, turns every word into lowercase letters and digits, skips common "stop words", counts the rest in a `HashMap`, and prints the ten most frequent. It is correct, and it is slower than it needs to be. This package grows through Lessons 19.3 to 19.5.

```bash
cd ~/projects/rust/code
cargo new p19-wordfreq
cd p19-wordfreq
mkdir src/bin
echo "words.txt" >> .gitignore
```

The last line keeps the big test file out of Git. Now add the `[profile.profiling]` section from Section 2 to `Cargo.toml`.

First a generator for test data, `src/bin/gen.rs`. It writes made-up words built from syllables, always the same file for the same arguments:

```rust
//! Writes a big test file of pseudo-random English-like words.
//! Usage: cargo run --release --bin gen -- words.txt 25000000
use std::env;
use std::fs::File;
use std::io::{BufWriter, Write};

/// A tiny pseudo-random number generator (xorshift). Same seed, same file.
struct Rng(u64);

impl Rng {
    fn next(&mut self) -> u64 {
        self.0 ^= self.0 << 13;
        self.0 ^= self.0 >> 7;
        self.0 ^= self.0 << 17;
        self.0
    }
}

fn main() -> std::io::Result<()> {
    let args: Vec<String> = env::args().collect();
    let path = args.get(1).map(String::as_str).unwrap_or("words.txt");
    let count: usize = args.get(2).map(|s| s.parse().unwrap()).unwrap_or(3_000_000);

    let syllables = [
        "ka", "lo", "mi", "ne", "ru", "sta", "the", "an", "is", "or", "pre", "ing",
    ];
    let mut rng = Rng(0x2545_F491_4F6C_DD1D);
    let mut out = BufWriter::new(File::create(path)?);
    for i in 0..count {
        // Short words are much more common than long ones.
        let len = 1 + (rng.next() % 4) as usize * (rng.next() % 2) as usize;
        let mut word = String::new();
        for _ in 0..len {
            word.push_str(syllables[(rng.next() % syllables.len() as u64) as usize]);
        }
        if rng.next().is_multiple_of(10) {
            word = word.to_uppercase(); // some words are in capitals
        }
        let end = match rng.next() % 20 {
            0 => ". ",
            1 => ", ",
            _ => " ",
        };
        write!(out, "{word}{end}")?;
        if i % 15 == 14 {
            writeln!(out)?;
        }
    }
    out.flush()
}
```

Then the counter itself, `src/main.rs`:

```rust
//! Counts how often each word appears in text files and prints the top 10.
//! Usage: p19-wordfreq <file>...
use std::collections::HashMap;
use std::env;
use std::fs;

/// Words that are too common to be interesting.
const STOP_WORDS: [&str; 24] = [
    "a", "an", "and", "are", "as", "at", "be", "by", "for", "from", "has", "in", "is", "it", "its",
    "of", "on", "or", "that", "the", "to", "was", "were", "with",
];

/// Lowercase letters and digits only: "The," -> "the".
#[inline(never)]
fn normalize(word: &str) -> String {
    word.to_lowercase()
        .chars()
        .filter(|c| c.is_alphanumeric())
        .collect()
}

#[inline(never)]
fn is_stop_word(word: &str) -> bool {
    STOP_WORDS.contains(&word)
}

#[inline(never)]
fn count_words(text: &str, counts: &mut HashMap<String, usize>) {
    for raw in text.split_whitespace() {
        let word = normalize(raw);
        if word.is_empty() || is_stop_word(&word) {
            continue;
        }
        *counts.entry(word).or_insert(0) += 1;
    }
}

fn top(counts: &HashMap<String, usize>, n: usize) -> Vec<(String, usize)> {
    let mut all: Vec<(String, usize)> = counts.iter().map(|(w, c)| (w.clone(), *c)).collect();
    // Most frequent first; equal counts in alphabetical order.
    all.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)));
    all.truncate(n);
    all
}

fn main() {
    let paths: Vec<String> = env::args().skip(1).collect();
    assert!(!paths.is_empty(), "usage: p19-wordfreq <file>...");
    let mut counts = HashMap::new();
    for path in &paths {
        let text = fs::read_to_string(path).expect("cannot read the file");
        count_words(&text, &mut counts);
    }
    println!("{} different words", counts.len());
    for (word, count) in top(&counts, 10) {
        println!("{count:>8} {word}");
    }
}
```

**What happens:**

1. `main` reads each file into one `String` and passes it to `count_words`.
2. `count_words` splits the text at whitespace. For every piece it calls `normalize`, skips empty words and stop words, and adds 1 in the map. `entry(word).or_insert(0)` inserts 0 for a new word and returns a mutable reference to the counter.
3. `normalize` lowercases the whole word (`to_lowercase` creates a new `String`), then keeps only letters and digits and collects them into *another* new `String`.
4. `top` copies the map into a vector, sorts it and keeps the first `n`.
5. `#[inline(never)]` asks the compiler not to inline these small functions. Optimized code would normally melt them into `count_words`. `samply` can still show inlined functions thanks to debug info, but simpler tools such as `sample` (Section 6) cannot, so we keep them as real functions while we study the profile.

Generate 25 million words (about 130 MB, megabytes) and run the counter:

```bash
cargo run --release --bin gen -- words.txt 25000000
cargo build --release
time ./target/release/p19-wordfreq words.txt
```

```text
22616 different words
 1303606 mi
 1302073 ru
 1302017 pre
 1301813 ing
 1301789 lo
 1301765 sta
 1301048 ka
 1300485 ne
   22122 kaor
   22108 nesta
./target/release/p19-wordfreq words.txt  1.87s user 0.05s system 86% cpu 2.220 total
```

About two seconds. Where do they go? Before you read on, write down your guess. Most people guess "the `HashMap`".

## 4. Recording a profile with samply

`samply` is a sampling profiler for macOS and Linux. It records the profile and shows it in the **Firefox Profiler**, a web page that runs in your browser and reads the data from a small local web server that `samply` starts. Your data stays on your computer unless you press the upload button.

```bash
cargo install --locked samply
samply --version
```

```text
samply 0.13.1
```

Build the profiling binary and record. `--save-only` writes the file and does not open the browser yet:

```bash
cargo build --profile profiling
samply record --save-only -o profile.json.gz ./target/profiling/p19-wordfreq words.txt
```

The program runs normally and prints its usual output. Now open the profile:

```bash
samply load profile.json.gz
```

Your browser opens the Firefox Profiler. This is what you see (in words, because it is an interactive page):

- **At the top, the timeline:** one track for our thread with a graph of CPU (Central Processing Unit) use over the two seconds. You can drag over a part of it to look only at that time range.
- **Below it, tabs:** *Call Tree*, *Flame Graph*, *Stack Chart* and *Marker Chart*. Start with *Call Tree*.
- **The Call Tree** is a tree of functions, starting from `start` and `main`. Each row has *Running Time* (total samples, in milliseconds) and *Self* (self samples). Click the small triangles to open the tree. Following the biggest numbers, you reach `p19_wordfreq::count_words` (about 97% of the running time), and below it `p19_wordfreq::normalize` (about 50%).
- **Inside `normalize`** you find `<str>::to_lowercase` (about 15%) and the collection of characters into a `String`, where most of the time is in `RawVecInner::finish_grow` and allocator functions from `libsystem_malloc.dylib` (about 27%). That is the string growing, again and again, while characters are pushed into it.
- **Next to `normalize`**, directly under `count_words`, there is a big block in `libsystem_malloc.dylib` (around 15–20%): that is `free`, giving back the memory of the strings that were just created. On macOS it even calls `mach_absolute_time`, a clock function, inside the allocator.
- The `HashMap` work (`rustc_entry`, `hash_one`) is only about 4%, and reading the file (`read_to_string`) about 3%.
- **The "Invert call stack" checkbox** turns the tree upside down: the top rows are then the functions with the most *self* time. It is the quickest way to answer "which code is actually running?".
- **Double-click a function** to open its source code with the number of samples next to each line.

So the guess "the `HashMap`" was wrong. About **three quarters** of the time goes to creating and destroying small strings in `normalize`. This is typical: the slow part is very often not where you expect it. That is why we profile before we optimize.

> 🔒 **Security:** a profile contains function names, file paths and sometimes command-line arguments. The Firefox Profiler can upload a profile to share it as a link. Never upload profiles of private code or of programs that work on secret data.

## 5. Reading a flame graph

Click the *Flame Graph* tab. The same data is now a picture. A simplified drawing of ours:

```text
          | malloc |                               |
          | grow   |                               |
    |to_lowercase| collect into String  |          |
    |          normalize                | free     |hash|
    |                 count_words                       |read|
    |                        main                            |
     <---------------------- 100% of samples ---------------->
```

The rules:

1. **Each box is a function.** The box directly above another box is a function it called.
2. **Width is the share of samples** (total time). `normalize` is about half as wide as the whole picture, so it ran in about half of the samples.
3. **The x-axis is not time.** Boxes are sorted alphabetically (or by size), not by when they ran. To see the order in time, use the *Stack Chart* tab.
4. **The top edges are self time.** Where a box has nothing above it, the function itself was running. Wide "plateaus" at the top are the hot spots.
5. **Colours are just to tell boxes apart.** In some tools they mean the type of code (your code, the standard library, the system), never "hot" or "cold".

Read our graph from the bottom: almost everything is inside `count_words`; inside it, `normalize` and `free` are the two wide towers; inside `normalize`, `to_lowercase` and the growing string are the two wide parts. The `HashMap` is a thin box.

## 6. A flame graph file with sample and inferno

The browser view is the best for exploring. To put a flame graph into a report, a pull request or a chat, you want a file. On macOS you can make one with three small tools:

- `sample` is part of macOS. It attaches to a running process for some seconds and writes a text report of its call stacks.
- `inferno` (a Rust port of the original FlameGraph scripts) turns such reports into the "collapsed" format (one line per stack, with a count) and draws an SVG (Scalable Vector Graphics) image.
- `rustfilt` demangles Rust names.

```bash
cargo install --locked inferno rustfilt
```

`sample` needs the process ID (PID), so we start the program in the background with `&`. `$!` is the PID of the last background command. We wait 0.2 s so that reading the file is over, and sample for 1 second:

```bash
./target/profiling/p19-wordfreq words.txt > /dev/null &
sleep 0.2
sample $! 1 -mayDie -file wordfreq.sample.txt
```

```text
Sampling process 35889 for 1 second with 1 millisecond of run time between samples
Sampling completed, processing symbols...
Sample analysis of process 35889 written to file wordfreq.sample.txt
```

(`-mayDie` tells `sample` to collect symbols early, because our process may end before it finishes.) The report is plain text. Near the end it has a section that lists the functions with the most self time. Demangled with `rustfilt` (long names shortened here):

```bash
grep -A 12 'Sort by top of stack' wordfreq.sample.txt | rustfilt
```

```text
Sort by top of stack, same collapsed (when >= 5):
        _xzm_xzone_malloc_tiny  (in libsystem_malloc.dylib)        125
        mach_absolute_time  (in libsystem_kernel.dylib)        113
        p19_wordfreq::count_words  (in p19-wordfreq)        93
        _xzm_free_main  (in libsystem_malloc.dylib)        69
        <alloc::string::String as core::iter::traits::collect::FromIterator<char>>::from_iter::<...>  (in p19-wordfreq)        42
        <deduplicated_symbol>  (in libsystem_malloc.dylib)        39
        <str>::to_lowercase  (in p19-wordfreq)        33
        _malloc_zone_malloc  (in libsystem_malloc.dylib)        27
        _free  (in libsystem_malloc.dylib)        23
        <alloc::raw_vec::RawVecInner<_>>::reserve::do_reserve_and_handle::<...>  (in p19-wordfreq)        14
        <alloc::raw_vec::RawVecInner>::finish_grow  (in p19-wordfreq)        14
        __rustc::__rdl_alloc  (in p19-wordfreq)        11
```

The same story in numbers: `malloc`, the clock call inside `free`, `free` itself and string growing fill most of the list. Now draw the flame graph:

```bash
inferno-collapse-sample wordfreq.sample.txt | rustfilt | inferno-flamegraph > flame.svg
open -a Safari flame.svg
```

**What happens:**

1. `inferno-collapse-sample` reads the report and prints one line per distinct stack, such as `...;p19_wordfreq::count_words;p19_wordfreq::normalize;<str>::to_lowercase 33`.
2. `rustfilt` replaces every mangled name like `_RNvCs7wEbHSVyFlW_12p19_wordfreq9normalize` with `p19_wordfreq::normalize`.
3. `inferno-flamegraph` draws the SVG; `open -a Safari` shows it in a browser (any browser works; the file contains a little JavaScript that makes it interactive). Hover over a box to see its name and share, click it to zoom in, and use *Search* (top right) to highlight every box whose name matches, for example `malloc`.

In our file, `normalize` covers about 60% of the width, with `String` collection (about 40%) and `to_lowercase` (about 20%) on top of it; `free` covers about a quarter. The numbers differ a little from `samply` because `sample` ran for a shorter time and started after the file was read.

## 7. Fix one thing, check, measure

The profile says: `normalize` creates too many short-lived strings. `to_lowercase` builds one `String`, then `collect` builds a second one and grows it step by step. Our input is almost always ASCII (American Standard Code for Information Interchange: the 128 basic Latin letters, digits and symbols), and ASCII has a much simpler lowercase rule. A plan:

- For ASCII words: one `String` with the right capacity, filled byte by byte.
- For all other words: exactly the old code, so that the output stays the same.

First keep the old program, so you can compare: `cp src/main.rs src/bin/slow.rs`. Then replace `normalize` in `src/main.rs`:

```rust
fn normalize(word: &str) -> String {
    if word.is_ascii() {
        // Fast path: one allocation of the right size, byte by byte.
        let mut out = String::with_capacity(word.len());
        for byte in word.bytes().filter(u8::is_ascii_alphanumeric) {
            out.push(char::from(byte.to_ascii_lowercase()));
        }
        out
    } else {
        // Rare path: the full Unicode rules, exactly as before.
        word.to_lowercase()
            .chars()
            .filter(|c| c.is_alphanumeric())
            .collect()
    }
}
```

(Keep `#[inline(never)]` and the doc comment above it.) For an ASCII word the two paths give the same result: an ASCII letter's lowercase form is its ASCII lowercase form, and `is_alphanumeric` is true for exactly the ASCII letters and digits. Exercise 2 turns that claim into a test.

**Check the output first**, then measure. A faster program with a different answer is a bug:

```bash
cargo build --release
./target/release/slow words.txt > before.txt
./target/release/p19-wordfreq words.txt > after.txt
diff before.txt after.txt && echo "identical output"
hyperfine --warmup 1 --runs 10 './target/release/slow words.txt' './target/release/p19-wordfreq words.txt'
```

```text
identical output
Benchmark 1: ./target/release/slow words.txt (10 runs)
                mean     ±       σ          min     …     max
  Wall Time    1.745 s   ±   0.077 s      1.626 s   …   1.879 s
  Memory       130.6 MiB ±     0.8 MiB    129.1 MiB …   131.5 MiB
 
Benchmark 2: ./target/release/p19-wordfreq words.txt (10 runs)
                mean     ±       σ          min     …     max
  Wall Time    1.179 s   ±   0.047 s      1.134 s   …   1.280 s            -32.5%  (1.5x faster)
  Memory       131.1 MiB ±     0.5 MiB    130.4 MiB …   131.5 MiB           +0.4%
```

One small function, 1.5 times faster for the whole program, same output. Now **profile again**: `cargo build --profile profiling`, record and load as in Section 4. In the new profile `normalize` drops to about a third of the time, the big `free` block shrinks to a few percent, and the `HashMap` (hashing in `hash_one` and SipHash `write`, plus `rustc_entry`) has become one of the biggest parts at about 18%. Every fix moves the hot spot somewhere else. You stop when the program is fast enough for its job, not when the profile is "flat".

> 🧪 **Testing:** before every optimization, have a test that pins down the current behaviour, and keep the old version as a reference. Comparing the outputs of the old and new versions on many inputs is one of the strongest tests for an optimization.

## Common mistakes

1. **Profiling a debug build.** The profile then shows iterator adapters, bounds checks and other code that the release build removes. Always profile optimized code (`--profile profiling`).
2. **No debug info.** Without it you see raw addresses (`p19-wordfreq+0x1a2b4`) or only the few functions that were not inlined. Add `debug = true` to the profile you use for profiling.
3. **Optimizing without a profile.** Our guess was the `HashMap`; the truth was string allocation. Exercise 3 shows a "smart" change that made the program slower.
4. **Reading the x-axis of a flame graph as time.** It is sorted by name. A box on the left did not run first.
5. **A faster version that gives different answers.** A tempting one-pass version of `normalize` lowercases each character separately: `out.extend(c.to_lowercase())`. It is fast, but `str::to_lowercase` knows that the Greek capital sigma `Σ` becomes `ς` at the end of a word, and `char::to_lowercase` cannot know where a word ends. With that version, the test from Exercise 2 fails:

   ```text
   thread 'tests::fast_normalize_matches_the_reference' (6204647) panicked at src/main.rs:89:13:
   assertion `left == right` failed: input: "ΟΔΟΣ"
     left: "οδοσ"
    right: "οδος"
   ```

   Our test data is ASCII, so `diff` would not have noticed. That is why the fast path in Section 7 handles only ASCII and leaves everything else to the old code.

## Try it yourself

1. In the Firefox Profiler, tick *Invert call stack* and compare the top five rows with the `Sort by top of stack` list from `sample`. Are they the same functions?
2. Drag over the first 0.1 s of the timeline. What does the program do at the start? (Reading the file: `read` and the check that the bytes are valid UTF-8, the encoding of Rust strings from Lesson 12.5, in `from_utf8`.)
3. Run `samply record --save-only -o slow.json.gz ./target/profiling/slow words.txt` and load the slow and the fast profiles one after the other. Find where the hot spot moved.

## Exercises

### Exercise 1 — Read a flame graph (easy)

A flame graph of another program, simplified (widths in % of all samples):

```text
                 | sha256 30% |
| parse_json 25% | hash_password 32% |  render 38%                |
|                         handle_request 95%                       | log 5% |
|                                main 100%                                  |
```

(a) Which function has the largest total time after `main`? (b) Which has the largest self time? (c) `render` has nothing above it. What does that mean? (d) Is `hash_password` called before `render`?

<details>
<summary>Hint</summary>

Self time = width minus the widths of the boxes directly above. The x-axis is not time.

</details>

<details>
<summary>Answer</summary>

(a) `handle_request`, 95%. (b) `render`: 38% self, because nothing is above it. `sha256` has 30% self; `parse_json` has 25%; `hash_password` has only 32 − 30 = 2% self. (c) `render` itself (or code inlined into it) was running in all of its samples: it is a hot spot of its own. (d) We cannot know: boxes are sorted by name or size, not by time. Use a stack chart (timeline view) to see the order.

</details>

### Exercise 2 — A test that guards the optimization (medium)

Add a test module to `src/main.rs` that keeps the original `normalize` as `normalize_reference` and checks that the new `normalize` gives exactly the same result for these inputs: `"The,"`, `"HELLO"`, `"it's"`, `"--"`, `""`, `"a1b2"`, `"Straße"`, `"ÉCOLE"`, `"ΟΔΟΣ"`, `"日本語"`, `"🦀rust"`. Run it.

<details>
<summary>Hint</summary>

Put the inputs in an array and loop over them. Add the input to the assertion message (`"input: {word:?}"`), so a failure tells you *which* input broke.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    /// The first, slow version: our reference for correct answers.
    fn normalize_reference(word: &str) -> String {
        word.to_lowercase()
            .chars()
            .filter(|c| c.is_alphanumeric())
            .collect()
    }

    #[test]
    fn fast_normalize_matches_the_reference() {
        let words = [
            "The,",
            "HELLO",
            "it's",
            "--",
            "",
            "a1b2",
            "Straße",
            "ÉCOLE",
            "ΟΔΟΣ",
            "日本語",
            "🦀rust",
        ];
        for word in words {
            assert_eq!(
                normalize(word),
                normalize_reference(word),
                "input: {word:?}"
            );
        }
    }
}
```

`cargo test` runs the tests of every binary in the package; the line for `src/main.rs` shows:

```text
test tests::fast_normalize_matches_the_reference ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the inputs cover ASCII punctuation, digits, the empty string, and non-ASCII words that go through the slow path (German `ß`, an accented capital, Greek with a final sigma, Japanese, an emoji). With the one-pass version from Common mistakes, this test fails on `"ΟΔΟΣ"`.

</details>

### Exercise 3 — Amdahl and a "smart" change (hard)

In the profile of the fast version, `is_stop_word` and the `memcmp` calls it makes take about 10% of the samples. `STOP_WORDS` is sorted, so you could replace the linear `contains` with `STOP_WORDS.binary_search(&word).is_ok()` (at most 5 comparisons instead of up to 24). (a) Using Amdahl's law, what is the *best* possible speed-up for the whole program? (b) Make the change in a copy, measure it with `hyperfine` against the current version, and explain the result.

<details>
<summary>Hint</summary>

If a part takes 10% and becomes infinitely fast, the whole program takes 90% of the time. For (b), think about what `==` on two `&str` does first, and how predictable the branches of a binary search are (Lesson 12.6).

</details>

<details>
<summary>Answer</summary>

(a) At most 1 ÷ 0.9 ≈ 1.11, about 11% faster, and only if the stop-word check became free.

(b) Save the current binary first (`cp target/release/p19-wordfreq ascii-wordfreq`), change `is_stop_word`, rebuild, copy the new one to `bsearch-wordfreq`, and compare:

```text
Benchmark 1: ./ascii-wordfreq words.txt (10 runs)
                mean     ±       σ          min     …     max
  Wall Time    2.339 s   ±   0.385 s      1.721 s   …   2.948 s
  Memory       129.6 MiB ±     0.1 MiB    129.4 MiB …   129.7 MiB
 
Benchmark 2: ./bsearch-wordfreq words.txt (10 runs)
                mean     ±       σ          min     …     max
  Wall Time    3.164 s   ±   0.433 s      2.641 s   …   3.889 s            +35.3%  (1.4x slower)
  Memory       129.7 MiB ±     0.2 MiB    129.5 MiB …   130.4 MiB           +0.1%
```

(The Mac was busy during this run, so both times are higher than in Section 7; compare the two lines with each other.) Binary search made the program *slower*. Comparing two `&str` with `==` first compares their lengths, which rejects most stop words at once and is very predictable for the branch predictor. Binary search compares strings letter by letter (`"ka"` vs `"for"`), and which half it continues in is different for every word, so the CPU often guesses wrong. A smaller number of comparisons is not the same as less time. Keep the linear check, and keep the measurement in your notes.

</details>

## Quiz

1. What is the difference between self time and total time?

<details>
<summary>Answer</summary>

Self time counts the samples where the function itself was running (top of the stack). Total time counts the samples where it was anywhere on the stack, including time spent in the functions it called.

</details>

2. Why do we add `debug = true` in a separate `profiling` profile instead of profiling a debug build?

<details>
<summary>Answer</summary>

We want the optimized machine code of a release build (that is what runs in production) plus the names and line numbers from debug info. A debug build is unoptimized and spends its time in different places.

</details>

3. In a flame graph, a wide box has nothing above it. What does that tell you?

<details>
<summary>Answer</summary>

The function was running its own code in many samples: it has a large self time. It is a hot spot to look at.

</details>

4. Does the left-to-right order in a flame graph show what happened first?

<details>
<summary>Answer</summary>

No. Boxes are sorted (usually by name). Order in time is shown by a stack chart or timeline view.

</details>

5. A function takes 4% of the samples. You make it twice as fast. About how much faster is the whole program?

<details>
<summary>Answer</summary>

About 2%: the 4% part becomes 2%, so the program takes about 98% of the time before. Small parts of a profile can only give small gains (Amdahl's law).

</details>

6. Why did the profile show so much time in `malloc` and `free` for a program that "just counts words"?

<details>
<summary>Answer</summary>

Because `normalize` created two new `String`s for every one of 25 million words (one in `to_lowercase`, one in `collect`, which also had to grow) and dropped them again. Each allocation and free costs time in the allocator.

</details>

## Summary

- A **sampling profiler** records the call stack about 1000 times per second; functions that appear in many samples are where the time goes.
- **Total time** includes the callees; **self time** is the function's own work. Look for wide boxes with nothing above them.
- Profile **optimized code with debug info**: a `[profile.profiling]` that inherits `release` and sets `debug = true`.
- On macOS, `samply record` + `samply load` gives the Firefox Profiler with call tree, inverted view, flame graph and source view.
- `sample` + `inferno-collapse-sample` + `rustfilt` + `inferno-flamegraph` makes an SVG flame graph you can share.
- The loop: profile, find the hot spot, change one thing, **check the output is identical**, measure with `hyperfine`, profile again.
- Guesses about performance are often wrong, and "smarter" code can be slower. Measure.

## Sources and further reading

- [samply on GitHub](https://github.com/mstange/samply) — installation, `record` and `load`, and why Rust code needs debug info for full stacks.
- [Firefox Profiler documentation](https://profiler.firefox.com/docs/) — a tour of the call tree, flame graph and stack chart panels.
- [inferno on GitHub](https://github.com/jonhoo/inferno) — the collapse tools for many profilers and the flame graph renderer.
- [Brendan Gregg — Flame Graphs](https://www.brendangregg.com/flamegraphs.html) — the original explanation of flame graphs by their inventor.
- [The Rust Performance Book — Profiling](https://nnethercote.github.io/perf-book/profiling.html) — a list of profilers that work with Rust and tips for using them.
- [The Cargo Book — Profiles](https://doc.rust-lang.org/cargo/reference/profiles.html) — `inherits`, `debug` and the other profile settings.
- [rustfilt on GitHub](https://github.com/luser/rustfilt) — the demangler used in the pipeline.

---

[🏠 Start](../START-HERE.md) · [← 19.2 Load testing services](02-load-testing.md) · [📋 Phase 19](README.md) · [19.4 Profiling on Linux with perf →](04-profiling-on-linux-with-perf.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
