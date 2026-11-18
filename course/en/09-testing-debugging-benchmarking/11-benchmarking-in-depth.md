<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 09.10 Logging and tracing](10-logging-and-tracing.md) · [📋 Phase 09](README.md) · [09.12 Continuous integration: GitHub Actions and GitLab CI →](12-continuous-integration.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/09-testing-debugging-benchmarking/11-benchmarking-in-depth.md) <!-- nav -->

# Lesson 09.11 — Benchmarking in depth

> **Phase 09 — Testing, debugging and benchmarking** · Lesson 11 of 14 · ⏱ about 120 minutes

## What you will learn

- How to benchmark one function on inputs of several sizes, and read the **throughput** number.
- How to keep setup work out of the measurement with `iter_batched`.
- How to save a **baseline** with `criterion`, change the code, and let `criterion` tell you whether it got faster or slower, and how to read its verdict.
- How to compare whole programs with `hyperfine`: parameter lists, warm-up, exporting a table.
- How to measure honestly on a laptop that is doing other things, and how to tell a real change from noise.

## Before you start

- You finished [Lesson 09.10 — Logging and tracing](10-logging-and-tracing.md). Span timings from that lesson are a quick look; this lesson is the careful measurement.
- You did the first benchmarks in [Lesson 05.12 — Benchmarking basics](../05-rust-intermediate/12-benchmarking-basics.md): release mode, `black_box`, a first `criterion` benchmark, a first `hyperfine` run and a checklist. Do not skip the rules there ("measure first", "release mode", "keep a correctness test"); this lesson uses them.
- `hyperfine` is installed (`cargo install --locked hyperfine`). Check with `hyperfine --version`.
- Your code goes in `~/projects/rust/code/p09-bench`.

## New words

| Word | Meaning |
|---|---|
| **throughput** | How much work is done per second, for example megabytes of text per second. Better than "time" when the input size changes. |
| **baseline** | A saved set of results from an earlier version of the code. A later run is compared with it. |
| **regression** (performance) | A change that makes the program slower. |
| **noise** | Random variation in a measurement that is not caused by your code: other programs, the operating system, the temperature of the chip. |
| **noise threshold** | The size of a change (in `criterion`, 2% by default) below which a difference is treated as noise. |
| **p-value** | A number between 0 and 1 from statistics. Small (below 0.05) means "the difference is unlikely to be luck". It says the change is *real*, not that it is *big*. |
| **standard deviation** (σ, "sigma") | How spread out repeated measurements are. A large σ means unreliable results. |
| **warm-up** | Runs that are made and thrown away before measuring, so that caches and the speed of the CPU (Central Processing Unit, the chip that runs the program) settle. |
| **setup** | Work you need before each measured call, such as making a copy of the input. You do not want to time it. |
| **Amdahl's law** | The speed-up of the whole program is limited by the part you did not improve. Making a part that takes 5% of the time ten times faster gives at most about 4.5%. |

## 1. A function worth measuring

We need something real to compare. The task: turn text such as `"3,14,15"` into the numbers `[3, 14, 15]`. There are two honest ways: use the standard library's `split` and `parse`, or walk through the bytes once and build the numbers by hand.

```bash
cd ~/projects/rust/code
cargo new --lib p09-bench
cd p09-bench
cargo add --dev criterion@0.8
mkdir benches src/bin
```

`src/lib.rs`:

```rust
/// Parses numbers separated by commas, for example `"3,14,15"`.
/// Version 1: the standard way, `split` + `parse`.
pub fn parse_split(text: &str) -> Vec<u32> {
    text.split(',').map(|part| part.parse().unwrap()).collect()
}

/// Version 2: one pass over the bytes, no intermediate strings.
pub fn parse_bytes(text: &str) -> Vec<u32> {
    let mut numbers = Vec::new();
    let mut current: u32 = 0;
    for &byte in text.as_bytes() {
        if byte == b',' {
            numbers.push(current);
            current = 0;
        } else {
            current = current * 10 + u32::from(byte - b'0');
        }
    }
    numbers.push(current);
    numbers
}

/// Builds a test text with `count` numbers, for example `"0,7,14,..."`.
pub fn make_text(count: usize) -> String {
    let numbers: Vec<String> = (0..count).map(|i| ((i * 7) % 100_000).to_string()).collect();
    numbers.join(",")
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn both_parsers_agree() {
        for count in [1, 2, 10, 1000] {
            let text = make_text(count);
            assert_eq!(parse_split(&text), parse_bytes(&text), "count = {count}");
        }
    }

    #[test]
    fn parses_a_small_example() {
        assert_eq!(parse_bytes("3,14,15"), vec![3, 14, 15]);
        assert_eq!(parse_split("3,14,15"), vec![3, 14, 15]);
    }
}
```

**What happens:**

1. `parse_split` is the version most people write. `parse_bytes` reads each byte: a comma ends the current number; a digit adds one more decimal place (`current * 10 + digit`). `b'0'` is the byte value of the character `0`, so `byte - b'0'` is the digit.
2. `make_text` makes a test input of any size. The numbers are always below 100,000, so they fit in a `u32`.
3. The correctness test comes **first**. A faster function that gives a different answer is worthless (Lesson 05.12).
4. A fairness warning: `parse_bytes` is faster partly because it does **less**. It does not check that the characters are digits, so it accepts `"1x,2"` and returns garbage, while `parse_split` panics. A comparison is honest only if both do the same job. Here we compare them on valid input only, and we know the limitation.

```bash
cargo test
```

```text
test tests::parses_a_small_example ... ok
test tests::both_parsers_agree ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

## 2. `criterion` with several input sizes

A single input size can mislead: a function that wins on 10 numbers may lose on 100,000. Measure a **range**. Add this to the end of `Cargo.toml` (you declared `harness = false` for one benchmark in Lesson 05.12; now there are two files):

```toml
[[bench]]
name = "parse"
harness = false

[[bench]]
name = "sort"
harness = false
```

`benches/parse.rs`:

```rust
use criterion::{BenchmarkId, Criterion, Throughput, criterion_group, criterion_main};
use p09_bench::{make_text, parse_bytes, parse_split};
use std::hint::black_box;

fn bench_parse(c: &mut Criterion) {
    let mut group = c.benchmark_group("parse");
    for count in [10, 1_000, 100_000] {
        let text = make_text(count);
        group.throughput(Throughput::Bytes(text.len() as u64));
        group.bench_with_input(BenchmarkId::new("split", count), &text, |b, text| {
            b.iter(|| parse_split(black_box(text)))
        });
        group.bench_with_input(BenchmarkId::new("bytes", count), &text, |b, text| {
            b.iter(|| parse_bytes(black_box(text)))
        });
    }
    group.finish();
}

criterion_group!(benches, bench_parse);
criterion_main!(benches);
```

**What happens:**

1. The loop makes one input for each size. The input is built **outside** the measured closure, so building it is not timed.
2. `group.throughput(Throughput::Bytes(n))` tells `criterion` how many bytes each call handles, so that it can report megabytes per second as well as time.
3. `bench_with_input(BenchmarkId::new("split", count), &text, |b, text| ...)` registers a benchmark named `parse/split/10` (group / function / parameter) and passes `&text` to the closure. `black_box` hides the input value from the optimizer; `criterion` itself hides the result.

Run it. The options after `--` go to `criterion`; here they shorten the warm-up and the measuring time, so that the whole run takes about half a minute instead of several minutes (use the defaults for real decisions):

```bash
cargo bench --bench parse -- --warm-up-time 1 --measurement-time 2
```

```text
parse/split/10          time:   [144.82 ns 152.25 ns 161.61 ns]
                        thrpt:  [159.33 MiB/s 169.12 MiB/s 177.80 MiB/s]
