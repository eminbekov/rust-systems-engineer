<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 20.3 Fuzzing in depth](03-fuzzing-in-depth.md) · [📋 Phase 20](README.md) · [20.5 Fault injection and chaos testing →](05-fault-injection-and-chaos-testing.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/20-reliability-and-hardening/04-miri-sanitizers-and-loom-in-ci.md) <!-- nav -->

# Lesson 20.4 — Miri, sanitizers and loom in CI

> **Phase 20 — Reliability and hardening** · Lesson 4 of 11 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- Which kind of bug each tool finds: Miri, AddressSanitizer, ThreadSanitizer and loom, and why passing tests prove nothing about undefined behaviour.
- How to run your test suite under Miri, and how to keep slow tests out of it with `cfg(miri)`.
- How to build and test with AddressSanitizer and ThreadSanitizer on the nightly toolchain, including `-Zbuild-std`.
- How to run loom tests with a bounded search, and how to put all of it into one CI (continuous integration) workflow.

## Before you start

- You finished [Lesson 20.3 — Fuzzing in depth](03-fuzzing-in-depth.md).
- You know undefined behaviour and Miri from [Lesson 06.6 — Unsafe Rust and Miri](../06-rust-advanced/06-unsafe-rust-and-miri.md), how to review `unsafe` from [Lesson 10.12](../10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md), and loom from [Lesson 14.9 — Testing concurrent code with loom](../14-concurrency-and-async/09-testing-concurrent-code-with-loom.md).
- You need the nightly toolchain with the `miri` and `rust-src` components. You installed nightly with `miri` in Lesson 06.6; `rust-src` (the source code of the standard library, needed for `-Zbuild-std` in Section 5) is new. Run `rustup toolchain install nightly --component miri --component rust-src`: it adds whatever is missing and keeps what is already there.
- Create the package for this lesson:

```bash
cd ~/projects/rust
cargo new --lib code/p20-checks
cd code/p20-checks
mkdir tests
```

## New words

| Word | Meaning |
|---|---|
| **instrumentation** | Extra checking code that the compiler adds to your program, for example "before every memory read, check that this address is allowed". |
| **ASan** (AddressSanitizer) | A sanitizer that finds invalid memory use: reading outside an allocation, use after free, double free. It slows a program down about two times. |
| **TSan** (ThreadSanitizer) | A sanitizer that finds data races: two threads using the same memory at the same time without synchronization, at least one of them writing. |
| **LSan** (LeakSanitizer) | A sanitizer that reports memory that was allocated and never freed when the program ends. |
| **MSan** (MemorySanitizer) | A sanitizer that finds reads of uninitialized memory. It works only on Linux. |
| **ABI** (Application Binary Interface) | The low-level rules for how compiled code calls other compiled code: where arguments go, how values are laid out. Code built with different ABIs must not be mixed. |
| **`-Zbuild-std`** | A nightly Cargo option that compiles the standard library from source together with your code, with the same compiler flags. |
| **target triple** | The name of the platform you compile for, such as `aarch64-apple-darwin` (Apple Silicon macOS) or `x86_64-unknown-linux-gnu` (64-bit Linux on Intel or AMD (Advanced Micro Devices) processors). |
| **`cfg(miri)`** | A condition that is true only when the code runs under Miri. Used to skip or shrink tests that are too slow for it. |
| **preemption** | A moment where loom lets another thread run before the current one reaches a natural stop. Limiting preemptions limits the search. |
| **build matrix** | A CI feature that runs the same job several times with different values, for example once per sanitizer. |

## 1. Three tools, three questions

Normal tests answer "does the code give the right result *on this run*?". Code with `unsafe` or with threads needs stronger questions, because a wrong program can give the right result by luck.

