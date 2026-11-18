<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 09.7 Code coverage and mutation testing](07-code-coverage-and-mutation-testing.md) · [📋 Phase 09](README.md) · [09.9 Debugging in depth: LLDB and RustRover →](09-debugging-with-lldb-and-rustrover.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/08-fuzzing.md) <!-- nav -->

# Lesson 09.8 — Fuzzing

> **Phase 09 — Testing, debugging and benchmarking** · Lesson 8 of 14 · ⏱ about 135 minutes

## What you will learn

- What fuzzing is, and why a program that feeds itself random-looking input finds bugs that hand-written tests miss.
- How to install `cargo-fuzz`, create a fuzz target, run it, and read the crash report.
- How to reproduce, shrink and fix a crash, and how to turn it into a normal regression test.
- How to fuzz with structured input (the `arbitrary` crate) and check a property such as "encode, then decode, gives back the original".
- What fuzzing cannot do, and how to run it in a time-boxed way.

## Before you start

- You finished [Lesson 09.7 — Code coverage and mutation testing](07-code-coverage-and-mutation-testing.md).
- You know property-based testing from [Lesson 09.4](04-property-based-testing.md): a test that checks a rule for many generated inputs. Fuzzing is the "much more input, much smarter generation" cousin of that idea.
- You installed the nightly toolchain in [Lesson 06.6 — Unsafe Rust and Miri](../06-rust-advanced/06-unsafe-rust-and-miri.md). If you did not, run `rustup toolchain install nightly --component miri` once.
- You know panics and slices: [Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md) and [Lesson 03.6](../03-ownership-and-borrowing/06-slices-and-string-types.md).
- Your code for this lesson goes in `~/projects/rust/code/p09-fuzzing`.

## New words

| Word | Meaning |
|---|---|
| **fuzzing** | Running your code again and again with automatically generated, often strange input, looking for crashes. |
| **fuzzer** | The program that generates the inputs and watches for failures. |
| **fuzz target** | A tiny function you write: "take these bytes, give them to my code". The fuzzer calls it millions of times. |
| **coverage-guided** | The fuzzer watches which lines of your code each input reaches. An input that reaches a new line is kept and mutated further. |
| **corpus** | The folder of interesting inputs that the fuzzer has collected. It grows while the fuzzer runs. |
| **mutation** | A small random change of a stored input: flip a bit, insert a byte, copy a piece. |
| **crash artifact** | The file that holds the exact input that made the program fail. |
| **libFuzzer** | The fuzzing engine from the LLVM project (the compiler toolkit that Rust uses). `cargo-fuzz` drives it. |
| **sanitizer** | A tool built into the compiler that adds checks to the program while it runs. `cargo-fuzz` turns on AddressSanitizer, which catches invalid memory use. |
| **minimizing** (`tmin`) | Making a crashing input as small as possible, so it is easy to understand. |
| **structured fuzzing** | Fuzzing with typed input (for example a struct) instead of raw bytes. |
| **round trip** | Converting a value to another form and back (encode, then decode). The result must equal the start. |

## 1. Why fuzzing?

You write a function that reads data from outside: a file, a network message, text typed by a person. You test it with five inputs that you can imagine. Then a user sends the five-thousand-and-first input, one you never imagined, and the program panics.

**Fuzzing** automates that "unimaginable" input. A *fuzzer* generates thousands to millions of inputs per second, runs your code on each, and reports the first input that makes it panic, overflow, run out of memory or hang.

Random bytes alone would waste time. Most random bytes fail at the very first check of your parser. Modern fuzzers are **coverage-guided**: the compiler adds small counters to your code, and the fuzzer sees which parts of the code each input reached. The loop looks like this:

```text
  ┌──────────────┐  pick one   ┌───────────────┐  change a few   ┌──────────────┐
  │   corpus     │ ──────────▶ │ stored input  │ ──────────────▶ │ new mutation │
  │ (saved files)│             └───────────────┘   bytes         └──────┬───────┘
  └──────▲───────┘                                                      │ run your code
         │ reached a NEW line?                                          ▼
         └───────────────── yes: keep it ◀──────────────  ┌────────────────────────┐
                                                          │ crash? save the input  │
                                                          └────────────────────────┘
```