Found 8 outliers among 100 measurements (8.00%)
  3 (3.00%) high mild
  5 (5.00%) high severe
parse/bytes/10          time:   [85.756 ns 86.845 ns 88.237 ns]
                        thrpt:  [291.82 MiB/s 296.50 MiB/s 300.26 MiB/s]
parse/split/1000        time:   [8.2698 µs 9.4174 µs 11.002 µs]
                        thrpt:  [419.47 MiB/s 490.03 MiB/s 558.04 MiB/s]
Found 17 outliers among 100 measurements (17.00%)
parse/bytes/1000        time:   [1.8176 µs 1.8207 µs 1.8243 µs]
                        thrpt:  [2.4704 GiB/s 2.4752 GiB/s 2.4794 GiB/s]
parse/split/100000      time:   [797.42 µs 801.68 µs 806.08 µs]
                        thrpt:  [696.72 MiB/s 700.54 MiB/s 704.29 MiB/s]
parse/bytes/100000      time:   [178.46 µs 180.59 µs 183.53 µs]
                        thrpt:  [2.9883 GiB/s 3.0370 GiB/s 3.0732 GiB/s]
```

(Shortened: some outlier lines are left out, and your numbers will differ. The `MiB/s` unit is mebibytes per second, 1 MiB = 1,048,576 bytes. `GiB/s` is gibibytes per second.)

**How to read it:**

1. The `time` line is the interval `[low best high]`. For `bytes/1000` it is `[1.8176 µs 1.8207 µs 1.8243 µs]`: a very narrow interval, a trustworthy number.
2. `split/1000` is `[8.27 µs 9.42 µs 11.0 µs]`: the interval is about 30% wide and 17% of the samples are outliers. Something else was running on the Mac. Do **not** trust this number; run again when the machine is quiet (Section 6). `criterion` shows you the doubt; it is your job to read it.
3. `thrpt` (throughput) makes sizes comparable. `bytes` handles about 2.5 to 3 gibibytes per second at the large sizes, `split` about 0.5 to 0.7. So `bytes` is roughly 4 to 5 times faster for 1,000 and 100,000 numbers, and about 1.8 times faster for 10 numbers, where the fixed cost of starting (allocating the vector) matters more.
4. The conclusion we may draw: on **valid input** of these sizes, the hand-written loop is several times faster. We may *not* conclude that hand-written code is always better: it is less safe, and for a program that parses 100 numbers once, nobody can measure the difference (Amdahl's law: speed up what takes time).

> ⚠️ **Common mistake:** If your package is a library, plain `cargo bench -- --warm-up-time 1` fails:
>
> ```text
> error: Unrecognized option: 'warm-up-time'
> error: bench failed, to rerun pass `--lib`
> ```
>
> `cargo bench` also runs the **library's own test harness** as a benchmark, and the harness does not know `criterion`'s options. Name the benchmark you want: `cargo bench --bench parse -- ...`. (Or put `[lib]` with `bench = false` in `Cargo.toml`.)

To run only some benchmarks, give a filter. It is a regular expression matched against the full name; `$` means "end of the name":

```bash
cargo bench --bench parse -- "bytes/1000$"
```

## 3. Keeping setup out of the measurement

Sorting changes its input. If a benchmark sorts the same vector again and again, the second run sorts an already sorted vector, and you measure something else. You need a **fresh copy for every call**, but the copying should not be timed. `iter_batched` does exactly this: the first closure makes the input (not timed), the second closure is measured. Create `benches/sort.rs`:

```rust
use criterion::{BatchSize, Criterion, criterion_group, criterion_main};
use std::hint::black_box;

fn data() -> Vec<u32> {
    (0..10_000u32)
        .map(|i| i.wrapping_mul(2_654_435_761) % 100_000)
        .collect()
}

