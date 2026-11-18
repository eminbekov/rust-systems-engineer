<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 20.2 Errors and panics in production](02-errors-and-panics-in-production.md) · [📋 Phase 20](README.md) · [20.4 Miri, sanitizers and loom in CI →](04-miri-sanitizers-and-loom-in-ci.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/20-reliability-and-hardening/03-fuzzing-in-depth.md) <!-- nav -->

# Lesson 20.3 — Fuzzing in depth

> **Phase 20 — Reliability and hardening** · Lesson 3 of 11 · ⏱ about 135 minutes

## What you will learn

- How to fuzz a whole *sequence of operations* and compare your code with a simple model, so that the fuzzer finds wrong answers, not only crashes.
- Why a derived `Arbitrary` input can attack your fuzz target instead of your code, and how to shape the input with a hand-written `Arbitrary` implementation.
- How to work with the corpus: seed inputs, dictionaries and corpus minimization with `cargo fuzz cmin`.
- How continuous fuzzing works: short runs on every pull request, long runs every night, and a corpus that is kept between runs.

## Before you start

- You finished [Lesson 20.2 — Errors and panics in production](02-errors-and-panics-in-production.md).
- You know the basics of fuzzing from [Lesson 09.8 — Fuzzing](../09-testing-debugging-benchmarking/08-fuzzing.md): `cargo fuzz init`, a fuzz target, `cargo +nightly fuzz run`, crash artifacts, `tmin`, and the `arbitrary` crate with `#[derive(Arbitrary)]`. This lesson builds on all of it.
- You know the scheduled fuzzing workflow from [Lesson 09.12 — Continuous integration](../09-testing-debugging-benchmarking/12-continuous-integration.md) and `VecDeque` from [Lesson 07.5 — Stacks and queues](../07-algorithms-and-data-structures/05-stacks-and-queues.md).
- Create the package for this lesson (you need the nightly toolchain and `cargo-fuzz` from Lesson 09.8):

```bash
cd ~/projects/rust
cargo new --lib code/p20-fuzz
cd code/p20-fuzz
cargo fuzz init -t ops
cargo add arbitrary@1 --features derive --manifest-path fuzz/Cargo.toml
```

`cargo fuzz init -t ops` creates the `fuzz/` package with one target named `ops`. The last command adds `arbitrary` to the *fuzz* package, not to your library.

## New words

| Word | Meaning |
|---|---|
| **model** (test oracle) | A second, very simple implementation that is obviously correct, used only to check the real one. Here: a `VecDeque`. |
| **differential fuzzing** | Fuzzing two implementations with the same input and failing when their answers differ. |
| **stateful fuzzing** | Fuzzing a sequence of operations on an object (push, pop, …) instead of one function call. |
| **`Unstructured`** | The type from the `arbitrary` crate that wraps the fuzzer's raw bytes and hands them out piece by piece to build values. |
| **seed** | A hand-written input that you put into the corpus, so that the fuzzer starts from something meaningful. |
| **dictionary** | A file of byte strings (keywords, magic numbers) that the fuzzer may insert into inputs. |
| **corpus minimization** (`cmin`) | Removing corpus files that add no new coverage, so the corpus stays small and fast to replay. |
| **continuous fuzzing** | Running fuzzers automatically and regularly (on every change, every night), keeping the corpus between runs. |
| **OSS-Fuzz** (Open Source Software Fuzz) | Google's free service that fuzzes important open-source projects around the clock. |
| **OOM** (Out Of Memory) | A failure in which a program asks for more memory than the system allows. |

## 1. Beyond "does it crash?"

In Lesson 09.8 the fuzz target had one job: call a parser and see whether it panics. A round-trip assertion was a first step further. Here we go all the way: the fuzzer generates a **list of operations**, runs them on the real code and on a **model**, and checks after every step that both agree. This finds *wrong answers*, the silent failures of Lesson 20.1.

Our code is a buffer that keeps only the newest values, the way a service keeps "the last 100 log lines" for a debug page. Put this in `src/lib.rs`:

```rust
//! A buffer that keeps only the newest values, for example the last log lines of a service.

use std::num::NonZeroUsize;

/// Keeps at most `capacity` values. When it is full, a new value replaces the oldest one.
#[derive(Debug, Clone)]
pub struct RecentBuffer {
    slots: Vec<u32>,
    head: usize, // index of the oldest value
    len: usize,
}

impl RecentBuffer {
    pub fn new(capacity: NonZeroUsize) -> Self {
        RecentBuffer {
            slots: vec![0; capacity.get()],
            head: 0,
            len: 0,
        }
    }

    pub fn push(&mut self, value: u32) {
        let capacity = self.slots.len();
        let tail = (self.head + self.len) % capacity;
        self.slots[tail] = value;
        if self.len < capacity {
            self.len += 1;
        }
        // BUG: when the buffer was full, `value` replaced the oldest value,
        // so `head` must move to the next oldest one. It does not.
    }

    pub fn pop_oldest(&mut self) -> Option<u32> {
        if self.len == 0 {
            return None;
        }
        let value = self.slots[self.head];
        self.head = (self.head + 1) % self.slots.len();
        self.len -= 1;
        Some(value)
    }

    pub fn len(&self) -> usize {
        self.len
    }

    pub fn is_empty(&self) -> bool {
        self.len == 0
    }
}
```

**What happens:**

1. `NonZeroUsize` (from `std::num`) is a `usize` that can never be zero. Taking it as the capacity makes "a buffer with room for nothing" impossible, so `% capacity` can never divide by zero.
2. The values live in `slots`, used as a ring buffer (Lesson 07.5): `head` points at the oldest value, and the next free slot is `head + len`, wrapped around with `%`.
3. The comment marks a planted bug. Tests that never fill the buffer completely will not notice it. Add two such tests yourself (Try it yourself, step 1): they pass.

## 2. A stateful fuzz target with a model

Replace `fuzz/fuzz_targets/ops.rs`:

```rust
#![no_main]
//! Property: RecentBuffer behaves exactly like a simple model built on VecDeque.

use std::collections::VecDeque;
use std::num::NonZeroUsize;

use arbitrary::Arbitrary;
use libfuzzer_sys::fuzz_target;
use p20_fuzz::RecentBuffer;

#[derive(Debug, Arbitrary)]
enum Op {
    Push(u32),
    Pop,
}

#[derive(Debug, Arbitrary)]
struct Scenario {
    capacity: NonZeroUsize,
    ops: Vec<Op>,
}

fuzz_target!(|scenario: Scenario| {
    let capacity = scenario.capacity.get();
    let mut real = RecentBuffer::new(scenario.capacity);
    let mut model: VecDeque<u32> = VecDeque::new();

    for op in scenario.ops {
        match op {
            Op::Push(value) => {
                real.push(value);
                if model.len() == capacity {
                    model.pop_front();
                }
                model.push_back(value);
            }
            Op::Pop => assert_eq!(real.pop_oldest(), model.pop_front()),
        }
        assert_eq!(real.len(), model.len());
    }
});
```

**What happens:**

1. `Op` lists the operations; `#[derive(Arbitrary)]` lets the fuzzer pick one variant (and a `u32` for `Push`) from its bytes.
2. A `Scenario` is a capacity plus a list of operations of any length.
3. The model is a `VecDeque` with the rule written in the most obvious way: "if full, drop the front, then push at the back". It is too slow and too simple for production, and that is exactly why we trust it.
4. After every operation, both sides must give the same answer and the same length. The first difference is a failed `assert_eq!`, which the fuzzer reports like a crash.

Run it for at most 30 seconds:

```bash
cargo +nightly fuzz run ops -- -max_total_time=30
```

```text
INFO: A corpus is not provided, starting from an empty corpus
...
==4575==ERROR: AddressSanitizer: requested allocation size 0x1c00000002c2c (0x1c00000003c30 after adjustments for alignment, red zones etc.) exceeds maximum supported size of 0x10000000000 (thread T0)
...
Output of `std::fmt::Debug`:

	Scenario {
	    capacity: 123145302313739,
	    ops: [],
	}
```

(Shortened; your numbers will differ.) The fuzzer found a problem within a second, but not the one we planted. It chose a capacity of 123 trillion, and `vec![0; capacity]` asked for hundreds of terabytes of memory. AddressSanitizer (the memory checker that `cargo fuzz` turns on, Lesson 09.8) stopped the allocation as an **OOM**: out of memory.

Is this a bug in `RecentBuffer`? Not really: a caller who asks for a buffer of 123 trillion values gets what they asked for. It is a weakness of our **fuzz target**: the derived `Arbitrary` gives *every* possible `usize`, and almost all of them are useless for testing a ring buffer. We have to shape the input.

