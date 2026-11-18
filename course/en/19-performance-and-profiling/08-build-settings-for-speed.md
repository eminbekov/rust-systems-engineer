<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 19.7 SIMD: one instruction, many data](07-simd.md) · [📋 Phase 19](README.md) · [19.9 Performance of async and I/O code →](09-async-and-io-performance.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/19-performance-and-profiling/08-build-settings-for-speed.md) <!-- nav -->

# Lesson 19.8 — Build settings for speed: profiles, LTO and PGO

> **Phase 19 — Performance and profiling** · Lesson 8 of 11 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a Cargo profile is, what the `dev` and `release` profiles contain, and how to make your own profiles.
- What codegen units, LTO (link-time optimization) and `panic = "abort"` do, and how they change speed, binary size and build time.
- How to compile for the exact CPU (Central Processing Unit) you run on (`target-cpu`), and why that can make a program crash on other computers.
- How to use PGO (profile-guided optimization) with `cargo pgo`.
- How to judge honestly whether a setting helped.

## Before you start

- You finished [Lesson 19.7 — SIMD: one instruction, many data](07-simd.md).
- You can use `hyperfine` and read `mean ± σ` ([Lesson 09.11 — Benchmarking in depth](../09-testing-debugging-benchmarking/11-benchmarking-in-depth.md)).
- You know what the compiler and the linker do ([Lesson 12.8 — From Rust to machine code](../12-computer-systems/08-from-rust-to-machine-code.md), [Lesson 12.12 — Linking and loading programs](../12-computer-systems/12-linking-and-loading.md)) and what a panic and unwinding are ([Lesson 04.10 — Error handling](../04-structs-enums-collections-errors/10-error-handling.md)).

## New words

| Word | Meaning |
|---|---|
| **profile** | A named set of compiler settings in `Cargo.toml`. `cargo build` uses the `dev` profile, `cargo build --release` the `release` profile. |
| **optimization level** (`opt-level`) | How hard the compiler works to make the code fast: `0` (not at all) to `3` (most), or `"s"` / `"z"` (small code). |
| **LLVM** (originally Low Level Virtual Machine; today just a name) | The code generator inside `rustc`. It optimizes the program and turns it into machine code. |
| **codegen unit** | A piece of one crate that LLVM optimizes on its own, in parallel with other pieces. More pieces build faster; fewer pieces let LLVM see more code at once. |
| **inlining** | Copying the body of a called function into the caller, so the call disappears and the code can be optimized together. |
| **LTO** (Link-Time Optimization) | Optimizing the whole program again when all crates are put together, so the optimizer can inline and remove code *across* crates. |
| **fat / thin LTO** | Fat LTO merges all code into one big unit and optimizes it (slow to build). Thin LTO optimizes in parallel using a summary of the whole program (faster to build, often similar results). |
| **panic strategy** | What happens on a panic: `unwind` (walk back up the stack and run `Drop` code) or `abort` (end the process at once). |
| **symbol** | A name stored in the binary for a function or variable, used by debuggers, profilers and crash reports. *Stripping* removes them. |
| **target CPU** | The CPU model the compiler assumes. It decides which optional instructions (target features, Lesson 19.7) the code may use. |
| **PGO** (Profile-Guided Optimization) | Building the program twice: first with counters, then you run it on typical work, then the compiler uses the counts to optimize the hot paths. |
| **instrumented binary** | A special build that counts how often each branch and function runs and writes the counts to a file when it exits. |
| **BOLT** (Binary Optimization and Layout Tool) | An LLVM tool that reorders the machine code of a finished binary using a profile. `cargo pgo` can use it on Linux; we do not use it here. |

## 1. Profiles: `dev` and `release`

When you run `cargo build`, Cargo passes many options to `rustc`. A **profile** is the named set of those options. You have used two of them since Phase 02:

| Setting | `dev` (`cargo build`) | `release` (`cargo build --release`) | What it controls |
|---|---|---|---|
| `opt-level` | `0` | `3` | how much the code is optimized |
| `debug` | `true` | `false` | debug information for debuggers |
| `debug-assertions` | `true` | `false` | `debug_assert!` checks |
| `overflow-checks` | `true` | `false` | panic on integer overflow |
| `incremental` | `true` | `false` | reuse work from the last build |
| `codegen-units` | `256` | `16` | how many pieces LLVM works on in parallel |
| `lto` | `false` | `false` | link-time optimization (`false` means only inside one crate) |
| `panic` | `"unwind"` | `"unwind"` | the panic strategy |

The `dev` profile is made for a fast edit–compile–run cycle; `release` is made for fast programs. You can change any setting in `Cargo.toml`, and you can make new profiles.

Our test program is a small calculator that parses and evaluates 50,000 random expressions like `(712 + 33) * 9 / 4`, 60 times. Create it:

```bash
cd ~/projects/rust
cargo new code/p19-build
cd code/p19-build
```

`src/lib.rs`, the calculator (a recursive-descent parser: one function per grammar rule):

```rust
//! A tiny calculator: parses and evaluates expressions like `(3 + 4) * 2 - 7 / 3`.

/// Evaluates one expression with `+ - * /`, parentheses and whole numbers.
/// Returns `None` for bad input or division by zero.
pub fn eval(text: &str) -> Option<i64> {
    let mut p = Parser {
        bytes: text.as_bytes(),
        pos: 0,
    };
    let value = p.expr()?;
    p.skip_spaces();
    if p.pos == p.bytes.len() {
        Some(value)
    } else {
        None
    }
}

struct Parser<'a> {
    bytes: &'a [u8],
    pos: usize,
}

impl Parser<'_> {
    fn skip_spaces(&mut self) {
        while self.pos < self.bytes.len() && self.bytes[self.pos] == b' ' {
            self.pos += 1;
        }
    }

    fn peek(&mut self) -> Option<u8> {
        self.skip_spaces();
        self.bytes.get(self.pos).copied()
    }

    // expr = term (('+' | '-') term)*
    fn expr(&mut self) -> Option<i64> {
        let mut value = self.term()?;
        while let Some(op @ (b'+' | b'-')) = self.peek() {
            self.pos += 1;
            let rhs = self.term()?;
            value = if op == b'+' {
                value.wrapping_add(rhs)
            } else {
                value.wrapping_sub(rhs)
            };
        }
        Some(value)
    }

    // term = factor (('*' | '/') factor)*
    fn term(&mut self) -> Option<i64> {
        let mut value = self.factor()?;
        while let Some(op @ (b'*' | b'/')) = self.peek() {
            self.pos += 1;
            let rhs = self.factor()?;
            value = if op == b'*' {
                value.wrapping_mul(rhs)
            } else {
                value.checked_div(rhs)?
            };
        }
        Some(value)
    }

    // factor = number | '(' expr ')'
    fn factor(&mut self) -> Option<i64> {
        match self.peek()? {
            b'(' => {
                self.pos += 1;
                let value = self.expr()?;
                if self.peek()? != b')' {
                    return None;
                }
                self.pos += 1;
                Some(value)
            }
            b'0'..=b'9' => {
                let mut value: i64 = 0;
                while let Some(&d @ b'0'..=b'9') = self.bytes.get(self.pos) {
                    value = value.wrapping_mul(10).wrapping_add(i64::from(d - b'0'));
                    self.pos += 1;
                }
                Some(value)
            }
            _ => None,
        }
    }
}

#[cfg(test)]
mod tests {
    use super::eval;

    #[test]
    fn follows_the_usual_rules() {
        assert_eq!(eval("1 + 2 * 3"), Some(7));
        assert_eq!(eval("(1 + 2) * 3"), Some(9));
        assert_eq!(eval("7 / 0"), None);
        assert_eq!(eval("2 +"), None);
    }
}
```

`src/main.rs`, which makes the input and runs the calculator:

```rust
use p19_build::eval;

/// A small, fast pseudo-random number generator (xorshift), so every run sees the same input.
struct Rng(u64);

impl Rng {
    fn next(&mut self) -> u64 {
        self.0 ^= self.0 << 13;
        self.0 ^= self.0 >> 7;
        self.0 ^= self.0 << 17;
        self.0
    }
}

/// Writes a random expression with about `depth` levels of nesting into `out`.
fn gen_expr(rng: &mut Rng, depth: u32, out: &mut String) {
    if depth == 0 || rng.next().is_multiple_of(3) {
        out.push_str(&(rng.next() % 1000).to_string());
        return;
    }
    let paren = rng.next().is_multiple_of(4);
    if paren {
        out.push('(');
    }
    gen_expr(rng, depth - 1, out);
    let op = [" + ", " - ", " * ", " / "][(rng.next() % 4) as usize];
    out.push_str(op);
    gen_expr(rng, depth - 1, out);
    if paren {
        out.push(')');
    }
}

fn main() {
    // Optional first argument: how deeply the expressions are nested (default 6).
    let depth: u32 = std::env::args()
        .nth(1)
        .and_then(|s| s.parse().ok())
        .unwrap_or(6);
    let mut rng = Rng(42);
    let mut lines = Vec::new();
    for _ in 0..50_000 {
        let mut line = String::new();
        gen_expr(&mut rng, depth, &mut line);
        lines.push(line);
    }

    let mut ok = 0u64;
    let mut failed = 0u64;
    let mut checksum = 0i64;
    for _ in 0..60 {
        for line in &lines {
            match eval(line) {
                Some(v) => {
                    ok += 1;
                    checksum = checksum.wrapping_add(v);
                }
                None => failed += 1,
            }
        }
    }
    println!("ok {ok}, failed {failed}, checksum {checksum}");
}
```

**What happens:**

1. The package has two crates: the library `p19_build` (`src/lib.rs`) and the program (`src/main.rs`) that uses it. That matters for LTO in Section 2.
2. `Rng` is a tiny **xorshift** random number generator: three shifts and XORs (exclusive-or operations, Lesson 12.4) give a new, random-looking number. It always starts from 42, so every run makes exactly the same expressions.
3. `gen_expr` builds a random expression tree up to `depth` levels deep. About 1 in 25 expressions contains a division by zero, and `eval` returns `None` for it.
4. The `checksum` adds all results. If a build setting ever changed the program's behaviour, the checksum would show it.

Run the test, then build and compare the two profiles:

```bash
cargo test
cargo build
cargo build --release
./target/release/p19-build
hyperfine -N --warmup 1 --runs 5 -n dev ./target/debug/p19-build -n release ./target/release/p19-build
```

```text
ok 2879220, failed 120780, checksum -6775572847266405736
Benchmark 1: dev (5 runs)
                mean     ±       σ          min     …     max
  Wall Time    2.344 s   ±   0.151 s      2.194 s   …   2.575 s
  ...
Benchmark 2: release (5 runs)
                mean     ±       σ          min     …     max
  Wall Time    0.482 s   ±   0.056 s      0.431 s   …   0.549 s            -79.4%  (4.9x faster)
  ...
```

`-N` starts each command directly, without a shell in between, which removes a little noise; `-n` gives each command a short name in the report. The release build is about 5 times faster. **This is by far the biggest build setting**: never measure or ship a `dev` build. Everything in the rest of this lesson changes a few percent at most, as you will see.

## 2. Codegen units and LTO

LLVM can only optimize code it can **see** at the same time. Two settings decide how much it sees.

**Codegen units.** To build faster, `rustc` cuts each crate into pieces (16 in `release`) and lets LLVM optimize them in parallel on several CPU cores. A function in one piece cannot be inlined into a function in another piece. `codegen-units = 1` gives LLVM the whole crate at once: a slower build, sometimes faster code.

**LTO.** Normally each crate is optimized alone, and the linker only glues the finished machine code together. With **LTO**, the compiler keeps an LLVM form of every crate (including `std` and all your dependencies) and optimizes the program once more as a whole, at link time. Then a function from a dependency can be inlined into your code, and code that nobody uses is removed.