Because it keeps every input that unlocks a new branch, the fuzzer slowly "learns" the shape of valid input: it finds the right first byte, then the right second byte, and so on.

> 🔒 **Security:** Fuzzing is one of the main ways that real parsers (image decoders, network protocols, file formats) get their serious bugs found. Any code that reads bytes from outside your program is a good fuzz candidate, because an attacker can send exactly the strange input that a fuzzer finds.

Fuzzing is different from the tests you wrote before. A unit test says "for this input, the answer is X". A fuzz target says only "for **any** input, do not crash" (and, with assertions, "and keep this rule true"). It does not know the right answers; it knows only the rules.

## 2. Installing and starting

Fuzzing in Rust needs two things: the tool `cargo-fuzz` and the **nightly** compiler. `cargo-fuzz` uses compiler options that exist only in nightly Rust. Your normal code stays on stable; only the fuzzing commands use nightly.

```bash
cargo install --locked cargo-fuzz
cargo fuzz --version
```

```text
cargo-fuzz 0.13.2
```

Now create the library that we will attack. It reads a tiny binary format: one byte with the length of a name, then the name itself as UTF-8 text (UTF-8, Unicode Transformation Format 8-bit, is the standard way to store text as bytes, Lesson 03.6), then one byte `count`. For example `[3, b'a', b'b', b'c', 7]` means the name `"abc"` and the count `7`.

```bash
cd ~/projects/rust/code
cargo new --lib p09-fuzzing
cd p09-fuzzing
```

Put this in `src/lib.rs`:

```rust
/// One record of a tiny binary format:
/// byte 0 = length of the name, then the name (UTF-8), then one byte `count`.
#[derive(Debug, PartialEq, Eq)]
pub struct Record {
    pub name: String,
    pub count: u8,
}

#[derive(Debug, PartialEq, Eq)]
pub enum ParseError {
    Empty,
    BadUtf8,
}

pub fn parse_record(data: &[u8]) -> Result<Record, ParseError> {
    if data.is_empty() {
        return Err(ParseError::Empty);
    }
    let len = data[0] as usize;
    let name_bytes = &data[1..1 + len];
    let name = std::str::from_utf8(name_bytes).map_err(|_| ParseError::BadUtf8)?;
    let count = data[1 + len];
    Ok(Record {
        name: name.to_string(),
        count,
    })
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_a_valid_record() {
        let data = [3, b'a', b'b', b'c', 7];
        assert_eq!(
            parse_record(&data),
            Ok(Record { name: "abc".to_string(), count: 7 })
        );
    }

    #[test]
    fn empty_input_is_an_error() {
        assert_eq!(parse_record(&[]), Err(ParseError::Empty));
    }
}
```

**What happens:**

1. `parse_record` reads the length from byte 0, takes that many bytes as the name, converts them to text with `std::str::from_utf8` (which returns an error if the bytes are not valid UTF-8), and reads the byte after the name as `count`.
2. The function returns `Result`, so it promises to report problems as errors and never to panic. That promise is exactly what we want to test.
3. Both tests pass. A careful person would write a few more tests. Let us see what a fuzzer finds instead.

```bash
cargo test
```

```text
running 2 tests
test tests::empty_input_is_an_error ... ok
test tests::parses_a_valid_record ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Create the fuzz setup:

```bash
cargo fuzz init
```

This creates a new folder `fuzz/`. It is a separate Cargo package that depends on your library, with its own `Cargo.toml` and a folder `fuzz_targets/`. A `fuzz/.gitignore` file is created too, which tells Git to ignore the `target`, `corpus`, `artifacts` and `coverage` folders inside `fuzz/`.

```text
p09-fuzzing/
├── Cargo.toml
├── src/lib.rs
└── fuzz/
    ├── Cargo.toml              ← the fuzz package (depends on your crate)
    ├── .gitignore
    └── fuzz_targets/
        └── fuzz_target_1.rs    ← the fuzz target (the generated name)