fn bench_sort(c: &mut Criterion) {
    let input = data();
    let mut group = c.benchmark_group("sort 10_000");
    group.bench_function("sort", |b| {
        b.iter_batched(|| input.clone(), |mut v| v.sort(), BatchSize::SmallInput)
    });
    group.bench_function("sort_unstable", |b| {
        b.iter_batched(
            || input.clone(),
            |mut v| v.sort_unstable(),
            BatchSize::SmallInput,
        )
    });
    group.bench_function("clone only", |b| b.iter(|| black_box(input.clone())));
    group.finish();
}

criterion_group!(benches, bench_sort);
criterion_main!(benches);
```

**What happens:**

1. `data()` makes 10,000 numbers that look random. `wrapping_mul` multiplies and wraps around on overflow instead of panicking (Lesson 02.2), so the numbers are mixed, not sorted.
2. `iter_batched(setup, routine, size)` calls `setup` to make a fresh vector, then times only `routine`. `BatchSize::SmallInput` tells `criterion` that the inputs are small, so it can prepare many at once.
3. `v.sort()` is the **stable** sort (equal elements keep their order). `v.sort_unstable()` is allowed to reorder equal elements and is often faster.
4. `"clone only"` measures the setup alone, so you know what `iter_batched` is hiding. A good habit: measure the part you removed, to see how large it was.

```bash
cargo bench --bench sort -- --warm-up-time 1 --measurement-time 2
```

```text
sort 10_000/sort        time:   [54.373 µs 54.736 µs 55.242 µs]
sort 10_000/sort_unstable
                        time:   [54.742 µs 54.868 µs 55.005 µs]
sort 10_000/clone only  time:   [522.97 ns 533.53 ns 544.20 ns]
```

**What we learn:** both sorts take 54 to 55 microseconds. The intervals overlap, so there is **no difference** on this input. A result of "no difference" is a real result: it tells you not to rewrite code for this. The clone costs 0.5 µs, one percent of the sort, so excluding it changed little here; for a cheaper function it would matter a lot.

## 4. Baselines: did my change help?

You change the code, and you want a number that says "20% faster" or "no change". `criterion` saves every run on disk (in `target/criterion/`) and compares the new run with the previous one. For a deliberate comparison, give the saved result a name, a **baseline**:

```bash
cargo bench --bench parse -- --save-baseline before "split/1000$"
```

Now run the same code again, this time **comparing** to that baseline with `--baseline before`:

```bash
cargo bench --bench parse -- --baseline before "split/1000$"
```

```text
parse/split/1000        time:   [7.6437 µs 7.6855 µs 7.7342 µs]
                        thrpt:  [596.67 MiB/s 600.46 MiB/s 603.74 MiB/s]
                 change:
                        time:   [+0.8594% +1.5051% +2.1350%] (p = 0.00 < 0.05)
                        thrpt:  [−2.0904% −1.4827% −0.8521%]
                        Change within noise threshold.
```

**How to read it:**

1. The code did not change, and the run says `+1.5%`: this is what noise looks like. `p = 0.00` says that the difference is "statistically real" (the two sets of samples are different), but `Change within noise threshold` says it is smaller than 2%, which `criterion` treats as noise anyway. **A small p-value does not mean a big change.** Read the size of the change as well.
2. Now make `parse_split` slower on purpose: build a `Vec<String>` of the parts first (one extra allocation per number), then parse them:

```rust
pub fn parse_split(text: &str) -> Vec<u32> {
    let parts: Vec<String> = text.split(',').map(|part| part.to_string()).collect();
    parts.iter().map(|part| part.parse().unwrap()).collect()
}
```

Run the comparison again:

```text
parse/split/1000        time:   [31.982 µs 36.966 µs 43.755 µs]
                        thrpt:  [105.47 MiB/s 124.84 MiB/s 144.29 MiB/s]
                 change:
                        time:   [+593.53% +733.49% +897.03%] (p = 0.00 < 0.05)
                        thrpt:  [−89.970% −88.002% −85.581%]
                        Performance has regressed.