```text
without LTO:   crate A ──optimize──▶ machine code ─┐
               crate B ──optimize──▶ machine code ─┴─ link ──▶ program

with LTO:      crate A ──┐
               crate B ──┴─ link + optimize everything together ──▶ program
```

Make three new profiles at the end of `Cargo.toml`. `inherits = "release"` means "start from all release settings and change only these":

```toml
[profile.cu1]
inherits = "release"
codegen-units = 1

[profile.thin]
inherits = "release"
lto = "thin"

[profile.fat]
inherits = "release"
lto = "fat"
codegen-units = 1
```

Build them with `--profile`. Each profile gets its own folder under `target/` with the same name:

```bash
cargo build --profile cu1
cargo build --profile thin
cargo build --profile fat
ls -l target/*/p19-build | awk '{print $5, $9}'
```

```text
455376 target/cu1/p19-build
544520 target/debug/p19-build
378752 target/fat/p19-build
455392 target/release/p19-build
430816 target/thin/p19-build
```

**What happens:**

1. `awk '{print $5, $9}'` prints only column 5 (the size in bytes) and column 9 (the file name) of `ls -l`.
2. Fat LTO makes the binary about 17% smaller: it removed code from `std` that our program never calls. Thin LTO removes less.
3. One codegen unit alone changes nothing here: our library is so small that it already fits into one piece.

We measure the speed of all variants together in Section 5. One fact explains why the gains are small for this program: since Rust 1.75, `rustc` already marks **small** functions so that they can be inlined into other crates, even without LTO. LTO helps more when a program spends its time in *larger* functions from other crates (for example a JSON (JavaScript Object Notation) parser or a hash map from a dependency).

> ⚡ **Performance:** fat LTO and `codegen-units = 1` make the **build** slower, because LLVM optimizes everything in one piece on one core. On this tiny package the difference is one or two seconds; on a service with hundreds of dependencies a fat-LTO release build can take several times longer. That is why `release` does not use them by default.

## 3. Panic strategy and stripping: a profile for shipping

Two more settings mostly change **size**, not speed:

- `panic = "abort"`: a panic ends the process at once. The code for unwinding the stack (running `Drop` for every value on the way up) is not needed and is removed.
- `strip = true`: removes the **symbols**, the function names stored in the binary.

Both have a price. With `panic = "abort"`, a panic in any thread kills the whole process, and code that catches panics (`std::panic::catch_unwind`; Tokio, for example, turns a panic inside a task into an error for whoever awaits it) no longer works; `Drop` code, such as flushing a file buffer, does not run. Without symbols, profilers and crash reports show addresses instead of function names.

A good habit is a separate profile for the builds you ship, so that daily `--release` builds stay quick. Add it to `Cargo.toml`:

```toml
[profile.dist]
inherits = "release"
lto = "fat"
codegen-units = 1
panic = "abort"
strip = true
```

Build it with `-v` (verbose) to see which `rustc` options the profile turned into:

```bash
cargo build --profile dist -v
ls -l target/release/p19-build target/dist/p19-build | awk '{print $5, $9}'
```

The `rustc` line for the program contains (shortened):

```text
-C opt-level=3 -C panic=abort -C lto=fat -C codegen-units=1 ... -C strip=symbols
```

```text
302576 target/dist/p19-build
455392 target/release/p19-build
```

The `dist` binary is a third smaller than `release`. A profile is nothing more than a list of `-C` options for `rustc`.

> 📏 **Convention:** profiles are read only from the `Cargo.toml` at the root of the workspace (the package or workspace you build). Profiles written in a dependency's `Cargo.toml` are ignored. Keep `release` close to the default, and put slow-to-build settings into an extra profile such as `dist`.

## 4. Building for your exact CPU

The compiler must produce code that runs on *every* CPU of the target. For `aarch64-apple-darwin` it assumes the CPU of the first Apple M1, `apple-m1`, so newer instructions of your M4 are not used. `-C target-cpu=native` tells `rustc` "use everything the CPU of *this* computer has":

```bash
rustc --print target-cpus | grep native
```

```text
    native          - Select the CPU of the current host (currently apple-m4).
```