| Tool | Question it answers | How it works | Cost |
|---|---|---|---|
| **Miri** | Did this run break any rule of Rust (undefined behaviour), including data races? | Interprets your program on a model of Rust's memory | Often hundreds of times slower; nightly; no FFI (Foreign Function Interface) calls into C |
| **ASan** | Did native code read or write memory it does not own? | Compiler instrumentation around every memory access | About 2 times slower; nightly |
| **TSan** | Did two threads race on memory in this run? | Instrumentation that records which thread touched what, and when | 5–15 times slower; nightly; needs `-Zbuild-std` |
| **loom** | Is there *any* thread schedule in which my test fails? | Runs the test again and again, once per possible schedule | Grows very fast with the number of threads and steps |

Miri and the sanitizers check **the paths your tests execute**, on one schedule per run (ASan, TSan) or a few (Miri). loom checks **all schedules**, but only of small models that you write with loom's types. They complement each other, and none of them replaces tests: they make tests stricter.

## 2. Code that passes its tests

Put this in `src/lib.rs`:

```rust
//! Small pieces of unsafe and concurrent code, checked by Miri, sanitizers and loom.

/// Adds the values in pairs: `[1, 2, 3, 4]` gives `[3, 7]`.
/// A last value without a partner stays alone: `[1, 2, 3]` gives `[3, 3]`.
pub fn pair_sums(values: &[u32]) -> Vec<u32> {
    let mut sums = Vec::with_capacity(values.len().div_ceil(2));
    let start = values.as_ptr();
    let mut i = 0;
    while i < values.len() {
        // SAFETY: (wrong!) we assume that every value has a partner at `i + 1`.
        let sum = unsafe { *start.add(i) + *start.add(i + 1) };
        sums.push(sum);
        i += 2;
    }
    sums
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn even_number_of_values() {
        assert_eq!(pair_sums(&[1, 2, 3, 4]), vec![3, 7]);
    }

    #[test]
    fn odd_number_of_values() {
        let values = vec![1, 2, 3];
        assert_eq!(pair_sums(&values), vec![3, 3]);
    }
}
```

**What happens:**

1. The loop reads `values[i]` and `values[i + 1]` through a raw pointer (Lesson 06.6), without bounds checks.
2. For an odd number of values the last step reads one element **past the end** of the slice. That is undefined behaviour. The `// SAFETY:` comment states the wrong assumption, which is exactly what a reviewer should catch (Lesson 10.12).
3. The second test even checks the odd case.

```bash
cargo test -q
cargo test -q --release
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
...
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Both pass, in debug and in release. The memory right after the three values happened to contain `0`, so `3 + 0` gave the "right" answer. On another machine, with another allocator or another compiler version, it could be any number, or a crash. This is why the tools below exist.

## 3. Miri on the whole test suite

In Lesson 06.6 you ran single programs with `cargo +nightly miri run`. In a real project you run the **whole test suite** under Miri:

```bash
cargo +nightly miri test
```

```text
running 2 tests
test tests::even_number_of_values ... ok
test tests::odd_number_of_values ... error: Undefined Behavior: memory access failed: attempting to access 4 bytes, but got alloc47039+0xc which is at or beyond the end of the allocation of size 12 bytes
  --> src/lib.rs:11:44
   |
11 |         let sum = unsafe { *start.add(i) + *start.add(i + 1) };
   |                                            ^^^^^^^^^^^^^^^^^ Undefined Behavior occurred here
   |
   = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
   = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
help: alloc47039 was allocated here:
  --> src/lib.rs:29:22
   |
29 |         let values = vec![1, 2, 3];
   |                      ^^^^^^^^^^^^^
...
error: test failed, to rerun pass `--lib`
```

**How to read it:** the allocation is 12 bytes (three `u32` values), and the program tried to read 4 bytes at offset `0xc` (12), right at the end. Miri shows the line of the bad read *and* the line where the memory was allocated. The first test passed under Miri: an even number of values never reaches the bug. Miri only checks the paths your tests run.

Two practical problems appear when a test suite grows:

- **Some tests are far too slow for Miri.** A test that handles a million values takes milliseconds natively and minutes under Miri. Mark it so that Miri skips it:

```rust
    #[test]
    #[cfg_attr(miri, ignore)] // a million values: seconds natively, many minutes under Miri
    fn a_million_values() {
        let values = vec![1; 1_000_000];
        assert_eq!(pair_sums(&values).len(), 500_000);
    }