```

## 3. The fuzz target

Open `fuzz/fuzz_targets/fuzz_target_1.rs` and replace its content:

```rust
#![no_main]

use libfuzzer_sys::fuzz_target;

fuzz_target!(|data: &[u8]| {
    let _ = p09_fuzzing::parse_record(data);
});
```

**What happens:**

1. `#![no_main]` tells Rust not to generate its own `main` function. `libFuzzer` brings its own `main`, which starts the fuzzing loop.
2. `fuzz_target!` is a macro from the `libfuzzer-sys` crate. The closure after it is called once per generated input. `data` is a slice of bytes: the input.
3. We call our function and ignore the result (`let _ =`). We do not care what it returns, only that it does not panic.
4. The crate name is `p09_fuzzing` in code: dashes become underscores.

If you run `cargo fuzz run fuzz_target_1` now with the normal stable compiler, you get an error (shortened):

```text
error: failed to run `rustc` to learn about target-specific information
  ...
  error: the option `Z` is only accepted on the nightly compiler
```

The fix is to select nightly for this one command with `+nightly`:

```bash
cargo +nightly fuzz run fuzz_target_1
```

```text
INFO: Seed: 398044694
INFO: Loaded 1 modules   (408 inline 8-bit counters): 408 [0x102805560, 0x1028056f8), 
INFO: A corpus is not provided, starting from an empty corpus

thread '<unnamed>' (4597100) panicked at src/lib.rs:20:27:
range end index 11 out of range for slice of length 1
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
==23035== ERROR: libFuzzer: deadly signal
...
Failing input:

	fuzz/artifacts/fuzz_target_1/crash-adc83b19e793491b1c6ea0fd8b46cd9f32e592fc

Output of `std::fmt::Debug`:

	[10]

Reproduce with:

	cargo fuzz run fuzz_target_1 fuzz/artifacts/fuzz_target_1/crash-adc83b19e793491b1c6ea0fd8b46cd9f32e592fc

Minimize test case with:

	cargo fuzz tmin fuzz_target_1 fuzz/artifacts/fuzz_target_1/crash-adc83b19e793491b1c6ea0fd8b46cd9f32e592fc
```

(The first build takes a while. Your seed, process numbers and file names will differ. The lines starting with `#` followed by a number are a long stack trace that we left out.)

**How to read it:**

1. The first lines are libFuzzer's start-up report: its random seed, how many code counters it found (`408` in our small library), and that there is no corpus yet.
2. The line `panicked at src/lib.rs:20:27` is the same panic message that you know from Lesson 02.9. The fuzzer found it within a fraction of a second.
3. `Failing input` is the **crash artifact**: a file holding the exact bytes. `Output of std::fmt::Debug` shows them as a Rust array: `[10]`. One byte, `10`.
4. It then prints two commands: one to reproduce the crash, one to minimize it.

The explanation of the bug: the first byte says "the name is 10 bytes long", but there are no more bytes. `&data[1..1 + len]` is `&data[1..11]` on a slice of length 1, so Rust panics. A function that returns `Result` just panicked on a one-byte input, and none of our two tests had thought of it.

> 💡 **Tip:** The most useful habit: the moment a fuzzer finds something, **reproduce it** with the printed command, so you are sure the crash is real and repeatable, then fix it.

## 4. Fix, run again, find the next bug

Add a new error case and use `get` instead of indexing. `slice.get(range)` returns `None` instead of panicking when the range is outside the slice, and `ok_or(error)` turns `None` into an `Err`:

```rust
#[derive(Debug, PartialEq, Eq)]
pub enum ParseError {
    Empty,
    BadUtf8,
    Truncated,
}
```

and in `parse_record` replace the line that makes `name_bytes`:

```rust
    let name_bytes = data.get(1..1 + len).ok_or(ParseError::Truncated)?;
```

Run the fuzzer again:

```bash
cargo +nightly fuzz run fuzz_target_1
```

```text
#2	INITED cov: 26 ft: 26 corp: 1/1b exec/s: 0 rss: 36Mb
#39	NEW    cov: 29 ft: 29 corp: 2/3b lim: 4 exec/s: 0 rss: 36Mb L: 2/2 MS: 2 CrossOver-CMP- DE: "\000\000"-

thread '<unnamed>' (4598175) panicked at src/lib.rs:23:17:
index out of bounds: the len is 1 but the index is 1
...
	[0]
```

**How to read it:** the lines starting with `#` are progress reports. `cov: 29` is the number of code points reached, `corp: 2/3b` means two inputs of three bytes in total are stored in the corpus, and `NEW` says that an input reached new code. The new crash is the input `[0]`: the name length is 0, the empty name is fine, but the byte after the name, `data[1 + len]`, does not exist. Fix this line too:

```rust
    let count = *data.get(1 + len).ok_or(ParseError::Truncated)?;
```

(`get` returns `Option<&u8>`, and `*` copies the byte out of the reference.) Run the fuzzer for ten seconds. Everything after `--` goes straight to libFuzzer; `-max_total_time=10` means "stop after 10 seconds":

```bash
cargo +nightly fuzz run fuzz_target_1 -- -max_total_time=10
```

```text
#2	INITED cov: 27 ft: 27 corp: 1/2b exec/s: 0 rss: 36Mb
#3	NEW    cov: 29 ft: 29 corp: 2/4b lim: 4 exec/s: 0 rss: 36Mb L: 2/2 MS: 1 ChangeBit-
#6	NEW    cov: 31 ft: 31 corp: 3/6b lim: 4 exec/s: 0 rss: 36Mb L: 2/2 MS: 3 ChangeBit-CopyPart-CopyPart-
...
#16513067	DONE   cov: 37 ft: 37 corp: 6/17b lim: 4096 exec/s: 1501187 rss: 611Mb
Done 16513067 runs in 11 second(s)
```

Sixteen million runs in eleven seconds, no crash. The coverage number stopped at `37`: the fuzzer has reached every branch of this small function, so more time would not help. That is a sign the target is "saturated". (Numbers like `exec/s` depend on your Mac.)

The corpus lives in `fuzz/corpus/fuzz_target_1/`: six small files that together reach all the code. Next time you run the target, the fuzzer starts from them instead of from nothing.

## 5. Turn every crash into a regression test

The `fuzz/artifacts` folder is ignored by Git, and the corpus too. So the crashes you fixed must live somewhere permanent: in normal tests. Add these to the `tests` module in `src/lib.rs`:

```rust
    #[test]
    fn crash_found_by_the_fuzzer_length_longer_than_data() {
        assert_eq!(parse_record(&[10]), Err(ParseError::Truncated));
    }

    #[test]
    fn crash_found_by_the_fuzzer_count_byte_missing() {
        assert_eq!(parse_record(&[0]), Err(ParseError::Truncated));
    }
```

`cargo test` now guards against both bugs on every run, in less than a second, without nightly and without a fuzzer.

> 🧪 **Testing:** The rule "every bug found becomes a test" (a *regression test*, Lesson 05.6) applies to fuzzer findings too. Name the test after what happened, so a reader knows where it came from. If the failing input is large, save the artifact file in a `tests/data/` folder and load it with `include_bytes!("data/crash-1.bin")`.

For a format this small you could even try *every* input up to two bytes in a normal test (Exercise 2). For longer inputs the number of possibilities explodes: three bytes already give 16.7 million inputs. That is where fuzzing takes over: it explores the huge space *smartly*, not completely.

## 6. Structured fuzzing and properties

Raw bytes are the right input for a parser. But sometimes you want to test code that takes a typed value, or you want to check a *rule* about the result. The crate `arbitrary` builds Rust values from the fuzzer's bytes.