```

`criterion` reports `Performance has regressed`: the new code takes 7 to 10 times longer. The three possible verdicts are `No change in performance detected`, `Change within noise threshold`, and `Performance has improved` / `Performance has regressed`. Put the original code back before you continue.

> 💡 **Tip:** Make a baseline before you start an optimization (`--save-baseline before`), and compare after each change. Commit the change that helped together with the numbers in the commit message ("parse: single pass over the bytes, 4x faster at 100k numbers on M4, see benches/parse.rs"). Later you can find out *why* the code looks the way it does (Lesson 01.4).

## 5. `hyperfine` for whole programs

`criterion` measures a function inside the process. Sometimes the question is about the **whole program**: reading the file, starting the process, everything. For that you use `hyperfine`, which runs a command many times and reports the time. Make a small program that uses both parsers. `src/bin/sumfile.rs`:

```rust
use p09_bench::{parse_bytes, parse_split};
use std::{env, fs, process};

fn main() {
    let args: Vec<String> = env::args().collect();
    if args.len() != 3 || (args[1] != "split" && args[1] != "bytes") {
        eprintln!("usage: sumfile <split|bytes> <file>");
        process::exit(2);
    }
    let text = fs::read_to_string(&args[2]).expect("cannot read the file");
    let text = text.trim();
    let numbers = if args[1] == "split" {
        parse_split(text)
    } else {
        parse_bytes(text)
    };
    let sum: u64 = numbers.iter().map(|&n| u64::from(n)).sum();
    println!("{} numbers, sum {}", numbers.len(), sum);
}
```

Make an input file of one million numbers (7.8 MB, megabytes: millions of bytes). `seq -f %.0f` prints plain digits (the macOS `seq` otherwise prints big numbers like `1e+06`, which our parsers cannot read), and `paste -sd,` joins the lines with commas:

```bash
seq -f %.0f 0 7 6999993 | paste -sd, - > numbers.txt
cargo build --release --bin sumfile
./target/release/sumfile split numbers.txt
./target/release/sumfile bytes numbers.txt
```

```text
1000000 numbers, sum 3499996500000
1000000 numbers, sum 3499996500000
```

Both programs give the same answer (the correctness check, before the timing). Now compare them:

```bash
hyperfine -N --warmup 3 --runs 20 -L impl split,bytes --command-name 'parse with {impl}' './target/release/sumfile {impl} numbers.txt'
```

```text
Benchmark 1: parse with split (20 runs)
                mean     ±       σ          min     …     max
  Wall Time     12.3 ms  ±     0.3 ms      11.8 ms  …    13.0 ms
  Memory        15.3 MiB ±     0.0 MiB     15.3 MiB …    15.3 MiB
 
Benchmark 2: parse with bytes (20 runs)
                mean     ±       σ          min     …     max
  Wall Time      5.8 ms  ±     0.2 ms       5.3 ms  …     6.3 ms           -53.2%  (2.1x faster)
  Memory        15.3 MiB ±     0.0 MiB     15.3 MiB …    15.4 MiB            0.0%