```

  `cfg_attr(miri, ignore)` means "add the attribute `#[ignore]` only when `cfg(miri)` is true". Under Miri the result line then says `1 ignored`. A better way is often to keep the test but make it smaller under Miri with `cfg!(miri)`, a boolean you can use in normal code (Exercise 3).
- **Some tests call code that Miri cannot run**, such as C libraries through FFI (Lesson 06.10) or network calls. Skip those the same way.

> 💡 **Tip:** `MIRIFLAGS` passes options to Miri. `MIRIFLAGS="-Zmiri-many-seeds=0..16" cargo +nightly miri test` runs every test with 16 different random seeds, which changes the thread schedules and the addresses that Miri picks. Use it for tests with threads.

## 4. AddressSanitizer: memory errors in native code

Miri is precise but slow, and it cannot run C code. **ASan** checks the real, compiled program at almost full speed. You turn it on with a compiler flag, on nightly:

```bash
RUSTFLAGS="-Zsanitizer=address" cargo +nightly test -q --target aarch64-apple-darwin
```

```text
running 2 tests
=================================================================
==24999==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x60200000201c at pc 0x000100e44374 bp 0x00016f3d21b0 sp 0x00016f3d21a8
READ of size 4 at 0x60200000201c thread T2
    #0 0x000100e44370 in _RNvCskEp1fDRXMd0_10p20_checks9pair_sums lib.rs:11
    #1 0x000100e42d90 in _RNvNtCskEp1fDRXMd0_10p20_checks5testss_20odd_number_of_values lib.rs:30
...
0x60200000201c is located 0 bytes after 12-byte region [0x602000002010,0x60200000201c)
allocated by thread T2 here:
...
    #7 0x000100e42b54 in _RNvNtCskEp1fDRXMd0_10p20_checks5testss_20odd_number_of_values lib.rs:29
```

**What happens:**

1. `RUSTFLAGS` passes extra flags to the compiler for this one command. `-Zsanitizer=address` adds ASan's checks; `-Z` flags exist only on nightly.
2. `--target aarch64-apple-darwin` names the **target triple** of your Mac explicitly. With an explicit target, Cargo applies `RUSTFLAGS` only to your code and its dependencies, not to build scripts and procedural macros, which run on the build machine and must not be instrumented. On Linux CI the triple is `x86_64-unknown-linux-gnu`.
3. The report: a **heap-buffer-overflow**, a read of 4 bytes "0 bytes after a 12-byte region". The same bug Miri found, now found in the compiled program.
4. The names such as `_RNvCskEp1fDRXMd0_10p20_checks9pair_sums` are *mangled* symbol names: the compiler's encoded form of `p20_checks::pair_sums`. Read the readable part near the end, and the file and line.

On Linux, related sanitizers are available with the same flag: `-Zsanitizer=leak` (LSan, memory leaks) and `-Zsanitizer=memory` (MSan, uninitialized reads; needs `-Zbuild-std`, see below).

> ⚡ **Performance:** ASan makes a program about two times slower and uses more memory, so you can run your *whole* test suite with it, and even your fuzzers: `cargo fuzz` turns ASan on by default (Lesson 09.8). Miri is too slow for that; use it on the unit tests of the code that contains `unsafe`.

Now fix the bug. The best fix for `unsafe` code is often no `unsafe` code at all. `chunks(2)` gives slices of two values, and a last slice of one value for odd lengths:

```rust
pub fn pair_sums(values: &[u32]) -> Vec<u32> {
    values.chunks(2).map(|pair| pair.iter().sum()).collect()
}
```