> 🔒 **Security:** If the capacity came from a configuration file or a request, this *would* be a real problem: one value could make the service allocate all memory. That is the "untrusted sizes" lesson from 10.12, and the limits of Lesson 20.8.

## 3. Shaping the input with a hand-written `Arbitrary`

The `Arbitrary` trait has one required method: build a value from an `Unstructured`. `Unstructured` wraps the fuzzer's bytes and offers helpers that take bytes from the front. Replace the `use arbitrary::Arbitrary;` line and the `Scenario` type in `ops.rs`:

```rust
use arbitrary::{Arbitrary, Unstructured};
```

```rust
#[derive(Debug)]
struct Scenario {
    capacity: NonZeroUsize,
    ops: Vec<Op>,
}

impl<'a> Arbitrary<'a> for Scenario {
    fn arbitrary(u: &mut Unstructured<'a>) -> arbitrary::Result<Self> {
        // A small capacity, 1 to 8, so that the buffer fills up and wraps around often.
        let extra = u.int_in_range(0..=7)?;
        let capacity = NonZeroUsize::MIN.saturating_add(extra);
        // The rest of the bytes become the list of operations.
        let ops: Vec<Op> = u.arbitrary()?;
        Ok(Scenario { capacity, ops })
    }
}
```

**What happens:**

1. `'a` is the lifetime of the fuzzer's byte slice (Lesson 06.5): an `Arbitrary` value may borrow from those bytes. We do not borrow anything, but the trait needs the parameter.
2. `u.int_in_range(0..=7)` takes a byte and turns it into a number from 0 to 7. `NonZeroUsize::MIN` is 1, and `saturating_add` adds without any risk of overflow, so the capacity is 1 to 8.
3. `u.arbitrary()` builds any type that implements `Arbitrary`, here the `Vec<Op>`, from the bytes that are left.
4. The `?` passes on `arbitrary::Error`, for example when the bytes run out. The fuzzer then simply skips this input.

Why 1 to 8? Small capacities mean the buffer fills up after a few pushes, so the interesting code path "full, replace the oldest" runs in almost every input. Shaping the input is how you *point* the fuzzer at the code you care about.

```bash
cargo +nightly fuzz run ops -- -max_total_time=30
```

```text
#3	INITED cov: 75 ft: 78 corp: 2/16b exec/s: 0 rss: 36Mb
#8	NEW    cov: 76 ft: 79 corp: 3/21b lim: 8 exec/s: 0 rss: 37Mb L: 5/8 MS: 5 CrossOver-ShuffleBytes-CrossOver-ChangeBinInt-CrossOver-
...
thread '<unnamed>' (6026437) panicked at fuzz_targets/ops.rs:48:24:
assertion `left == right` failed
  left: Some(421075225)
 right: Some(2602113305)
...
Output of `std::fmt::Debug`:

	Scenario {
	    capacity: 2,
	    ops: [
	        Push(
	            2913840473,
	        ),
	        Push(
	            2602113305,
	        ),
	        Push(
	            421075225,
	        ),
	        Pop,
	    ],
	}
```

**How to read it:** capacity 2, three pushes, one pop. The third push should replace the oldest value (2913840473), so the oldest is now 2602113305: that is what the model returned (`right`). The real buffer returned the *newest* value (`left`), because `head` still points at the slot that was just overwritten. A wrong answer, no crash, found in about ten thousand runs. The input is already tiny, so `tmin` is not needed.

Fix `push` in `src/lib.rs`:

```rust
        if self.len < capacity {
            self.len += 1;
        } else {
            // The buffer was full: `value` replaced the oldest value,
            // so the oldest one is now the next slot.
            self.head = (self.head + 1) % capacity;
        }
```

and keep the finding as a normal test (Lesson 09.8, Section 5), in a `tests` module at the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn cap(n: usize) -> NonZeroUsize {
        NonZeroUsize::new(n).unwrap()
    }

    #[test]
    fn found_by_fuzzing_a_full_buffer_drops_the_oldest_value() {
        let mut buffer = RecentBuffer::new(cap(2));
        buffer.push(1);
        buffer.push(2);
        buffer.push(3); // replaces 1
        assert_eq!(buffer.pop_oldest(), Some(2));
        assert_eq!(buffer.pop_oldest(), Some(3));
        assert_eq!(buffer.pop_oldest(), None);
    }
}
```

```bash
cargo test -q
cargo +nightly fuzz run ops -- -max_total_time=30
```

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
...
#8334597	DONE   cov: 109 ft: 400 corp: 87/18Kb lim: 4096 exec/s: 268857 rss: 630Mb
Done 8334597 runs in 31 second(s)
```