Let us test the rule "**round trip**: if I write a record and read it back, I get the same record". First add the writer to `src/lib.rs`, with a small mistake that is easy to make:

```rust
/// Writes a record in the format that `parse_record` reads.
pub fn encode_record(record: &Record) -> Vec<u8> {
    let mut bytes = Vec::new();
    bytes.push(record.name.len() as u8);
    bytes.extend_from_slice(record.name.as_bytes());
    bytes.push(record.count);
    bytes
}
```

(`as u8` cuts a number down to its lowest 8 bits without any warning, Lesson 02.2. A name of 256 bytes would get length `0`.)

Add the `arbitrary` crate to the **fuzz** package (its `Cargo.toml` is `fuzz/Cargo.toml`), with the `derive` feature, which lets us write `#[derive(Arbitrary)]`, and register a second target at the end of that file:

```toml
[dependencies]
libfuzzer-sys = "0.4"
arbitrary = { version = "1", features = ["derive"] }

[[bin]]
name = "roundtrip"
path = "fuzz_targets/roundtrip.rs"
test = false
doc = false
bench = false
```

(Keep the other lines that `cargo fuzz init` generated. Only add what is shown. `cargo fuzz add roundtrip` creates the target file and this `[[bin]]` entry for you, and you then only need to add the `arbitrary` line.)

Create `fuzz/fuzz_targets/roundtrip.rs`:

```rust
#![no_main]

use arbitrary::Arbitrary;
use libfuzzer_sys::fuzz_target;
use p09_fuzzing::{Record, encode_record, parse_record};

#[derive(Debug, Arbitrary)]
struct Input {
    name: String,
    count: u8,
}

fuzz_target!(|input: Input| {
    let record = Record {
        name: input.name,
        count: input.count,
    };
    let bytes = encode_record(&record);
    assert_eq!(parse_record(&bytes), Ok(record));
});
```

**What happens:**

1. `#[derive(Arbitrary)]` generates code that builds an `Input` from the raw bytes libFuzzer provides: the first bytes become the `String`, the next one the `u8`. The fuzzer still works on bytes, but your target sees a typed value.
2. `fuzz_target!(|input: Input| ...)` accepts the typed value directly. The `Input` type must implement `Arbitrary`, and `Debug` (for the report).
3. The body builds a `Record`, encodes it, parses it back and asserts that the result equals the start. That one line is the whole **property**.

```bash
cargo +nightly fuzz run roundtrip -- -max_total_time=20
```

```text
#32114	NEW    cov: 96 ft: 96 corp: 10/662b lim: 317 exec/s: 0 rss: 44Mb L: 314/314 MS: 3 ShuffleBytes-CMP-InsertByte- DE: "\377\377"-
...
thread '<unnamed>' (4603635) panicked at fuzz_targets/roundtrip.rs:19:5:
assertion `left == right` failed
```

The fuzzer needed a name longer than 255 bytes, and it found one after tens of thousands of runs. It saved the input, but the raw bytes are hard to read. Let the fuzzer minimize it:

```bash
cargo +nightly fuzz tmin roundtrip fuzz/artifacts/roundtrip/crash-ef395ca4346fca40e3f833914fa81130824a2213
```

```text
Minimized artifact:

	fuzz/artifacts/roundtrip/minimized-from-b00c2ee4af8e0c729b8811209a33618e5a2b9a57

Output of `std::fmt::Debug`:

	Input {
	    name: "\0\0\0\u{7}\0\u{8}\u{8}\u{8}\u{8}\u{8}\u{8}\u{8}\u{8}\u{8}\u{8}\u{8}\u{8}...",
	    count: 255,
	}
```

(Yours will have a different file name, and a name of 256 or more bytes; we cut the long line.) The report shows a real `Input` value. It is the bug we planted: the name is too long for one length byte.

**The fix** belongs in the library, not in the fuzz target. A name that does not fit cannot be encoded, so say that in the type: return `None`. Replace `encode_record` with:

```rust
/// Writes a record in the format that `parse_record` reads.
/// Returns `None` when the name is longer than 255 bytes (the length must fit in one byte).
pub fn encode_record(record: &Record) -> Option<Vec<u8>> {
    let len = u8::try_from(record.name.len()).ok()?;
    let mut bytes = vec![len];
    bytes.extend_from_slice(record.name.as_bytes());
    bytes.push(record.count);
    Some(bytes)
}
```

**What happens:** `u8::try_from(usize)` returns `Err` if the number does not fit in a `u8`; `.ok()` turns the `Result` into an `Option`, and `?` returns `None` early. Now update the target: names that cannot be encoded are simply skipped (that is a rule of the library, not a bug):

```rust
fuzz_target!(|input: Input| {
    let record = Record {
        name: input.name,
        count: input.count,
    };
    let Some(bytes) = encode_record(&record) else {
        return; // name too long: nothing to round-trip
    };
    assert_eq!(parse_record(&bytes), Ok(record));
});
```

Add the regression test to the library, run `cargo test`, and run the fuzzer for 15 seconds:

```rust
    #[test]
    fn names_longer_than_255_bytes_cannot_be_encoded() {
        let record = Record { name: "x".repeat(256), count: 1 };
        assert_eq!(encode_record(&record), None);
    }
```

```bash
cargo test
cargo +nightly fuzz run roundtrip -- -max_total_time=15
```

```text
test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
...
#9045772	DONE   cov: 99 ft: 99 corp: 10/1313b lim: 4096 exec/s: 565360 rss: 527Mb
Done 9045772 runs in 16 second(s)
```

Nine million round trips, all fine.

## 7. What fuzzing can and cannot do

| It is good at | It is weak at |
|---|---|
| Panics, overflow (the fuzz build keeps debug assertions on), out-of-range indexes | Logic bugs where the code returns a *wrong but valid* answer, unless you write an assertion (a property) |
| Parsers, decoders, anything that takes bytes or text | Code that needs complicated setup (databases, networks) |
| Memory errors in `unsafe` code (AddressSanitizer is on) | Very long "magic" values and checksums that the fuzzer cannot guess |
| Hangs and huge memory use (libFuzzer has a time-out and a memory limit) | Programs that are slow per input: the fuzzer needs thousands of runs per second |

Practical rules:

1. **Keep a fuzz target fast and deterministic.** No sleeping, no reading the clock, no random numbers, no files, no network. The same bytes must always give the same result, otherwise a crash cannot be reproduced.
2. **Time-box it.** A fuzzer never "finishes". Run it for a fixed time: five minutes before you merge a change to a parser, longer on a schedule. `-max_total_time=SECONDS` does this. In Lesson 09.12 you will see how a CI (continuous integration: a server that checks every change automatically) job runs a short fuzz session.
3. **Fuzz in the background of your normal work.** Start it before lunch, read the result after.
4. **Commit the targets, not the corpus** (the default `.gitignore` is right). Commit the *regression tests* made from findings.
5. **Fuzzing and property tests are friends.** A property test (Lesson 09.4) runs a few hundred cases in `cargo test` on every build. Fuzzing runs for minutes or hours and learns from coverage. Use the same property in both if you like.

> ⚡ **Performance:** `cargo fuzz` builds in release mode with optimization and debug assertions together, so it is fast but still catches integer overflow. Each run of the fuzz target should take microseconds. If one input takes a second, fuzzing becomes useless.

> 📏 **Convention:** Name the fuzz targets after what they test (`parse_record`, `roundtrip`), keep them in `fuzz/fuzz_targets/`, and add a comment on top that states the property.

## Common mistakes

**1. Running `cargo fuzz` on the stable toolchain.** You get `error: the option Z is only accepted on the nightly compiler` (shown in Section 3). Use `cargo +nightly fuzz run …`.