Run `cargo +nightly miri test` again: both tests pass.

## 5. ThreadSanitizer: data races

Create `src/counter.rs` with a "fast" request counter that skips synchronization, and add `pub mod counter;` at the top of `src/lib.rs`:

```rust
//! A request counter shared by many threads.

use std::cell::UnsafeCell;

/// BUG: a "fast" counter without any synchronization.
pub struct RacyCounter {
    value: UnsafeCell<u64>,
}

// SAFETY: (wrong!) we claim that sharing between threads is fine.
unsafe impl Sync for RacyCounter {}

impl RacyCounter {
    pub fn new() -> Self {
        RacyCounter {
            value: UnsafeCell::new(0),
        }
    }

    pub fn increment(&self) {
        // SAFETY: (wrong!) two threads can be here at the same time.
        unsafe { *self.value.get() += 1 };
    }

    pub fn get(&self) -> u64 {
        // SAFETY: (wrong!) another thread may be writing right now.
        unsafe { *self.value.get() }
    }
}

impl Default for RacyCounter {
    fn default() -> Self {
        Self::new()
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn two_threads_count_requests() {
        let counter = RacyCounter::new();
        thread::scope(|s| {
            s.spawn(|| {
                for _ in 0..100 {
                    counter.increment();
                }
            });
            s.spawn(|| {
                for _ in 0..100 {
                    counter.increment();
                }
            });
        });
        assert_eq!(counter.get(), 200);
    }
}
```

**What happens:**

1. `UnsafeCell` (Lesson 06.4) allows changing a value through a shared reference; the `unsafe impl Sync` (Lesson 06.6) promises the compiler that this is safe across threads. The promise is false: `+= 1` is a read and a write, and two threads can interleave them.
2. `thread::scope` (Lesson 14.4) runs two threads that borrow `counter` and waits for both.

Run the test five times normally: `for i in 1 2 3 4 5; do cargo test -q counter; done`. It passes every time. One hundred increments are so quick that the two threads rarely overlap. Now run ThreadSanitizer the same way as ASan:

```bash
RUSTFLAGS="-Zsanitizer=thread" cargo +nightly test -q --target aarch64-apple-darwin counter
```

```text
error: mixing `-Zsanitizer` will cause an ABI mismatch in crate `p20_checks`
 |
 = help: the `-Zsanitizer` flag modifies the ABI so Rust crates compiled with different values of this flag cannot be used together safely
 = note: `-Zsanitizer=thread` in this crate is incompatible with `-Zsanitizer` being unset in dependency `std`
 = help: unset `-Zsanitizer` in this crate or set `-Zsanitizer=thread` in `std`
...
```

TSan must see *every* memory access, including those inside the standard library, and the standard library that comes with the toolchain was built without TSan. The two do not fit together: they have a different **ABI**. The fix is to build the standard library from source with the same flag, which is what `-Zbuild-std` does (it needs the `rust-src` component):

```bash
RUSTFLAGS="-Zsanitizer=thread" cargo +nightly test -q -Zbuild-std --target aarch64-apple-darwin counter
```

```text
running 1 test
==================
WARNING: ThreadSanitizer: data race (pid=26447)
  Read of size 8 at 0x00016b355368 by thread T3:
    #0 _RNvMs_NtCs1JKWEFB9EXy_10p20_checks7counterNtB4_11RacyCounter9increment counter.rs:22 (p20_checks-a99037034d792b30:arm64+0x100003450)
...
  Previous write of size 8 at 0x00016b355368 by thread T2:
    #0 _RNvMs_NtCs1JKWEFB9EXy_10p20_checks7counterNtB4_11RacyCounter9increment counter.rs:22 (p20_checks-a99037034d792b30:arm64+0x1000034d4)
...
SUMMARY: ThreadSanitizer: data race counter.rs:22 in _RNvMs_NtCs1JKWEFB9EXy_10p20_checks7counterNtB4_11RacyCounter9increment
==================
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 2 filtered out; finished in 0.17s

ThreadSanitizer: reported 1 warnings
error: test failed, to rerun pass `--lib`
```