```

(The output format is the one of `hyperfine` 2.0; older versions print a slightly different layout.)

**What happens:**

1. `-N` runs the program directly, without a shell in between. Without it, `hyperfine` starts a shell for each run and subtracts its start-up time; for programs that take only a few milliseconds this correction is a source of noise, so `-N` is better.
2. `--warmup 3` makes three runs that are thrown away, so the file is in the operating system's cache and the CPU is at speed. `--runs 20` makes 20 measured runs.
3. `-L impl split,bytes` is a **parameter list**: the command is run once for each value, with `{impl}` replaced. `--command-name` gives a readable label.
4. Read `mean ± σ`: `12.3 ± 0.3 ms` says "about 12.3 ms, and the runs differ by about 0.3 ms". The difference between the two commands (12.3 vs 5.8) is many times larger than σ, so the conclusion is solid.
5. Notice the difference in size: the function-level test said `bytes` was about 4 times faster, the program-level test says 2.1 times. The rest of the time in the program (reading 7.8 MB from the disk cache, starting the process, summing) is the same for both. This is Amdahl's law in action. **The program-level number is the one a user feels.**

To keep a table for a pull request or a report, add `--export-markdown result.md`; hyperfine writes this:

```text
| Command | Mean Wall Time [ms] | Change | Factor |
|:---|---:|---:|:---|
| `parse with split` | 12.3 ± 0.3 |  |  |
| `parse with bytes` | 5.8 ± 0.2 | -53.2% | (2.1x faster) |
```

`--export-json result.json` writes all single measurements, for your own scripts and graphs.

When to use which tool:

| Question | Tool |
|---|---|
| How fast is this function, for inputs of different sizes? | `criterion` |
| Did my last commit make this function slower? | `criterion` with a baseline |
| How long does the whole command take, including start-up and I/O (Input/Output, reading and writing)? | `hyperfine` |
| Which of two programs or two versions of a program is faster? | `hyperfine` |
| *Why* is it slow, which line? | A profiler (a tool that samples where the time goes, for example `cargo flamegraph` or `samply`; not covered here) |

## 6. Measuring honestly on a real computer

Your Mac is not a laboratory. Everything below changes numbers.

1. **Noise from other work.** Browsers, builds and syncing run in the background. To see the effect, we started 14 CPU-hungry background programs and measured the same command again:

```text
quiet:   Wall Time   5.8 ms  ±   0.2 ms      5.3 ms  …    6.3 ms
loaded:  Wall Time   6.9 ms  ±   0.8 ms      5.7 ms  …    8.7 ms
```

(Example; your values will differ.) The mean moved by 19%, and σ became four times larger. The best run (`min`) is close to the quiet one: the *minimum* is often the most stable number, because noise only ever adds time. But you cannot hide behind the minimum when the real program runs on a busy server.

2. **Heat and power.** A laptop on battery or a hot chip slows itself down. Plug in the charger, wait a minute between big runs, and do not compare a run in the morning with one in the evening.
3. **Warm-up.** The first run is slower: files are not cached, branch predictors are empty. That is why both tools throw the first runs away.
4. **Look at the spread, not only the mean.** A wide `criterion` interval, many outliers or a large `σ` means: do not decide yet. Close other programs and run again.
5. **Compare in the same session.** Run old and new code one after the other (a baseline), not on different days or different machines. Numbers from another computer say nothing about yours.
6. **Use realistic input.** A benchmark with a tiny, perfectly ordered input can behave differently from real data. Try sizes from small to larger than your cache (Section 2).
7. **Keep a correctness check** next to every benchmark (Section 1), and compare the same work.
8. **Do not optimize what is not slow.** First find where the time goes (Amdahl's law). A function that takes 2% of the run cannot give you more than a 2% gain.

> ⚡ **Performance:** Benchmarks in a shared CI (continuous integration) machine (Lesson 09.12) are noisy: the neighbours change from run to run. Use them to catch big regressions (2 times slower), not 3% ones. Judge the small changes on your own quiet computer.

> 🧪 **Testing:** A benchmark is not a test, but a slow benchmark that has never run is a bug waiting to happen. At least make sure that `cargo bench --no-run` (build all benchmarks) succeeds in CI, so your benchmarks do not rot.

## Common mistakes

**1. `cargo bench -- --some-criterion-option` fails with `Unrecognized option`.** Add `--bench <name>` (Section 2).

**2. Timing a debug build.** `cargo bench` and `cargo build --release` optimize; `cargo run` does not. `hyperfine` on `target/debug/...` measures nothing useful.

**3. Putting the setup inside `b.iter`.** `b.iter(|| { let mut v = input.clone(); v.sort(); })` measures the clone and the sort together. Use `iter_batched`.

**4. Forgetting `black_box` on the input.** The compiler may compute the answer for a constant at compile time, and you measure nothing (Lesson 05.12).

**5. Trusting one noisy number.** If the interval is wide or the outliers are many, repeat the run before you write a conclusion.

**6. Comparing two different jobs.** A faster program that skips validation or checks is not a faster solution to the same problem.

## Try it yourself

1. Create `code/p09-bench` with `src/lib.rs`, `benches/parse.rs` and `benches/sort.rs` and run `cargo bench --bench parse -- --warm-up-time 1 --measurement-time 2`. Find the benchmark with the widest interval.
2. Save a baseline, make `parse_split` slower as in Section 4, and see `Performance has regressed`. Then restore the code and run once more: what does `criterion` say?
3. Create `numbers.txt` and compare the two implementations with `hyperfine`. Export the table with `--export-markdown`.
4. Run the `hyperfine` command twice, once while a video plays in a browser. Compare `σ`.

## Exercises

### Exercise 1 — Filter and read throughput (easy)

Run only the benchmark `bytes/100000`. How many bytes does one call process, and about how many gibibytes per second?

<details>
<summary>Hint</summary>

The filter is a regular expression: `"bytes/100000$"`. The text has 100,000 numbers, each followed by a comma; most of the numbers have 5 digits. The throughput line begins with `thrpt`.

</details>

<details>
<summary>Answer</summary>

```bash
cargo bench --bench parse -- --warm-up-time 1 --measurement-time 2 "bytes/100000$"
```

```text
parse/bytes/100000      time:   [178.46 µs 180.59 µs 183.53 µs]
                        thrpt:  [2.9883 GiB/s 3.0370 GiB/s 3.0732 GiB/s]