**2. Forgetting `#![no_main]`.** The fuzz binary then has two `main` functions: yours and libFuzzer's. The build fails, or the fuzzer never starts. Always keep the first line `#![no_main]` in the target file.

**3. Fuzzing the wrong thing.** A target that calls `parse_record(data).unwrap()` crashes on every normal parse error and shows you nothing useful. Ignore expected errors (`let _ = …`) and crash only on real violations, such as a failed `assert_eq!`.

**4. Fixing the bug but not adding a test.** The artifact file is in an ignored folder. Without a regression test, the bug can come back unnoticed.

## Try it yourself

1. Build `code/p09-fuzzing` as in Sections 2 and 3, **before** any fix. Run the target and write down the crash input.
2. Reproduce the crash with the printed command, apply both fixes of Section 4, add the two regression tests and run `cargo test`.
3. Add `encode_record` and the `roundtrip` target. Let the fuzzer find the long name, minimize it, and fix the library.

## Exercises

### Exercise 1 — Read a crash file (easy)

The crash files in `fuzz/artifacts/fuzz_target_1/` are raw bytes, which you cannot read in an editor. Use one `cargo fuzz` subcommand to print the content of the artifact that holds the input `[0]` (from Section 4) in the readable Rust form.

<details>
<summary>Hint</summary>

The subcommand is called `fmt` (short for "format"). It takes the target name and the path of the artifact file.

</details>

<details>
<summary>Answer</summary>

```bash
cargo +nightly fuzz fmt fuzz_target_1 fuzz/artifacts/fuzz_target_1/crash-5ba93c9db0cff93f52b521d7420e43f6eda2784f
```

```text
Output of `std::fmt::Debug`:

[0]
```

Why it works: `fmt` runs the fuzz target's input decoder on the file and prints the result with `Debug`, the same text that the crash report prints. Use your own file name from the `ls fuzz/artifacts/fuzz_target_1` listing.

</details>

### Exercise 2 — Try every small input (medium)

Write a normal test in `tests/small_inputs.rs` that calls `parse_record` on **every** input of up to two bytes and checks only that it never panics. How many calls is that? Does the test need the nightly compiler?

<details>
<summary>Hint</summary>

Use two nested loops `for a in 0..=255u8` and `for b in 0..=255u8`. You need `use p09_fuzzing::parse_record;`. Do not forget the empty input and the inputs of one byte.

</details>

<details>
<summary>Answer</summary>

```rust
use p09_fuzzing::parse_record;

#[test]
fn never_panics_on_any_input_of_up_to_two_bytes() {
    let _ = parse_record(&[]);
    for a in 0..=255u8 {
        let _ = parse_record(&[a]);
        for b in 0..=255u8 {
            let _ = parse_record(&[a, b]);
        }
    }
}
```

```text
test never_panics_on_any_input_of_up_to_two_bytes ... ok
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: there are 1 + 256 + 65,536 = 65,793 calls. Each call that panics would fail the test, so passing means no panic. It is an ordinary test: no nightly, no fuzzer. It works only while the input space is tiny.

</details>

### Exercise 3 — A second property (hard)

Write a third fuzz target `reencode` with the property: "if `parse_record(data)` succeeds, then encoding the parsed record gives exactly the first bytes of `data`". Register it, run it for ten seconds, and say whether it finds a bug. What does the property tell you about trailing bytes?

<details>
<summary>Hint</summary>

Use the raw-bytes form `|data: &[u8]|`. The encoded record has length `bytes.len()`, so compare `bytes[..]` with `data[..bytes.len()]`. Add a `[[bin]]` entry in `fuzz/Cargo.toml`.

</details>

<details>
<summary>Answer</summary>

`fuzz/fuzz_targets/reencode.rs`:

```rust
#![no_main]

use libfuzzer_sys::fuzz_target;
use p09_fuzzing::{encode_record, parse_record};