**How to read it:** thread T3 read the counter at `counter.rs:22` while thread T2 had written the same 8 bytes at the same line, with nothing in between that orders the two. That is a data race. Note the end: the test's own assertion passed (`1 passed`), but TSan made the process fail when it exited, so the command fails and a CI job would fail too. The first build takes a minute or two, because the standard library is compiled.

Miri finds the same race, without any extra flags:

```bash
cargo +nightly miri test counter
```

```text
test counter::tests::two_threads_count_requests ... error: Undefined Behavior: Data race detected between (1) non-atomic write on thread `unnamed-2` and (2) non-atomic read on thread `unnamed-3` at alloc42582
  --> src/counter.rs:22:18
   |
22 |         unsafe { *self.value.get() += 1 };
   |                  ^^^^^^^^^^^^^^^^^^^^^^ (2) just happened here
...
```

The fix is the atomic counter from Lesson 14.5. Replace `RacyCounter` and its `impl` blocks (keep the test, but make it use `AtomicCounter::new()`):

```rust
/// The fixed counter: one atomic read-modify-write per increment.
pub struct AtomicCounter {
    value: crate::sync::AtomicU64,
}

impl AtomicCounter {
    pub fn new() -> Self {
        AtomicCounter {
            value: crate::sync::AtomicU64::new(0),
        }
    }

    pub fn increment(&self) {
        self.value.fetch_add(1, crate::sync::Ordering::Relaxed);
    }

    pub fn get(&self) -> u64 {
        self.value.load(crate::sync::Ordering::Relaxed)
    }
}

impl Default for AtomicCounter {
    fn default() -> Self {
        Self::new()
    }
}
```

`crate::sync` is the switch module from Lesson 14.9, which we add in the next section so that loom can replace the atomic type.

## 6. loom in CI

Set up loom exactly as in Lesson 14.9. Add to `Cargo.toml`:

```toml
[target.'cfg(loom)'.dependencies]
loom = "0.7"

[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ["cfg(loom)"] }
```

Create `src/sync.rs`, and add `mod sync;` to `src/lib.rs` next to `pub mod counter;`:

```rust
//! One place that decides where the sync types come from.

#[cfg(loom)]
pub(crate) use loom::sync::atomic::{AtomicU64, Ordering};

#[cfg(not(loom))]
pub(crate) use std::sync::atomic::{AtomicU64, Ordering};
```

In `src/counter.rs`, change the line `#[cfg(test)]` above the test module to `#[cfg(all(test, not(loom)))]`: that test uses `std::thread`, which must not run inside a loom build. Then create `tests/loom.rs`:

```rust
#![cfg(loom)]

use loom::sync::Arc;
use loom::thread;
use p20_checks::counter::AtomicCounter;

#[test]
fn two_threads_never_lose_an_increment() {
    loom::model(|| {
        let counter = Arc::new(AtomicCounter::new());
        let other = Arc::clone(&counter);
        let handle = thread::spawn(move || other.increment());
        counter.increment();
        handle.join().unwrap();
        assert_eq!(counter.get(), 2);
    });
}
```

```bash
LOOM_MAX_PREEMPTIONS=3 RUSTFLAGS="--cfg loom" CARGO_TARGET_DIR=target/loom cargo test --release --test loom
```