Compiler options that are not part of a profile go into the environment variable `RUSTFLAGS`. A different `RUSTFLAGS` changes every crate, so use a separate target folder to keep the normal builds:

```bash
RUSTFLAGS="-C target-cpu=native" cargo build --profile dist --target-dir target/native
```

To use it for one project every time, put it into `.cargo/config.toml` in the project folder instead:

```toml
[build]
rustflags = ["-C", "target-cpu=native"]
```

> ⚠️ **Common mistake:** shipping a `target-cpu=native` binary. It may use instructions that other CPUs do not have. On a server with an older CPU it crashes at once with "illegal instruction" (`SIGILL`, Lesson 13.7). Use `native` only for programs that run on the machine where they were built, or name a specific older CPU level (for x86-64 servers, for example `-C target-cpu=x86-64-v3`, which requires AVX2, as in Lesson 19.7).

## 5. PGO: let a real run guide the optimizer

The optimizer must guess many things: which side of an `if` is usual, which function is hot and worth inlining, which code is cold and can be moved out of the way. **PGO** replaces the guesses with facts from a real run:

```text
1. build with counters      ──▶  instrumented binary
2. run it on typical work   ──▶  counts written to *.profraw files
3. merge the counts         ──▶  one *.profdata file
4. build again using it     ──▶  optimized binary
```

The tool `cargo-pgo` does steps 1, 3 and 4 for you. It needs `llvm-profdata`, which comes with the `llvm-tools-preview` component. Install both once:

```bash
rustup component add llvm-tools-preview
cargo install --locked cargo-pgo
cargo pgo info
```

```text
[rustc version]: 1.100.0 is recent enough
[llvm-profdata]: found at /Users/you/.rustup/toolchains/stable-aarch64-apple-darwin/lib/rustlib/aarch64-apple-darwin/bin/llvm-profdata
[llvm-bolt]: could not be found (Build LLVM with BOLT and add its `bin` directory to PATH.)
[merge-fdata]: could not be found (Build LLVM with BOLT and add its `bin` directory to PATH.)
Some requirements were not satisfied
```

The missing BOLT tools are only needed for the optional BOLT step; PGO works without them. Now the four steps:

```bash
cargo pgo build
./target/aarch64-apple-darwin/release/p19-build
cargo pgo optimize
```

Output (shortened; timestamps and long paths removed):

```text
INFO  cargo_pgo::pgo::instrument] PGO-instrumented binary p19-build built successfully.
INFO  cargo_pgo::pgo::instrument] Now run .../target/aarch64-apple-darwin/release/p19-build on your workload.
ok 2879220, failed 120780, checksum -6775572847266405736
INFO  cargo_pgo::pgo::optimize] Found 1 PGO profile file with total size 2.59 KiB at .../target/pgo-profiles.
INFO  cargo_pgo::pgo::optimize] Merged PGO profile(s) to .../target/pgo-profiles/merged-59625e....profdata.
INFO  cargo_pgo::pgo::optimize] PGO-optimized binary p19-build built successfully.
PGO optimized build successfully finished.
WARN  cargo_pgo::pgo::optimize] PGO profile data was not found for 1 function.
```

**What happens:**

1. `cargo pgo build` builds the instrumented binary. It passes `--target aarch64-apple-darwin` to Cargo, so the binary lands in `target/aarch64-apple-darwin/release/`.
2. Running it is **your** job: this is where you give it typical work. When it exits, it writes a `.profraw` file into `target/pgo-profiles/`.
3. `cargo pgo optimize` merges the files with `llvm-profdata` and builds again with `-C profile-use=...`. The final binary replaces the instrumented one in the same folder.
4. The warning is harmless: one function never ran during the training run, so there are no counts for it.

Now compare everything (this takes about a minute):

```bash
hyperfine -N --warmup 3 --runs 20 -n release ./target/release/p19-build -n cu1 ./target/cu1/p19-build -n thin ./target/thin/p19-build -n fat ./target/fat/p19-build -n dist ./target/dist/p19-build -n native ./target/native/dist/p19-build -n pgo ./target/aarch64-apple-darwin/release/p19-build --export-markdown results.md
```