fuzz_target!(|data: &[u8]| {
    if let Ok(record) = parse_record(data) {
        let bytes = encode_record(&record).expect("a parsed name is at most 255 bytes");
        assert_eq!(bytes[..], data[..bytes.len()]);
    }
});
```

In `fuzz/Cargo.toml`:

```toml
[[bin]]
name = "reencode"
path = "fuzz_targets/reencode.rs"
test = false
doc = false
bench = false
```

```bash
cargo +nightly fuzz run reencode -- -max_total_time=10
```

```text
#13250496	DONE   cov: 58 ft: 58 corp: 6/17b lim: 4096 exec/s: 1204590 rss: 686Mb
Done 13250496 runs in 11 second(s)
```

Why it works: no crash, so the property holds. Notice that we compare only with the *beginning* of `data`: `parse_record` ignores bytes after the record, so the input may be longer than the encoded record. If your parser should reject trailing bytes, this property would show you that it does not.

</details>

## Quiz

1. Why does the fuzzer keep an input that reached a new line of code?

<details>
<summary>Answer</summary>

Because that input unlocked a new branch (for example a correct first byte). Mutating it produces inputs that go deeper into the code. This is what "coverage-guided" means, and it is why fuzzing beats purely random bytes.

</details>

2. Why does `cargo fuzz` need `+nightly`?

<details>
<summary>Answer</summary>

It uses compiler options (the sanitizer and coverage instrumentation, passed with `-Z`) that exist only in the nightly compiler. Your own code can stay on stable.

</details>

3. A fuzz target runs `parse_record(data).unwrap()`. What is the problem?

<details>
<summary>Answer</summary>

Every ordinary parse error (`Err`) becomes a panic, so the fuzzer reports "crashes" that are correct behaviour. Ignore the `Result` and assert only real rules.

</details>

4. After the fuzzer finds and you fix a bug, what must you add to the project?

<details>
<summary>Answer</summary>

A regression test in the normal test suite with the failing input, because the artifacts and the corpus are not committed.

</details>

5. Can a fuzzer find a function that returns the wrong answer without crashing?

<details>
<summary>Answer</summary>

Only if you tell it what "right" means with an assertion in the fuzz target, such as the round-trip property in Section 6. Without an assertion it only finds crashes, hangs and memory errors.

</details>

## Summary

- Fuzzing runs your code on huge numbers of generated inputs and reports the first one that makes it crash. Coverage guidance makes the search smart.
- `cargo-fuzz` needs the nightly compiler: `cargo +nightly fuzz run <target>`. A target is a small function, `fuzz_target!(|data: &[u8]| { … })`, in a file with `#![no_main]`.
- The report names a crash artifact with the exact input; reproduce it, minimize it with `cargo fuzz tmin`, fix the code, and add a regular regression test.
- With the `arbitrary` crate the target can take a typed value and check a property, like encode-then-decode giving back the original.
- Targets must be fast and deterministic. Run them time-boxed with `-- -max_total_time=SECONDS`.
- Commit the targets and the regression tests; the corpus and artifacts are ignored by default.

## Sources and further reading

- [The Rust Fuzz Book](https://rust-fuzz.github.io/book/) — the official guide to `cargo-fuzz`, including the `arbitrary` crate and the other fuzzers.
- [cargo-fuzz on GitHub](https://github.com/rust-fuzz/cargo-fuzz) — installation, options and the list of subcommands (`run`, `tmin`, `cmin`, `fmt`, `coverage`).
- [libFuzzer documentation](https://llvm.org/docs/LibFuzzer.html) — every option you can pass after `--` and the meaning of the output lines (`cov`, `ft`, `corp`).
- [The `arbitrary` crate](https://docs.rs/arbitrary/latest/arbitrary/) — how to generate typed values from bytes and how to implement `Arbitrary` by hand.

---

[🏠 Start](../START-HERE.md) · [← 09.7 Code coverage and mutation testing](07-code-coverage-and-mutation-testing.md) · [📋 Phase 09](README.md) · [09.9 Debugging in depth: LLDB and RustRover →](09-debugging-with-lldb-and-rustrover.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