```text
running 1 test
test two_threads_never_lose_an_increment ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**What happens:**

1. `RUSTFLAGS="--cfg loom"` switches the `sync` module to loom's types, and `--test loom` runs only the loom test file (Lesson 14.9).
2. `LOOM_MAX_PREEMPTIONS=3` tells loom to explore only schedules with at most three forced thread switches. Most real bugs need only one or two, and without a limit the search for a bigger model can run for hours. In CI, a limit keeps the job time predictable.
3. `CARGO_TARGET_DIR=target/loom` keeps the loom build apart from the normal one, so neither has to rebuild everything (the tip from Lesson 14.9).

> 🧪 **Testing:** loom does not need nightly and finds logic races between *atomic* operations, which neither TSan nor Miri reports, because atomics never race in the TSan sense. TSan and Miri find races on *non-atomic* memory. You want both kinds of checks.

## 7. One workflow for all deep checks

These checks are slower than `cargo test`, so they get their own workflow next to the quality gate of Lesson 09.12. Save it as `.github/workflows/deep-checks.yml`:

```yaml
name: Deep checks

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  miri:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install nightly --profile minimal --component miri
      - run: cargo +nightly miri setup
      - run: cargo +nightly miri test

  sanitizers:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    strategy:
      fail-fast: false
      matrix:
        sanitizer: [address, thread]
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install nightly --profile minimal --component rust-src
      - name: Test with the ${{ matrix.sanitizer }} sanitizer
        env:
          RUSTFLAGS: -Zsanitizer=${{ matrix.sanitizer }}
        run: cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu --lib --tests

  loom:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v7
      - name: Explore thread schedules with loom
        env:
          RUSTFLAGS: --cfg loom
          LOOM_MAX_PREEMPTIONS: 3
        run: cargo test --release --test loom
```

**What happens:**

1. Three jobs run in parallel on fresh Linux machines. Each one stops after 30 minutes at most.
2. `cargo +nightly miri setup` prepares Miri's own copy of the standard library once, so the test step does not mix that work into its output.
3. The `sanitizers` job is a **build matrix**: it runs twice, once with `address` and once with `thread`. `fail-fast: false` lets the second run continue when the first one fails, so you see both reports.
4. On Linux CI the job uses `-Zbuild-std` for both sanitizers, as the official documentation recommends, and `--lib --tests` so that only the unit and integration tests are built with the sanitizer (documentation tests are compiled by a different tool that does not receive `RUSTFLAGS`).
5. The loom job runs on stable Rust: loom is a normal crate.

```bash
actionlint .github/workflows/deep-checks.yml && echo "actionlint: no problems"
```

```text
actionlint: no problems
```

(The workflow itself runs on GitHub's Linux machines, which this course cannot use. Every command in it was run on macOS with the target `aarch64-apple-darwin`; on Linux only the target triple changes.)

> 🔒 **Security:** Every `unsafe` block is a promise that a human checked. These tools check the promise on every pull request, long after the reviewer has moved on. For crates with `unsafe` code, a green Miri job is one of the strongest signals of quality you can show.

## Common mistakes

**1. "My tests pass, so the unsafe code is correct."** Section 2 showed a test that passes in debug and release while reading past the end of a buffer. Only a tool that checks the rules finds this.

**2. Forgetting `--target` with sanitizers.** Without an explicit target, `RUSTFLAGS` also applies to build scripts and procedural macros, which then fail to build or crash in confusing ways. Always pass `--target <triple>`.

**3. TSan without `-Zbuild-std`.** You get `error: mixing -Zsanitizer will cause an ABI mismatch` (Section 5). Add `-Zbuild-std` and make sure `rust-src` is installed for nightly.

**4. Running everything under Miri.** A test suite that takes ten seconds natively can take hours under Miri. Mark slow tests with `#[cfg_attr(miri, ignore)]` or shrink them with `cfg!(miri)`, and run Miri on the code that actually contains `unsafe`.

## Try it yourself