(Three tests: your two from step 1 of Try it yourself and the regression test.) Eight million scenarios without a difference.

> 💡 **Tip:** A model is the most powerful assertion you can give a fuzzer. Good models are everywhere: a `HashMap` for your own hash table, a `BTreeMap` for a B-tree (Phase 17), the old slow function for its new fast version. Comparing two implementations like this is called **differential fuzzing**.

> 📏 **Convention:** For a single field, the `arbitrary` crate also offers an attribute instead of a whole `impl`: `#[arbitrary(with = some_function)]`. Exercise 3 uses it. Write the full `impl` when several fields depend on each other.

## 4. Working with the corpus

The corpus in `fuzz/corpus/ops/` is the fuzzer's memory: every file is an input that reached new code. A good corpus makes the next run start where the last one stopped. Three tools help.

**Seeds.** You can put your own files into the corpus folder before the first run, for example real messages for a protocol parser, or the crash files of old bugs. The fuzzer reads them first and mutates them. For a parser of a text format, a few valid examples as seeds save the fuzzer a lot of guessing.

**Dictionaries.** A dictionary is a text file of byte strings, one per line in double quotes, which the fuzzer may insert into inputs. It helps with keywords and magic numbers that random mutation rarely finds. Create `fuzz/ops.dict`:

```text
# A capacity byte that gives the largest buffer, and two common values.
"\x07"
"\x00\x00\x00\x00"
"\xff\xff\xff\xff"
```

```bash
cargo +nightly fuzz run ops -- -dict=fuzz/ops.dict -max_total_time=3
```

```text
Dictionary: 3 entries
...
```

At the end of every run libFuzzer also prints a **recommended dictionary**: the byte strings that helped most in this run (`###### Recommended dictionary. ######` and lines such as `"\377\377" # Uses: 59733`). Copy useful lines into your dictionary file. For text formats such as HTTP (HyperText Transfer Protocol) or JSON (JavaScript Object Notation), the dictionary holds the words of the format: `"GET"`, `"Content-Length:"`, `"\r\n"`.

**Corpus minimization.** When several runs, on several machines or nights, add files to the same corpus, it fills up with inputs that cover the same code. `cargo fuzz cmin` replays all files and keeps only the smallest set with the same coverage. Here is a corpus of 170 files, made by two separate 20-second runs that were copied together:

```bash
ls fuzz/corpus/ops | wc -l
cargo +nightly fuzz cmin ops
ls fuzz/corpus/ops | wc -l
```

```text
     170
...
MERGE-OUTER: 170 files, 0 in the initial corpus, 0 processed earlier
...
MERGE-OUTER: 92 new files with 411 new features added; 120 new coverage edges
      92
```

The 92 remaining files reach exactly the same code (the same "features") as the 170 files did. A smaller corpus means faster start-up, which matters in CI (continuous integration), where every run starts by replaying the whole corpus.

> ⚡ **Performance:** A fuzz target should run in microseconds. Two tricks keep it fast: shape the input so that it is small (our capacity of 1 to 8), and avoid allocations and logging in the target. Watch `exec/s` in the progress lines: here about 270 000 runs per second.

## 5. Continuous fuzzing

A fuzzer that ran once found the bugs of that day. Code keeps changing, so fuzzing must keep running. **Continuous fuzzing** has three parts:

1. **A short run on every pull request**, for example 60 seconds per target. It catches the obvious crash that a change brings in, before it is merged.
2. **A long run every night**, for example 30 minutes, which goes deeper.
3. **A corpus that survives between runs**, so that each run continues where the last one stopped instead of starting from zero.

Here is the workflow from Lesson 09.12, grown into continuous fuzzing. Save it as `.github/workflows/fuzz.yml`:

```yaml
name: Fuzz

on:
  pull_request:
  schedule:
    - cron: "17 2 * * *" # every night at 02:17 UTC
  workflow_dispatch:

permissions:
  contents: read

jobs:
  fuzz:
    runs-on: ubuntu-latest
    timeout-minutes: 45
    env:
      # Short runs for pull requests, long runs at night.
      FUZZ_SECONDS: ${{ github.event_name == 'pull_request' && '60' || '1800' }}
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install nightly --profile minimal
      - run: cargo install --locked cargo-fuzz

      - name: Restore the corpus from earlier runs
        uses: actions/cache/restore@v6
        with:
          path: fuzz/corpus
          key: fuzz-corpus-${{ github.run_id }}
          restore-keys: fuzz-corpus-

      - name: Fuzz
        run: cargo +nightly fuzz run ops -- -max_total_time="$FUZZ_SECONDS"

      - name: Minimize the corpus
        if: github.event_name != 'pull_request'
        run: cargo +nightly fuzz cmin ops

      - name: Save the corpus for the next run
        if: github.event_name != 'pull_request'
        uses: actions/cache/save@v6
        with:
          path: fuzz/corpus
          key: fuzz-corpus-${{ github.run_id }}

      - name: Keep the crash files
        if: failure()
        uses: actions/upload-artifact@v7
        with:
          name: fuzz-artifacts
          path: fuzz/artifacts/
```

**What happens:**