The table from `results.md` (M4; yours will differ):

```text
| Command | Mean Wall Time [ms] | Change | Factor |
|:---|---:|---:|:---|
| `release` | 465.3 ± 33.1 |  |  |
| `cu1` | 460.2 ± 18.1 | -1.1% |  |
| `thin` | 464.2 ± 48.9 | -0.3% |  |
| `fat` | 447.0 ± 18.9 | -3.9% |  |
| `dist` | 450.3 ± 28.6 | -3.2% |  |
| `native` | 440.8 ± 9.7 | -5.3% |  |
| `pgo` | 436.0 ± 19.3 | -6.3% |  |
```

**What happens:**

1. Every difference is between 0 and 6 percent. PGO and `native` look best.
2. Now read the `±` column. The noise `σ` of `release` is 33 ms, which is 7 percent. A difference smaller than the noise is **not** a result yet. In other runs on the same Mac, PGO was anything from 4 percent faster to 1 percent slower.
3. To decide, repeat the measurement (more runs, a quiet machine, Lesson 09.11), and keep a setting only if the gain is clearly larger than the noise and worth its cost: slower builds for LTO, a training step in every release for PGO, less portable binaries for `native`.

Compare this with Section 1: changing `dev` to `release` gave 5 times. In [Lesson 19.10](10-project-make-a-slow-program-fast.md) you will make a program 10 times faster by changing the code. Build settings are the last few percent, not the first big step.

> 🧪 **Testing:** settings like `panic`, `overflow-checks` and `debug-assertions` change how a program **behaves**, not only how fast it is. Run your tests with the profile you ship: `cargo test --profile dist`.

## 6. A sensible setup

| Goal | Setting |
|---|---|
| Daily work | `dev`; if the program is too slow to try out, add `opt-level = 1` to `[profile.dev]` (our calculator: 2.3 s → 0.44 s, almost as fast as `release`) |
| Benchmarks and profiling | `release` for benchmarks; for profilers the `profiling` profile from Lesson 19.3 (`release` plus `debug = true`). `debug = "line-tables-only"` is a smaller alternative that still gives function names and lines |
| Shipping | an extra profile such as `dist`: `lto = "fat"` or `"thin"`, `codegen-units = 1`, and maybe `panic = "abort"` |
| Smallest binary | `opt-level = "z"` with `strip = true` (usually slower; see Exercise 1) |
| The last few percent | PGO with a realistic training workload, measured carefully |

## Common mistakes

> ⚠️ **Common mistake:** forgetting `--profile`. `cargo build --release` does not use your `dist` settings. Check that the `Finished` line names the `dist` profile and that the binary is in `target/dist/`.

> ⚠️ **Common mistake:** training PGO on unusual work, for example only on tiny test inputs. The optimizer then speeds up the wrong paths. Use input that looks like real use.

> ⚠️ **Common mistake:** calling a 2 percent difference "faster" when `σ` is 5 percent. Always compare the difference with the noise.

> ⚠️ **Common mistake:** setting `panic = "abort"` in a service that relies on panics being caught (for example a panicking Tokio task that should not stop the whole server).

## Try it yourself

1. Build `code/p19-build` and reproduce the `dev` vs `release` comparison.
2. Add the `cu1`, `thin`, `fat` and `dist` profiles, build them and write down the binary sizes.
3. Do the PGO steps and run the big `hyperfine` command twice. Do the numbers agree between the two runs?
4. Add `opt-level = 1` to a `[profile.dev]` section, run `cargo build` and time `./target/debug/p19-build`.

## Exercises

### Exercise 1 — The smallest binary (easy)

Make a profile `small` that inherits from `dist` and sets `opt-level = "z"`. Compare its size and speed with `dist`. Is it worth it for this program?

<details>
<summary>Hint</summary>

`[profile.small]` with `inherits = "dist"`. Then `cargo build --profile small`, `ls -l` and a `hyperfine` with both binaries.

</details>