1. Create `code/p20-checks` with the buggy `pair_sums` and its two tests. Run `cargo test` and `cargo +nightly miri test`.
2. Run the ASan command and find the words `heap-buffer-overflow` and `lib.rs:11` in the report.
3. Fix `pair_sums` with `chunks(2)` and add the `#[cfg_attr(miri, ignore)]` test. Run Miri again: `2 passed; 0 failed; 1 ignored`.
4. Add `src/counter.rs` with `RacyCounter`. Run the test normally five times, then with TSan (first without `-Zbuild-std`, to see the error), then under Miri.
5. Replace it with `AtomicCounter`, add `src/sync.rs` and `tests/loom.rs`, and run the loom command.
6. Save the workflow and check it with `actionlint`. Then add `leak` to the sanitizer matrix (LeakSanitizer is available on `x86_64-unknown-linux-gnu`) and check it again.

## Exercises

### Exercise 1 — Pick the tool (easy)

Which tool (Miri, ASan, TSan, loom) would you use first for each problem, and why?

1. A C library that you call through FFI sometimes crashes with a corrupted heap.
2. A lock-free stack built from `AtomicPtr` sometimes loses an element under load.
3. A small `unsafe` function that converts `&[u8]` to `&[u32]` with a pointer cast.

<details>
<summary>Hint</summary>

Ask: is C code involved? Are the operations atomic? Is the code small and pure Rust?

</details>

<details>
<summary>Answer</summary>

1. **ASan.** Miri cannot run C code; ASan instruments native code and catches heap overflows and use-after-free at almost full speed.
2. **loom.** The bug is a logic race between atomic operations; loom explores all schedules of a small model. (Miri with many seeds can also help.)
3. **Miri.** Small, pure Rust `unsafe` code: Miri checks alignment, bounds and the validity of the produced references precisely.

</details>

### Exercise 2 — A test that reaches the bug (medium)

This function has an off-by-one error, but its tests pass, even under Miri. Write one more test that makes `cargo +nightly miri test` report the undefined behaviour. Then fix the function.

```rust
/// Returns the value at `index`, or 0 if there is no such value.
pub fn get_or_zero(values: &[u32], index: usize) -> u32 {
    if index <= values.len() {
        // SAFETY: we checked that `index` is inside `values`.
        unsafe { *values.as_ptr().add(index) }
    } else {
        0
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn reads_an_existing_value() {
        assert_eq!(get_or_zero(&[5, 6], 0), 5);
    }

    #[test]
    fn far_away_index_gives_zero() {
        assert_eq!(get_or_zero(&[5, 6], 9), 0);
    }
}
```

<details>
<summary>Hint</summary>

Test the edge case (Lesson 05.6): which index is exactly at the border of the check?

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod edge_tests {
    use super::*;

    #[test]
    fn index_equal_to_len_gives_zero() {
        let values = vec![5, 6];
        assert_eq!(get_or_zero(&values, 2), 0);
    }
}
```

With plain `cargo test` this test passes too (the byte after the vector happened to be 0). Under Miri:

```text
test edge_tests::index_equal_to_len_gives_zero ... error: Undefined Behavior: memory access failed: attempting to access 4 bytes, but got alloc40367+0x8 which is at or beyond the end of the allocation of size 8 bytes
  --> src/lib.rs:5:18
```

The fix is `if index < values.len()`. After it, all three tests pass under Miri. Why it works: Miri can only judge the paths that run; the edge-case test makes the bad path run. (A safe version, `values.get(index).copied().unwrap_or(0)`, needs no `unsafe` at all.)

</details>

### Exercise 3 — Smaller under Miri instead of skipped (medium)

Instead of ignoring the million-value test under Miri, write a test that uses 1 000 values under Miri and 1 000 000 otherwise, and checks that every sum is 2.

<details>
<summary>Hint</summary>

`cfg!(miri)` is a `bool` expression: `if cfg!(miri) { … } else { … }`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn many_values_but_fewer_under_miri() {
        let count = if cfg!(miri) { 1_000 } else { 1_000_000 };
        let values = vec![1; count];
        let sums = pair_sums(&values);
        assert_eq!(sums.len(), count / 2);
        assert!(sums.iter().all(|&sum| sum == 2));
    }
```