```

About 3 GiB/s. The text is 588,889 bytes long (100,000 numbers of up to 5 digits plus the commas), and `3.04 GiB/s * 180.6 µs` is about 0.59 MB, which matches. Why it works: the throughput is the number of bytes given to `Throughput::Bytes`, divided by the time.

</details>

### Exercise 2 — A test for the edge cases (medium)

Write a test in `src/lib.rs` that checks that both parsers agree on the single number `"0"` and on `"99999,0,5"`. Why is it a good idea to have this next to the benchmark?

<details>
<summary>Hint</summary>

Use `assert_eq!` with `vec![...]`. Put the test in the existing `mod tests`.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn parsers_agree_on_edge_cases() {
        for text in ["0", "99999,0,5"] {
            assert_eq!(parse_split(text), parse_bytes(text), "text = {text}");
        }
        assert_eq!(parse_bytes("99999,0,5"), vec![99_999, 0, 5]);
    }
```

```text
test tests::parsers_agree_on_edge_cases ... ok
```

Why it works: both functions are given the same inputs and must give the same vector; the last line also states the expected answer. The test protects the benchmark: if someone makes `parse_bytes` faster and breaks the answer, the test fails, and nobody celebrates a fake speed-up.

</details>

### Exercise 3 — A third version (hard)