<details>
<summary>Answer</summary>

```toml
[profile.small]
inherits = "dist"
opt-level = "z"
```

```text
302464 target/dist/p19-build
286080 target/small/p19-build
```

```text
Benchmark 1: dist (10 runs)
  Wall Time    443.2 ms  ±    31.2 ms     427.4 ms  …   528.5 ms
Benchmark 2: small (10 runs)
  Wall Time    632.2 ms  ±    41.5 ms     593.6 ms  …   714.1 ms           +42.7%  (1.4x slower)
```

Why it works: `"z"` tells LLVM to prefer small code over fast code (for example, no loop unrolling, less inlining). The binary is only 5% smaller, because most of it is `std` code that was already removed or shrunk by LTO and stripping, but the program is 1.4 times slower. For a program on a normal computer, not worth it; for a tiny device with little storage it can be.

</details>

### Exercise 2 — A test that depends on the profile (medium)

Make a profile `safe` that inherits from `release` and sets `overflow-checks = true`. Write an integration test `tests/overflow.rs` that checks that adding `250u8 + 10` panics with "attempt to add with overflow". Run it with `cargo test --profile safe` and with `cargo test --release`. What happens, and why?

<details>
<summary>Hint</summary>

Use `#[should_panic(expected = "...")]`. Hide the numbers with `std::hint::black_box`, otherwise the compiler sees the overflow at compile time and refuses to build (the `arithmetic_overflow` lint).

</details>

<details>
<summary>Answer</summary>

```toml
[profile.safe]
inherits = "release"
overflow-checks = true
```

```rust
use std::hint::black_box;

#[test]
#[should_panic(expected = "attempt to add with overflow")]
fn adding_past_u8_max_panics() {
    let big: u8 = black_box(250);
    let _ = big + black_box(10);
}
```

`cargo test --profile safe --test overflow`:

```text
test adding_past_u8_max_panics - should panic ... ok
```

`cargo test --release --test overflow`:

```text
test adding_past_u8_max_panics - should panic ... FAILED
note: test did not panic as expected at tests/overflow.rs:5:4
```

Why it works: in `release`, `overflow-checks` is `false`, so `250 + 10` silently wraps around to 4 and nothing panics. The same code behaves differently in different profiles. Some teams ship with `overflow-checks = true` because a panic is safer than a wrong number (Lesson 12.2); the cost is usually small, but measure it.

</details>

### Exercise 3 — PGO with the wrong training (medium)

The program takes the nesting depth as its first argument. Train PGO with very simple expressions (`./target/aarch64-apple-darwin/release/p19-build 1` instead of the normal run), copy the result to `pgo-bad`, then train again normally and copy the result to `pgo-good`. Compare `release`, `pgo-good` and `pgo-bad` on the normal workload. What do you see, and what would you expect on a bigger program?

<details>
<summary>Hint</summary>

`cargo pgo build`, run with `1`, `cargo pgo optimize`, `cp target/aarch64-apple-darwin/release/p19-build pgo-bad`. Then the same without the `1`, into `pgo-good`. `cargo pgo build` clears the old profiles itself.

</details>

<details>
<summary>Answer</summary>

```bash
cargo pgo build
./target/aarch64-apple-darwin/release/p19-build 1
cargo pgo optimize
cp target/aarch64-apple-darwin/release/p19-build pgo-bad
cargo pgo build
./target/aarch64-apple-darwin/release/p19-build
cargo pgo optimize
cp target/aarch64-apple-darwin/release/p19-build pgo-good
hyperfine -N --warmup 3 --runs 20 -n release ./target/release/p19-build -n pgo-good ./pgo-good -n pgo-bad ./pgo-bad
```

One run on an M4:

```text
Benchmark 1: release (20 runs)
  Wall Time    426.7 ms  ±     7.6 ms     414.6 ms  …   444.0 ms
Benchmark 2: pgo-good (20 runs)
  Wall Time    430.8 ms  ±     8.6 ms     418.1 ms  …   457.9 ms            +1.0%
Benchmark 3: pgo-bad (20 runs)
  Wall Time    431.7 ms  ±    14.3 ms     421.1 ms  …   470.4 ms            +1.2%
```