```text
test tests::many_values_but_fewer_under_miri ... ok
```

Why it works: the test still runs under Miri, so the code path is still checked for undefined behaviour, but with an input small enough to finish in about a second.

</details>

## Quiz

1. Why can a test with undefined behaviour pass in both debug and release builds?

<details>
<summary>Answer</summary>

Undefined behaviour does not have to crash. The program may read memory that happens to contain a harmless value. The result is luck, and it can change with another machine, allocator or compiler version.

</details>

2. What does Miri check that a normal test does not, and what is its main limit?

<details>
<summary>Answer</summary>

It checks every memory access and borrow of the executed code against Rust's rules (bounds, alignment, aliasing, initialization, data races). Its limits: it only checks the paths that run, it is very slow, and it cannot run foreign code such as C libraries.

</details>

3. Why does a sanitizer build need `--target <triple>`?

<details>
<summary>Answer</summary>

So that `RUSTFLAGS` with `-Zsanitizer=…` applies only to the code built for the target, and not to build scripts and procedural macros that run on the build machine during compilation.

</details>

4. What does the error "mixing -Zsanitizer will cause an ABI mismatch" mean, and how do you fix it?

<details>
<summary>Answer</summary>

Your crate is built with the sanitizer but the precompiled standard library is not, and the two are not compatible. Build the standard library from source with the same flags: add `-Zbuild-std` (with the `rust-src` component).

</details>

5. TSan reported a race but printed `1 passed`. Does the CI job pass?

<details>
<summary>Answer</summary>

No. TSan makes the test process fail when it exits after reporting warnings, so `cargo test` returns an error and the job fails.

</details>

6. Which tool finds a lost update between two `load` and `store` calls on an `AtomicU64`?

<details>
<summary>Answer</summary>

loom (and possibly Miri with many seeds). TSan does not report it, because atomic operations never form a data race; the bug is a logic race between correct atomic operations.

</details>

## Summary

- Passing tests prove nothing about undefined behaviour: wrong `unsafe` code often gives the right answer by luck.
- **Miri** (`cargo +nightly miri test`) checks the executed paths precisely, including data races; skip or shrink slow tests with `#[cfg_attr(miri, ignore)]` and `cfg!(miri)`.
- **ASan** (`RUSTFLAGS="-Zsanitizer=address" cargo +nightly test --target <triple>`) finds memory errors in compiled code, also in C, at about half speed.
- **TSan** finds data races in compiled code and needs `-Zbuild-std`; it fails the run when it reports.
- **loom** explores all schedules of a small model; `LOOM_MAX_PREEMPTIONS` bounds the search in CI.
- Put the deep checks into their own CI workflow: a Miri job, a sanitizer matrix and a loom job, each with a timeout, checked with `actionlint`.

## Sources and further reading

- [Miri](https://github.com/rust-lang/miri) — the official README: installation, `cargo miri test`, `MIRIFLAGS` options and CI advice.
- [The Unstable Book — sanitizer](https://doc.rust-lang.org/nightly/unstable-book/compiler-flags/sanitizer.html) — every sanitizer that Rust supports, on which targets, and the `-Zbuild-std` recommendation.
- [AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html) and [ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html) — how the sanitizers of LLVM (the compiler toolkit behind Rust) work and how to read their reports.
- [loom — documentation](https://docs.rs/loom/0.7.3/loom/) — `loom::model`, `LOOM_MAX_PREEMPTIONS` and the other environment variables.
- [The Cargo Book — Unstable features: build-std](https://doc.rust-lang.org/cargo/reference/unstable.html) — what `-Zbuild-std` does and its options.

---

[🏠 Start](../START-HERE.md) · [← 20.3 Fuzzing in depth](03-fuzzing-in-depth.md) · [📋 Phase 20](README.md) · [20.5 Fault injection and chaos testing →](05-fault-injection-and-chaos-testing.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