1. The workflow runs on pull requests, every night (`schedule`), and by hand (`workflow_dispatch`).
2. `FUZZ_SECONDS` is 60 for a pull request and 1800 (30 minutes) otherwise. `a && b || c` is the expression language's way to write "if a then b else c".
3. `actions/cache/restore` looks for a cache with the exact key (this run's ID, its identifier, which never exists yet) and then falls back to the newest cache whose key starts with `fuzz-corpus-`: the corpus of the last nightly run.
4. Only the nightly run minimizes and saves the corpus, under a new key. Caches cannot be overwritten, which is why every save gets a new key, and old ones expire on their own. Pull requests only read it.
5. If the fuzzer finds a crash, the job fails, and the artifact files are uploaded for you to reproduce at home.

Check it before you push it, as in Lesson 09.12:

```bash
actionlint .github/workflows/fuzz.yml && echo "actionlint: no problems"
```

```text
actionlint: no problems
```

(We cannot run GitHub's servers from this course; `actionlint` checks the syntax, the expressions and the inputs of the actions.)

For open-source projects there are free services that do all of this at a much larger scale. **OSS-Fuzz** fuzzes important open-source projects on Google's machines, day and night, and reports bugs privately to the maintainers; many Rust crates are in it. **ClusterFuzzLite** is a smaller version of the same system that runs inside your own CI. Both use the same `cargo fuzz` targets that you write in this lesson.

> 🧪 **Testing:** Fuzzing does not replace `cargo test`. Normal tests check known cases in milliseconds on every build; the fuzzer looks for unknown cases in minutes and hours. Every finding becomes a normal test, so both grow together.

## Common mistakes

**1. Letting the fuzzer pick huge sizes.** A derived `Arbitrary` for a capacity, a length or a count produces values up to `usize::MAX`. The fuzzer then reports OOM in your *target* (Section 2). Shape such values with `int_in_range`.

**2. A model that shares code with the real implementation.** If the model calls the same helper function as the real code, both can be wrong in the same way, and the fuzzer sees no difference. Keep the model independent and as simple as possible.

**3. Changing the input format and keeping the old corpus.** After you change `Scenario` or `Op`, old corpus files decode into different values. That is not wrong, but the corpus loses much of its value. Run `cmin` after big changes, or start a fresh corpus.

**4. Committing the corpus to Git.** It changes on every run and grows large. Keep it in a CI cache (Section 5) or a separate storage place; commit only targets, dictionaries, seeds and regression tests.

## Try it yourself

1. Create `code/p20-fuzz` and add `src/lib.rs` with the bug. Add two tests that do not fill the buffer: one that pushes 1 and 2 into a buffer of 3 and pops them, one that pushes 1 and 2 into a buffer of 2, pops once, pushes 3 and pops twice. Run `cargo test`: both pass, despite the bug.
2. Write the derived-`Arbitrary` target and see the OOM report.
3. Replace `Scenario` with the hand-written `Arbitrary` and let the fuzzer find the wrong answer. Fix `push`, add the regression test, and run the fuzzer again for 30 seconds.
4. Make `fuzz/ops.dict` and run with `-dict=fuzz/ops.dict`. Look for `Dictionary: 3 entries`.
5. Run `cargo +nightly fuzz cmin ops` and compare the number of corpus files before and after.

## Exercises

### Exercise 1 — Read the report (easy)

In the report of Section 3, which value did the real buffer return, which value did the model return, and which of the two is right? Explain in one sentence why the real buffer gave its answer.

<details>
<summary>Hint</summary>

`left` is the first argument of `assert_eq!`, here `real.pop_oldest()`.

</details>

<details>
<summary>Answer</summary>

The real buffer returned 421075225 (the third, newest value); the model returned 2602113305 (the second value). The model is right: after the third push into a buffer of 2, the first value is gone and the second is the oldest. The real buffer wrote the new value into the oldest slot but did not move `head`, so `head` pointed at the newest value.

</details>

### Exercise 2 — A new operation, with a test (medium)

Add `pub fn peek_oldest(&self) -> Option<u32>` to `RecentBuffer`: it returns the oldest value without removing it. Write a unit test for it that includes a full buffer. Then add `Peek` to `Op` and check it against the model in the fuzz target. Fuzz for 10 seconds.

<details>
<summary>Hint</summary>

The model's version is `model.front().copied()`: `front()` gives `Option<&u32>`, and `copied()` turns it into `Option<u32>`.

</details>

<details>
<summary>Answer</summary>

In `src/lib.rs`, inside `impl RecentBuffer`:

```rust
    /// The oldest value, without removing it.
    pub fn peek_oldest(&self) -> Option<u32> {
        if self.len == 0 {
            None
        } else {
            Some(self.slots[self.head])
        }
    }
```

and in the `tests` module:

```rust
    #[test]
    fn peek_shows_the_oldest_value_and_keeps_it() {
        let mut buffer = RecentBuffer::new(cap(2));
        assert_eq!(buffer.peek_oldest(), None);
        buffer.push(1);
        buffer.push(2);
        buffer.push(3);
        assert_eq!(buffer.peek_oldest(), Some(2));
        assert_eq!(buffer.len(), 2);
    }
```

In `fuzz/fuzz_targets/ops.rs`, add the variant `Peek,` to `Op` and this arm to the `match`:

```rust
            Op::Peek => assert_eq!(real.peek_oldest(), model.front().copied()),
```

```text
#2659717	DONE   cov: 117 ft: 457 corp: 100/22Kb lim: 4096 exec/s: 241792 rss: 423Mb
Done 2659717 runs in 11 second(s)
```

Why it works: the unit test pins down one known case, including the full-buffer case that hid the first bug; the fuzz target checks `peek_oldest` after every possible sequence of operations.

</details>

### Exercise 3 — The `with` attribute (medium)

Replace the hand-written `impl Arbitrary for Scenario` by `#[derive(Arbitrary)]` plus a field attribute `#[arbitrary(with = small_capacity)]`, where `small_capacity` is a function that you write. The behaviour must stay the same (capacity 1 to 8).

<details>
<summary>Hint</summary>

The function has the signature `fn small_capacity(u: &mut Unstructured) -> arbitrary::Result<NonZeroUsize>`.

</details>

<details>
<summary>Answer</summary>

```rust
/// A capacity from 1 to 8, so that the buffer fills up and wraps around often.
fn small_capacity(u: &mut Unstructured) -> arbitrary::Result<NonZeroUsize> {
    let extra = u.int_in_range(0..=7)?;
    Ok(NonZeroUsize::MIN.saturating_add(extra))
}

#[derive(Debug, Arbitrary)]
struct Scenario {
    #[arbitrary(with = small_capacity)]
    capacity: NonZeroUsize,
    ops: Vec<Op>,
}
```

Why it works: the derive generates the same code as our `impl`, but calls `small_capacity` for the one field that needs special treatment. A 10-second run finishes with `DONE` and no failure.

</details>

### Exercise 4 — Bug or not? Ask the fuzzer (hard)

Start from the fixed `src/lib.rs`. Try two changes in `pop_oldest`, one at a time, each with a fresh corpus (`rm -rf fuzz/corpus`) and a 20-second run. Before you run, predict which change is a bug.

- **Change A:** after `self.len -= 1;` add `if self.len == 0 { self.head = 0; }`.
- **Change B:** replace `self.head = (self.head + 1) % self.slots.len();` by `self.head += 1;`.

<details>
<summary>Hint</summary>

When the buffer is empty, does it matter where `head` points? And what happens to `head` after many pops without `%`?

</details>

<details>
<summary>Answer</summary>

Change A is **not** a bug: when the buffer is empty, the next push writes at `head + 0`, so any position works. The fuzzer agrees: `Done 7114681 runs in 21 second(s)`, no failure.

Change B **is** a bug: `head` grows past the end of `slots`. The fuzzer finds it within seconds:

```text
thread '<unnamed>' (6077894) panicked at src/lib.rs:39:31:
index out of bounds: the len is 2 but the index is 2
```

with a scenario of capacity 2 and alternating `Pop` and `Push` operations. (The report shows the full path of `src/lib.rs`.) Why it matters: the fuzzer is a quick way to test your *reasoning* about a change, not only to find bugs you did not suspect.

</details>

## Quiz

1. What does a model add to a fuzz target that a plain "call it and see whether it panics" target does not have?

<details>
<summary>Answer</summary>

A precise expectation for every step. The fuzzer then finds wrong answers (silent failures), not only panics.

</details>

2. Why did the first version of the target report an out-of-memory error, and whose fault was it?

<details>
<summary>Answer</summary>

The derived `Arbitrary` produced a gigantic `NonZeroUsize` capacity, and `vec![0; capacity]` tried to allocate it. It was a weakness of the fuzz target, not a bug in the buffer.

</details>

3. What does `u.int_in_range(0..=7)` do?

<details>
<summary>Answer</summary>

It takes bytes from the front of the `Unstructured` input and turns them into a number from 0 to 7 (inclusive).

</details>

4. What is the difference between a seed and a dictionary?

<details>
<summary>Answer</summary>

A seed is a whole input file placed in the corpus as a starting point. A dictionary is a list of small byte strings (keywords, magic numbers) that the fuzzer may insert into any input.

</details>

5. What does `cargo fuzz cmin` keep, and why do you want that in CI?

<details>
<summary>Answer</summary>

The smallest set of corpus files with the same coverage as the whole corpus. Every CI run starts by replaying the corpus, so a smaller corpus means faster runs with no loss.

</details>

6. In the workflow, why does every save of the corpus use a new cache key?

<details>
<summary>Answer</summary>

A cache entry cannot be overwritten once it is saved. A new key per run (`github.run_id`) together with `restore-keys: fuzz-corpus-` always finds the newest corpus and saves a newer one.

</details>

## Summary

- Stateful, model-based fuzzing runs a generated *sequence* of operations on the real code and on a simple model, and fails at the first difference. It finds wrong answers, not only crashes.
- A derived `Arbitrary` may produce values that attack the fuzz target (huge sizes, OOM). Shape the input with a hand-written `impl Arbitrary` or `#[arbitrary(with = …)]`, using helpers such as `int_in_range`.
- Shaping also *points* the fuzzer at the interesting paths (small capacities fill up quickly).
- The corpus is the fuzzer's memory: seed it with real examples, help it with a dictionary, and shrink it with `cargo fuzz cmin`.
- Continuous fuzzing = short runs on pull requests, long runs at night, a corpus kept in a cache, and crash files uploaded. OSS-Fuzz and ClusterFuzzLite do the same at scale.
- Every finding becomes a normal regression test.

## Sources and further reading

- [Rust Fuzz Book](https://rust-fuzz.github.io/book/) — the official guide to `cargo fuzz`, including [structure-aware fuzzing](https://rust-fuzz.github.io/book/cargo-fuzz/structure-aware-fuzzing.html).
- [arbitrary — documentation](https://docs.rs/arbitrary/1.5.0/arbitrary/) and [Unstructured](https://docs.rs/arbitrary/1.5.0/arbitrary/struct.Unstructured.html) — the trait, the derive attributes and all helper methods.
- [libFuzzer documentation](https://llvm.org/docs/LibFuzzer.html) — the meaning of the progress lines, dictionaries, `-merge` (what `cmin` uses) and all options.
- [actions/cache](https://github.com/actions/cache) — the `restore` and `save` actions and how cache keys work.
- [OSS-Fuzz](https://google.github.io/oss-fuzz/) and [ClusterFuzzLite](https://google.github.io/clusterfuzzlite/) — continuous fuzzing as a service and inside your own CI.

---

[🏠 Start](../START-HERE.md) · [← 20.2 Errors and panics in production](02-errors-and-panics-in-production.md) · [📋 Phase 20](README.md) · [20.4 Miri, sanitizers and loom in CI →](04-miri-sanitizers-and-loom-in-ci.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