Why it works: for this small parser all three are equal within the noise; there is little for PGO to improve. That is a valid result. In a large program with many branches (a compiler, a database, an interpreter), a profile from simple inputs marks the wrong paths as hot, and the PGO build can even become slower than plain `release`. Train on input that looks like real use, and measure on it.

</details>

## Quiz

1. Which build setting changed the speed of the calculator the most?

<details>
<summary>Answer</summary>

`dev` → `release` (in fact `opt-level` 0 → 3): about 5 times faster. All other settings changed it by a few percent at most.

</details>

2. What is the difference between thin and fat LTO?

<details>
<summary>Answer</summary>

Both optimize across crates at link time. Fat LTO merges all code into one unit (best view, slowest build). Thin LTO keeps the units separate, shares a summary of the whole program and works in parallel (faster build, often similar results).

</details>

3. Name two risks of `panic = "abort"`.

<details>
<summary>Answer</summary>

Any panic ends the whole process at once, so catching panics (`catch_unwind`, a panicking Tokio task) no longer works; and `Drop` code does not run, so buffers are not flushed and cleanup does not happen.

</details>

4. Why can a binary built with `-C target-cpu=native` crash on another computer?

<details>
<summary>Answer</summary>

It may use instructions that only the build machine's CPU has. On a CPU without them, the program stops with "illegal instruction".

</details>

5. What are the steps of PGO?

<details>
<summary>Answer</summary>

Build an instrumented binary, run it on typical work so it writes counts (`.profraw`), merge the counts (`.profdata`), and build again with the profile. `cargo pgo build`, your run, `cargo pgo optimize`.

</details>

6. A setting makes the program 2% faster, and `σ` is 4%. What do you conclude?

<details>
<summary>Answer</summary>

Nothing yet: the difference is smaller than the noise. Measure again with more runs on a quiet machine; keep the setting only if the gain is clearly larger than the noise and worth its cost.

</details>

## Summary

- A profile is a named list of `rustc` options in `Cargo.toml`. Make your own with `[profile.name]` and `inherits = "release"`, build with `--profile name`; the output goes to `target/name/`.
- `release` instead of `dev` is the big step (5 times here). The other settings are the last few percent.
- `codegen-units = 1` and LTO let LLVM see more code at once. They often make binaries smaller and sometimes faster, and they always make builds slower.
- `panic = "abort"` and `strip = true` mainly reduce size, but change behaviour on panic and remove names from crash reports and profiles.
- `-C target-cpu=native` uses all instructions of your CPU; such a binary may crash on other computers.
- PGO (`cargo pgo build` → run → `cargo pgo optimize`) uses counts from a real run. Train on realistic work.
- Compare every gain with the noise `σ`, and test with the profile you ship.

## Sources and further reading

- [The Cargo Book — Profiles](https://doc.rust-lang.org/cargo/reference/profiles.html) — every profile setting, its default and custom profiles.
- [The rustc book — Codegen options](https://doc.rust-lang.org/rustc/codegen-options/index.html) — the `-C` options such as `opt-level`, `lto`, `codegen-units`, `target-cpu` and `strip`.
- [The rustc book — Profile-guided Optimization](https://doc.rust-lang.org/rustc/profile-guided-optimization.html) — how PGO works in `rustc` without extra tools.
- [cargo-pgo](https://github.com/Kobzol/cargo-pgo) — the tool used in this lesson, with its BOLT support.
- [The Cargo Book — Configuration](https://doc.rust-lang.org/cargo/reference/config.html) — `.cargo/config.toml`, including `build.rustflags`.
- [The Rust Performance Book — Build Configuration](https://nnethercote.github.io/perf-book/build-configuration.html) — practical advice on which settings to try.

---

[🏠 Start](../START-HERE.md) · [← 19.7 SIMD: one instruction, many data](07-simd.md) · [📋 Phase 19](README.md) · [19.9 Performance of async and I/O code →](09-async-and-io-performance.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