Add `parse_split_with_capacity` (reserve the vector's room first with `Vec::with_capacity(text.matches(',').count() + 1)`, then `push` in a loop over `text.split(',')`). Add it to `benches/parse.rs` under the name `split_cap`, test it for agreement, benchmark it at 1,000 numbers and decide: is it faster than `split`?

<details>
<summary>Hint</summary>

`matches(',').count()` walks over the whole text once to count the commas, which costs time too. Compare the intervals of `split` and `split_cap`, and repeat the run if the intervals are wide.

</details>

<details>
<summary>Answer</summary>

```rust
/// Version 3: like version 1, but the vector gets its room first.
pub fn parse_split_with_capacity(text: &str) -> Vec<u32> {
    let mut numbers = Vec::with_capacity(text.matches(',').count() + 1);
    for part in text.split(',') {
        numbers.push(part.parse().unwrap());
    }
    numbers
}
```

Put the function in `src/lib.rs` above `mod tests` (Clippy complains about items placed after a test module). In `benches/parse.rs`, add the import `parse_split_with_capacity` and, next to `split`:

```rust
        group.bench_with_input(BenchmarkId::new("split_cap", count), &text, |b, text| {
            b.iter(|| parse_split_with_capacity(black_box(text)))
        });
```

Result on a quiet run: `split/1000` about 7.6 µs, `split_cap/1000` about 8.0 µs. The difference is within a few percent: counting the commas costs about as much as the vector growth it avoids, and parsing each number dominates anyway. The honest conclusion is "no useful improvement". (If you see a big difference on your computer, check for noise first: on a busy Mac the `split` value jumped between 7.6 and 31 µs in our runs.)

Why it works: `Vec` grows by doubling (Lesson 04.7), so the copying is only a few percent of the work compared with parsing text.

</details>

## Quiz

1. Why measure several input sizes?

<details>
<summary>Answer</summary>

Because the faster version at one size can be slower at another. The fixed costs dominate small inputs, and the cost per item dominates large ones.

</details>

2. What does `p = 0.00 < 0.05` together with `Change within noise threshold` mean?

<details>
<summary>Answer</summary>

The two sets of samples are statistically different, but the difference is smaller than `criterion`'s noise threshold (2%). It is real but too small to care about; do not report it as an improvement or a regression.

</details>

3. Why use `iter_batched` for a sort benchmark?

<details>
<summary>Answer</summary>

Sorting modifies its input. `iter_batched` gives each call a fresh input and does not count the time of making it, so every measured call sorts the same kind of data.

</details>

4. Why does `hyperfine` report a smaller speed-up (2.1x) than `criterion` (about 4x) for the same two functions?

<details>
<summary>Answer</summary>

The whole program also reads the file, starts the process and sums the numbers; that part is the same for both and is not sped up (Amdahl's law).

</details>

5. Which number of a noisy measurement is often the most stable one?

<details>
<summary>Answer</summary>

The minimum, because noise only adds time. But the typical user feels the mean, so report both, together with the spread.

</details>

6. What does the `-N` option of `hyperfine` do?

<details>
<summary>Answer</summary>

It runs the command directly, without starting a shell for each run, which removes the shell's start-up time and the noise of correcting for it.

</details>

## Summary

- Measure a range of input sizes and read throughput. A narrow interval is a trustworthy result; a wide one or many outliers mean "repeat when the machine is quiet".
- `iter_batched` keeps setup out of the measurement; measure the setup alone to know its size.
- Save a baseline with `--save-baseline` and compare with `--baseline`. Read the verdict and the size of the change, not only the p-value.
- `hyperfine` measures whole programs: `-N`, `--warmup`, `--runs`, `-L` parameter lists, `--export-markdown`. It shows what the user feels; `criterion` shows where the function stands.
- Be fair (same job, valid input, a correctness test next to the benchmark), be patient (plugged in, quiet machine, repeat), and optimize only what takes time.
- A result of "no difference" is a result. Write the numbers into the commit message.

## Sources and further reading

- [The Criterion.rs User Guide](https://bheisler.github.io/criterion.rs/book/) — the official guide: comparing functions, `iter_batched`, baselines, command-line options, and how the analysis works.
- [criterion on docs.rs](https://docs.rs/criterion/latest/criterion/) — the API: `BenchmarkGroup`, `Throughput`, `BatchSize`, `BenchmarkId`.
- [hyperfine on GitHub](https://github.com/sharkdp/hyperfine) — all options (`--prepare`, `--parameter-scan`, exports) and advice on benchmarking shell commands.
- [The Rust Performance Book](https://nnethercote.github.io/perf-book/) — how to find and fix slow code in Rust: profiling, allocation and build settings.

---

[🏠 Start](../START-HERE.md) · [← 09.10 Logging and tracing](10-logging-and-tracing.md) · [📋 Phase 09](README.md) · [09.12 Continuous integration: GitHub Actions and GitLab CI →](12-continuous-integration.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
