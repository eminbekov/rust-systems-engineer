<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 19.9 Performance of async and I/O code](09-async-and-io-performance.md) · [📋 Phase 19](README.md) · [19.11 Checkpoint →](11-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/19-performance-and-profiling/10-project-make-a-slow-program-fast.md) <!-- nav -->

# Lesson 19.10 — Project: make a slow program fast

> **Phase 19 — Performance and profiling** · Lesson 10 of 11 · ⏱ about 8 hours (4 sittings)

## What you will learn

- How to work like a performance engineer: freeze the output, measure, profile, change one thing, test, measure again.
- How to protect the behaviour of a program with golden tests before you touch its code.
- How to find the real hot spots of a program with a profiler instead of guessing.
- How to make a correct but slow log analyzer at least **10 times faster** without changing a single byte of its output.

## Before you start

- You finished [Lesson 19.9 — Performance of async and I/O code](09-async-and-io-performance.md) and the earlier lessons of this phase, especially [Lesson 19.3 — Profilers and flame graphs](03-profilers-and-flame-graphs.md) (`samply`, `sample`, `inferno`) and [Lesson 19.5 — Memory profiling and fewer allocations](05-memory-profiling-and-allocations.md).
- `hyperfine`, `samply` and `rustfilt` are installed (Lessons 09.11 and 19.3).
- You know golden or snapshot tests ([Lesson 09.5 — Snapshot testing](../09-testing-debugging-benchmarking/05-snapshot-testing.md)) and hash maps ([Lesson 04.9](../04-structs-enums-collections-errors/09-hash-maps.md)).

## New words

| Word | Meaning |
|---|---|
| **golden test** | A test that compares the whole output of a program with a saved file (the "golden" file) that is known to be right. |
| **golden master** | The saved output of the *old* program, used as the golden file while you rewrite it. It freezes the current behaviour, whatever it is. |
| **baseline** | The measured time of the program before any change. Every later number is compared with it. |
| **speed-up** | Old time divided by new time. 6.8 s → 0.68 s is a speed-up of 10 (10×). |
| **hot spot** | The small part of the code where most of the time goes. |
| **linear search** | Looking for an item by checking the elements of a list one by one. With n elements it costs about n/2 comparisons on average. |
| **Amdahl's law** | If a part takes a share *p* of the time, making only that part infinitely fast gives at most a speed-up of 1 / (1 − *p*). You already met it in Lesson 09.11. |

## 1. The method

Making a program fast is a loop, not a single clever idea:

```text
            ┌──────────────────────────────────────────────┐
            ▼                                              │
  freeze the output ──▶ measure ──▶ profile ──▶ change ONE thing
   (golden tests)       (baseline)  (hot spot)       │
                                                     ▼
                                     tests still green? ──no──▶ undo
                                                     │yes
                                                     ▼
                                     measure: faster? ──no──▶ undo
                                                     │yes
                                                     └──▶ commit, and loop again
```

Three rules make the loop work:

1. **Tests first.** A faster program that prints a different answer is a broken program. Golden tests catch every changed byte.
2. **Profile, do not guess.** In Lesson 19.3 the guess "the `HashMap`" was wrong. Look at where the time really goes.
3. **One change at a time.** If you change five things and the program gets slower, you do not know which change was bad.

## 2. The program: `p19-logstat`

The program reads a web-server **access log** (one line per HTTP request; HTTP is the HyperText Transfer Protocol of the web) and prints a summary: how many requests, which status codes, which endpoints are used most and how fast they are, and which clients send the most requests. It is correct. It is also slow.

Create the package and its folders:

```bash
cd ~/projects/rust
cargo new code/p19-logstat
cd code/p19-logstat
mkdir -p src/bin tests/data
```

### The log generator

`src/bin/gen.rs` writes a fake log with the same pseudo-random numbers every time, so your log is identical to the one in this lesson. **Do not change this file.**

```rust
//! Writes a fake web-server access log to standard output.
//! Usage: gen <number of lines>
use std::io::{self, BufWriter, Write};

/// xorshift: a tiny pseudo-random generator, so every run writes the same log.
struct Rng(u64);

impl Rng {
    fn next(&mut self) -> u64 {
        self.0 ^= self.0 << 13;
        self.0 ^= self.0 >> 7;
        self.0 ^= self.0 << 17;
        self.0
    }
    fn below(&mut self, n: u64) -> u64 {
        self.next() % n
    }
}

const PATHS: [&str; 12] = [
    "/",
    "/login",
    "/api/users",
    "/api/users/{}",
    "/api/users/{}/orders",
    "/api/orders/{}",
    "/api/products",
    "/api/products/{}",
    "/api/search",
    "/static/app.js",
    "/static/style.css",
    "/health",
];
const METHODS: [&str; 4] = ["GET", "GET", "POST", "DELETE"];
const STATUSES: [u16; 8] = [200, 200, 200, 201, 304, 404, 429, 500];

fn main() -> io::Result<()> {
    let lines: u64 = std::env::args()
        .nth(1)
        .and_then(|s| s.parse().ok())
        .unwrap_or(1000);
    let mut rng = Rng(2026);
    let mut out = BufWriter::new(io::stdout().lock());
    for i in 0..lines {
        if rng.below(1000) == 0 {
            writeln!(out, "garbage line {i}")?; // about 1 in 1000 lines is broken
            continue;
        }
        let secs = i / 50;
        let (h, m, s) = (10 + secs / 3600 % 14, secs / 60 % 60, secs % 60);
        // A few clients are very busy, most are not.
        let client = if rng.below(4) == 0 {
            rng.below(8)
        } else {
            rng.below(5_000)
        };
        let ip = format!(
            "10.{}.{}.{}",
            client / 65536,
            client / 256 % 256,
            client % 256
        );
        let method = METHODS[rng.below(4) as usize];
        let template = PATHS[rng.below(12) as usize];
        let mut path = template.replace("{}", &rng.below(100_000).to_string());
        if rng.below(10) == 0 {
            path.push_str(&format!("?page={}", rng.below(20)));
        }
        let status = STATUSES[rng.below(8) as usize];
        let mut micros = 200 + rng.below(20_000);
        if rng.below(100) == 0 {
            micros += rng.below(900_000); // 1 in 100 requests is very slow
        }
        let bytes = rng.below(50_000);
        writeln!(
            out,
            "2026-11-18T{h:02}:{m:02}:{s:02}Z {ip} {method} {path} {status} {}.{:03} {bytes}",
            micros / 1000,
            micros % 1000
        )?;
    }
    out.flush()
}
```

Each line has seven fields separated by single spaces: time, client IP (Internet Protocol) address, method, path, status code, latency in milliseconds with three decimals, and response size in bytes:

```text
2026-11-18T10:00:00Z 10.0.12.189 GET /static/style.css?page=19 500 0.992 8989
2026-11-18T10:00:00Z 10.0.6.127 DELETE /api/orders/54708 201 7.680 27991
2026-11-18T10:00:00Z 10.0.10.241 POST /static/style.css 200 7.937 29033
```

### The slow analyzer

`src/lib.rs` is the program you will speed up. Read it carefully: every line of output it prints is now the specification.

```rust
//! logstat: reads a web-server access log and prints a summary.
//!
//! This version is correct but slow. Do not change what it prints.

use std::collections::HashMap;
use std::io::{self, BufRead, Write};

/// All numbers collected for one endpoint, such as `GET /api/users/{id}`.
struct Endpoint {
    key: String,
    latencies_us: Vec<u64>,
    total_bytes: u64,
}

/// Reads log lines from `input` and writes the report to `out`.
pub fn run(input: impl BufRead, out: &mut impl Write) -> io::Result<()> {
    let mut lines = 0u64;
    let mut bad = 0u64;
    let mut status_classes: HashMap<String, u64> = HashMap::new();
    let mut endpoints: Vec<Endpoint> = Vec::new();
    let mut clients: Vec<(String, u64)> = Vec::new();

    for line in input.lines() {
        let line = line?;
        lines += 1;
        let fields: Vec<String> = line.split(' ').map(|s| s.to_string()).collect();
        if fields.len() != 7 {
            bad += 1;
            continue;
        }
        let ip = fields[1].clone();
        let method = fields[2].clone();
        let path = fields[3].clone();
        let (Ok(status), Ok(latency_ms), Ok(bytes)) = (
            fields[4].parse::<u16>(),
            fields[5].parse::<f64>(),
            fields[6].parse::<u64>(),
        ) else {
            bad += 1;
            continue;
        };
        let latency_us = (latency_ms * 1000.0).round() as u64;

        *status_classes
            .entry(format!("{}xx", status / 100))
            .or_insert(0) += 1;

        let key = format!("{} {}", method, normalize(&path));
        match endpoints.iter_mut().find(|e| e.key == key) {
            Some(e) => {
                e.latencies_us.push(latency_us);
                e.total_bytes += bytes;
            }
            None => endpoints.push(Endpoint {
                key,
                latencies_us: vec![latency_us],
                total_bytes: bytes,
            }),
        }

        match clients.iter_mut().find(|(c, _)| *c == ip) {
            Some((_, count)) => *count += 1,
            None => clients.push((ip, 1)),
        }
    }

    writeln!(out, "lines: {lines}")?;
    writeln!(out, "bad lines: {bad}")?;
    for class in ["2xx", "3xx", "4xx", "5xx"] {
        let count = status_classes.get(class).copied().unwrap_or(0);
        writeln!(out, "status {class}: {count}")?;
    }
    writeln!(out, "unique clients: {}", clients.len())?;

    endpoints.sort_by(|a, b| {
        b.latencies_us
            .len()
            .cmp(&a.latencies_us.len())
            .then(a.key.cmp(&b.key))
    });
    writeln!(
        out,
        "top endpoints (requests, p50, p95, p99 in ms, average bytes):"
    )?;
    for e in endpoints.iter().take(10) {
        let n = e.latencies_us.len() as u64;
        writeln!(
            out,
            "  {:<28} {:>8} {:>9} {:>9} {:>9} {:>7}",
            e.key,
            n,
            ms(percentile(&e.latencies_us, 50.0)),
            ms(percentile(&e.latencies_us, 95.0)),
            ms(percentile(&e.latencies_us, 99.0)),
            e.total_bytes / n
        )?;
    }

    clients.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)));
    writeln!(out, "top clients:")?;
    for (ip, count) in clients.iter().take(5) {
        writeln!(out, "  {ip:<15} {count:>8}")?;
    }
    Ok(())
}

/// Replaces every path segment made only of digits with `{id}` and drops the query string:
/// `/api/users/42/orders?page=2` becomes `/api/users/{id}/orders`.
fn normalize(path: &str) -> String {
    let without_query = path.split('?').next().unwrap_or("").to_string();
    let parts: Vec<String> = without_query
        .split('/')
        .map(|part| {
            if !part.is_empty() && part.chars().all(|c| c.is_ascii_digit()) {
                "{id}".to_string()
            } else {
                part.to_string()
            }
        })
        .collect();
    parts.join("/")
}

/// The `p`-th percentile with the nearest-rank method (Lesson 19.1).
fn percentile(values: &[u64], p: f64) -> u64 {
    let mut sorted = values.to_vec();
    sorted.sort();
    let rank = (p / 100.0 * sorted.len() as f64).ceil() as usize;
    sorted[rank.saturating_sub(1)]
}

/// Formats microseconds as milliseconds with three decimals: 12345 -> "12.345".
fn ms(us: u64) -> String {
    format!("{}.{:03}", us / 1000, us % 1000)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn normalize_replaces_ids_and_drops_the_query() {
        assert_eq!(
            normalize("/api/users/42/orders?page=2"),
            "/api/users/{id}/orders"
        );
        assert_eq!(normalize("/"), "/");
    }

    #[test]
    fn percentile_uses_nearest_rank() {
        let values: Vec<u64> = (1..=100).collect();
        assert_eq!(percentile(&values, 50.0), 50);
        assert_eq!(percentile(&values, 99.0), 99);
    }
}
```

**What happens:**

1. `run` reads the log line by line and splits every line at single spaces. A line without exactly seven fields, or with a status, latency or size that does not parse, counts as a **bad line**.
2. The latency is turned into whole microseconds. Keeping integers (not floats) makes the percentiles and the output exact, whatever order the values are collected in (remember Lesson 19.7).
3. Each request belongs to an **endpoint**: the method plus the path with numeric segments replaced by `{id}` and without the query string.
4. Endpoints and clients are kept in plain `Vec`s and found by looking through them.
5. At the end, endpoints are sorted by number of requests (ties by name) and the top 10 are printed with p50, p95 and p99 latency and average size; then the top 5 clients (ties by IP).

`src/main.rs` opens the file and calls `run`:

```rust
use std::fs::File;
use std::io::{self, BufReader};
use std::process;

fn main() -> io::Result<()> {
    let Some(path) = std::env::args().nth(1) else {
        eprintln!("usage: p19-logstat <access.log>");
        process::exit(2);
    };
    let input = BufReader::new(File::open(path)?);
    p19_logstat::run(input, &mut io::stdout().lock())
}
```

### The golden tests

`tests/data/small.log` contains hand-made edge cases. Create it with exactly these ten lines; line 4 is empty, and line 9 has **two** spaces after the IP address:

```text
2026-11-18T10:00:00Z 10.0.0.1 GET /api/users/42 200 12.345 1000
2026-11-18T10:00:01Z 10.0.0.2 GET /api/users/7?tab=orders 404 0.500 20
2026-11-18T10:00:02Z 10.0.0.1 POST /api/users 201 100.000 300

garbage line
2026-11-18T10:00:03Z 10.0.0.3 GET /api/users/42 abc 1.000 10
2026-11-18T10:00:04Z 10.0.0.1 GET /api/v2/users/42/ 500 3.001 7
2026-11-18T10:00:05Z 10.0.0.2 GET /api/users/12a 304 7.250 0
2026-11-18T10:00:06Z 10.0.0.3  GET /health 200 1.000 2
2026-11-18T10:00:07Z 10.0.0.2 GET /api/users/42 200 2.000 3000
```

`tests/data/small.expected` is what the original program prints for it:

```text
lines: 10
bad lines: 4
status 2xx: 3
status 3xx: 1
status 4xx: 1
status 5xx: 1
unique clients: 2
top endpoints (requests, p50, p95, p99 in ms, average bytes):
  GET /api/users/{id}                 3     2.000    12.345    12.345    1340
  GET /api/users/12a                  1     7.250     7.250     7.250       0
  GET /api/v2/users/{id}/             1     3.001     3.001     3.001       7
  POST /api/users                     1   100.000   100.000   100.000     300
top clients:
  10.0.0.1               3
  10.0.0.2               3
```

`tests/golden.rs` runs the real binary, like a user would (Lesson 09.2):

```rust
//! Golden tests: the program must print exactly what the original slow version printed.
use std::fs;
use std::path::Path;
use std::process::Command;

/// Runs the logstat binary on `log` and returns what it printed.
fn logstat(log: &Path) -> String {
    let output = Command::new(env!("CARGO_BIN_EXE_p19-logstat"))
        .arg(log)
        .output()
        .expect("failed to start p19-logstat");
    assert!(output.status.success(), "p19-logstat failed: {output:?}");
    String::from_utf8(output.stdout).expect("output is not UTF-8")
}

#[test]
fn small_log_with_edge_cases() {
    let expected = fs::read_to_string("tests/data/small.expected").unwrap();
    assert_eq!(logstat(Path::new("tests/data/small.log")), expected);
}

#[test]
fn generated_log_matches_the_original_output() {
    // Make the same 20,000-line log as in milestone M1.
    let generated = Command::new(env!("CARGO_BIN_EXE_gen"))
        .arg("20000")
        .output()
        .expect("failed to start gen");
    let log = std::env::temp_dir().join(format!("p19-logstat-{}.log", std::process::id()));
    fs::write(&log, generated.stdout).unwrap();
    let actual = logstat(&log);
    fs::remove_file(&log).unwrap();
    let expected = fs::read_to_string("tests/data/gen20000.expected").unwrap();
    assert_eq!(actual, expected);
}
```

**What happens:**

1. `env!("CARGO_BIN_EXE_p19-logstat")` is set by Cargo for integration tests: the path of the freshly built binary. The same works for `gen`.
2. The first test checks the edge cases: an empty line, garbage, a bad status, a query string, a trailing `/`, a segment that is not all digits (`12a`), a double space.
3. The second test generates 20,000 lines, runs the analyzer and compares with `tests/data/gen20000.expected`, the **golden master** that you make in milestone M1 from the original program.

## Common mistakes

> ⚠️ **Common mistake:** "improving" the parsing. `split_whitespace()` instead of `split(' ')` accepts the line with the double space, so `bad lines` changes. Trimming lines, accepting other number formats, or rounding the latency in a different way all change the output. The golden tests will tell you; listen to them.

> ⚠️ **Common mistake:** sorting with an unstable or incomplete order. The original sorts by count and then by name. If you drop the second rule, two endpoints with the same count may swap places on another run.

> ⚠️ **Common mistake:** measuring a debug build, or comparing two runs taken while the computer was busy. Use `--release` (Lesson 19.8) and `hyperfine` with several runs (Lesson 09.11).

> ⚠️ **Common mistake:** changing three things at once. When the result is slower or a test fails, you do not know why. Commit after every successful step.

## The project

### What you will build

A version of `p19-logstat` that prints **exactly** the same bytes as the original for every input, and runs **at least 10 times faster** on a log of one million lines. You keep a short `README.md` that documents every step with its measured time, like a lab notebook.

### Contract

- **Usage:** `p19-logstat <access.log>`. Without an argument it prints `usage: p19-logstat <access.log>` to standard error and exits with code 2. A file that cannot be opened is an error (exit code not 0).
- **Output:** byte for byte the same as the original program, for every input. In particular:
  - Fields are separated by **single** spaces; a line with other than exactly 7 fields is bad. The status must parse as `u16`, the latency as `f64`, the size as `u64`; otherwise the line is bad. Empty lines and bad lines still count in `lines`.
  - Latency in microseconds is `(latency_ms * 1000.0).round()`.
  - Endpoint key: method, one space, path without the part from the first `?`, with every non-empty all-digit segment replaced by `{id}`.
  - Percentiles: nearest rank on the sorted latencies of that endpoint. Average bytes: integer division.
  - Order: endpoints by request count descending, then key ascending (top 10); clients by count descending, then IP ascending (top 5).
- **Speed:** on the log made by `gen 1000000`, built with `--release`, `hyperfine` reports your version at least 10 times faster than the original.
- **Allowed:** any change to `src/lib.rs` and `src/main.rs`, including the signature of `run`, only the standard library, no `unsafe`. **Not allowed:** changing `src/bin/gen.rs`, the golden files or the tests (you may *add* tests).
- **Quality:** `cargo test`, `cargo clippy --all-targets -- -D warnings` and `cargo fmt --check` pass; one Git commit per milestone.

### Step-by-step plan

#### M1 — Set up and freeze the output

**Goal:** the package from Section 2 works, and the current output is saved as the golden master.

Create the files from Section 2 and a `.gitignore` with these lines (big files and binaries must not go into Git):

```text
/target
big.log
logstat-orig
*.json.gz
*.sample.txt
```

Build, try both binaries, and make the golden master from the **original** program:

```bash
cargo build --release
./target/release/gen 3
./target/release/gen 20000 > gen20000.log
./target/release/p19-logstat gen20000.log > tests/data/gen20000.expected
rm gen20000.log
cargo test
```

<details>
<summary>Hint</summary>

Run the binaries from `./target/release/` and not with `cargo run`: the package has two binaries, `gen` and `p19-logstat`, so `cargo run` without `--bin` does not know which one to start. If `small_log_with_edge_cases` fails, check `small.log` for the empty line 4 and the two spaces in line 9.

</details>

**Done when:** `./target/release/gen 3` prints the three log lines shown in Section 2, and `cargo test` ends with:

```text
test small_log_with_edge_cases ... ok
test generated_log_matches_the_original_output ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.53s
```

Commit: `git add code/p19-logstat` and `git commit -m "p19-logstat: slow original with golden tests"` (from `~/projects/rust`).

#### M2 — Measure the baseline

**Goal:** you know how fast the original is, and you keep a copy of it to compare against.

```bash
./target/release/gen 1000000 > big.log
./target/release/p19-logstat big.log
cp target/release/p19-logstat logstat-orig
hyperfine -N --warmup 1 --runs 5 './logstat-orig big.log'
```

`big.log` is about 67 MB (megabytes). The analyzer must print exactly this (if not, your `gen.rs` differs from the lesson):

```text
lines: 1000000
bad lines: 1003
status 2xx: 500101
status 3xx: 124344
status 4xx: 249390
status 5xx: 125162
unique clients: 5000
top endpoints (requests, p50, p95, p99 in ms, average bytes):
  GET /                           41992    10.305    19.412    20.195   24989
  GET /api/products/{id}          41875    10.256    19.329    20.182   24915
  GET /static/style.css           41796    10.362    19.364    48.355   24991
  GET /api/users/{id}/orders      41788    10.345    19.398    20.199   25125
  GET /api/users/{id}             41727    10.368    19.427    20.187   24993
  GET /api/users                  41723    10.335    19.386    44.195   25208
  GET /api/search                 41654    10.299    19.387    20.191   24918
  GET /api/orders/{id}            41646    10.245    19.380    26.649   24987
  GET /static/app.js              41599    10.397    19.396    52.844   25030
  GET /health                     41577    10.296    19.392    20.184   24986
top clients:
  10.0.0.6           31716
  10.0.0.0           31650
  10.0.0.4           31617
  10.0.0.7           31483
  10.0.0.5           31327
```

Start `README.md` with a table and write the baseline into it:

```text
| Step | Change | Time (mean ± σ) | Speed-up |
|---|---|---|---|
| 0 | original | 3.54 s ± 0.30 s | 1.0× |
```

<details>
<summary>Hint</summary>

On an M4 the baseline was:

```text
Benchmark 1: ./logstat-orig big.log (5 runs)
                mean     ±       σ          min     …     max
  Wall Time    3.539 s   ±   0.301 s      3.313 s   …   4.050 s
```

Your number will differ; what matters is that every later number is measured the same way on the same computer.

</details>

**Done when:** the output above matches exactly, `logstat-orig` exists, and the README has the baseline row. Commit.

#### M3 — Profile and find the hot spot

**Goal:** you know which lines of `src/lib.rs` take most of the time, from a profile, not a guess.

Add the profiling profile from Lesson 19.3 to `Cargo.toml`, build and record:

```toml
[profile.profiling]
inherits = "release"
debug = true
```

```bash
cargo build --profile profiling
samply record --save-only -o slow.json.gz ./target/profiling/p19-logstat big.log > /dev/null
samply load slow.json.gz
```

In the Firefox Profiler, open the *Call Tree*, tick *Invert call stack*, and look at the top rows. Then un-tick it and follow the biggest numbers down from `main`. Double-click the hottest function to see the source lines with their sample counts.

If you prefer text, `sample` from Lesson 19.3 gives the same answer:

```bash
./target/profiling/p19-logstat big.log > /dev/null &
sample $! 2 -mayDie -f slow.sample.txt
grep -A 8 'Sort by top of stack' slow.sample.txt | rustfilt
```

Write into the README the three functions with the most *self* time, and the source line in `src/lib.rs` that is responsible for each.

<details>
<summary>Hint</summary>

On an M4, the top of the inverted tree was `p19_logstat::run` itself, then `_platform_memcmp` and `DYLD-STUB$$memcmp` (the "stub" is a tiny jump into the system library). Together they were about 80% of all samples. `memcmp` compares bytes of memory; ask yourself which line compares strings again and again, and how many times per log line. The allocator functions (`_xzm_xzone_malloc_tiny`, `_xzm_free_main`) and `mach_absolute_time` come next.

</details>

**Done when:** the README names the hot spot and the line of code behind it, and explains in one sentence *why* it is slow (think: how many comparisons per log line, with 5,000 clients?). Commit.

#### M4 — Fix the biggest hot spot

**Goal:** remove the hot spot from M3 with one focused change, with all tests green.

Change only what the profile pointed to. Then:

```bash
cargo test
cargo build --release
hyperfine -N --warmup 1 --runs 5 './logstat-orig big.log' './target/release/p19-logstat big.log'
```

<details>
<summary>Hint</summary>

A `Vec` searched from the start for every line is a **linear search**. A data structure from Lesson 04.9 finds a key in about constant time. The output order does not depend on how you store the counts, because the final sort decides it (count, then IP). Use the `entry` API (API: Application Programming Interface, here the `entry` methods of `HashMap`) to count.

</details>

**Done when:** `cargo test` passes, and `hyperfine` shows a clear speed-up. On an M4 this one change gave about 5×:

```text
Benchmark 1: orig (5 runs)
  Wall Time    4.061 s   ±   0.582 s      3.419 s   …   4.772 s
Benchmark 2: first-fix (5 runs)
  Wall Time    0.838 s   ±   0.104 s      0.741 s   …   0.972 s            -79.4%  (4.8x faster)
```

(Here the two commands were given short names with `-n orig` and `-n first-fix`.) Add the row to the README. Commit.

#### M5 — Profile again, until 10×

**Goal:** reach a speed-up of at least 10, one measured step at a time.

Profile the new version exactly as in M3. The picture has changed: the old hot spot is gone, and things that were hidden behind it are now on top. Repeat the loop from Section 1: pick the biggest remaining hot spot, change one thing, `cargo test`, measure, write a README row, commit.

<details>
<summary>Hint 1 — where to look</summary>

After M4, most of the remaining time is **memory allocation**: `malloc`, `free` and copying bytes (`memmove`). Count the heap allocations the loop makes for *one* log line: the `String` from `lines()`, the `Vec<String>` and its seven `String`s, the three `clone()`s, `normalize` (a `String`, a `Vec<String>`, one `String` per segment, the `join`), and two `format!` calls. Lesson 19.5 showed how to measure this with `dhat`.

</details>

<details>
<summary>Hint 2 — ideas, one at a time</summary>

- Work with `&str` slices of the line instead of owned `String`s; `split(' ')` already gives slices.
- Read the whole file into one `String` (`std::fs::read_to_string`) and iterate with `.lines()` over slices of it, or reuse one `String` with `read_line`.
- Build the endpoint key into **one reused** `String` buffer (`clear()`, then `push_str`), and look it up in a `HashMap` with `get_mut(key.as_str())`; allocate a new `String` only when the key is new.
- Count status classes in a small array instead of a `HashMap<String, u64>` with a `format!` per line.
- At the end, sort each latency list **once** and take the three percentiles from it.

</details>

<details>
<summary>Hint 3 — traps</summary>

- `str::lines()` and `BufRead::lines()` both remove a trailing `\r`; `split('\n')` does not. Choose one that gives the same lines.
- Keep `split(' ')` and the "exactly 7 fields" rule. With an iterator, check that there is no 8th field.
- Keep `parse::<f64>()` for the latency: a hand-written parser must round exactly like `(x * 1000.0).round()` for every input, which is harder than it looks.

</details>

**Done when:** `cargo test` passes and `hyperfine` shows your version at least `10.0x faster` than `./logstat-orig` (an example of the format: `-90.4%  (10.5x faster)`). The README has one row per step. A careful version with only the standard library reaches about 20× on an M4. Commit.

#### M6 — Clean up and write down what you learned

**Goal:** the code is clean, and the README tells the story.

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```

Finish the README: the table of steps, the profile findings of each round, and two or three sentences on which change gave the most and why (use Amdahl's law: after the first fix, why could the second fix give so much more than it would have given at the start?).

**Done when:** all three commands pass with no output from `fmt` and no warnings from Clippy, `logstat-orig`, `big.log` and the profile files are not in Git (`git status` does not show them), and the README is complete. Commit.

### Stretch goals

- **20× or more**, still with only the standard library. Profile after every step.
- **Fewer allocations, measured:** use `dhat` (Lesson 19.5) to show the number of allocations before and after; aim for a number that does not grow with the number of lines.
- **Build settings:** try `lto`, `codegen-units = 1` and PGO (profile-guided optimization) from Lesson 19.8 on your final version. Is the gain larger than the noise?
- **Parallel:** split the file into chunks at line boundaries, analyze them on several threads with `std::thread::scope` (Lesson 14.4; `rayon` would break the "only the standard library" rule of the contract) and merge the results. The output must stay byte-identical, so think about how the merge keeps all counts and latency lists complete.
- **Memory map:** read the file with a memory map (Lesson 13.11) instead of `read_to_string` and measure whether it helps.

## Quiz

1. Why do you write the golden tests *before* you change any code?

<details>
<summary>Answer</summary>

They freeze the current behaviour. Every later change can then be checked automatically: if the output changes by one byte, a test fails, and you know that step broke something.

</details>

2. The profile shows `memcmp` near the top, but your code never calls `memcmp`. Where does it come from?

<details>
<summary>Answer</summary>

From comparing strings: `==` on two `String`s compares their bytes with `memcmp`. In the original program, the linear search through 5,000 client IPs makes thousands of such comparisons per log line.

</details>

3. The first fix removed about 80% of the time. What is the largest speed-up it could give, by Amdahl's law, and what did it give?

<details>
<summary>Answer</summary>

At most 1 / (1 − 0.8) = 5×. It gave about 4.8×, close to the limit. To go further, you must speed up the remaining 20%.

</details>

4. Why does the program store latencies as whole microseconds instead of `f64` milliseconds?

<details>
<summary>Answer</summary>

Integers are exact. Percentiles of integers do not depend on rounding, and an optimized version that collects the values in a different order still prints identical numbers. With floats, a different order of operations can change the last digits (Lesson 19.7).

</details>

5. Why do you keep `logstat-orig` and measure both binaries in the same `hyperfine` command?

<details>
<summary>Answer</summary>

Both run under the same conditions (same moment, same machine load, same file in the cache), so the comparison is fair, and `hyperfine` computes the speed-up for you.

</details>

## Summary

- Performance work is a loop: freeze the output, measure, profile, change one thing, test, measure, commit.
- Golden tests on the whole output turn "is it still correct?" into a command.
- The profile, not intuition, chooses the next change; after each fix the picture changes.
- Typical big wins in such programs: the right data structure instead of a linear search, and far fewer allocations per item.
- Amdahl's law tells you how much a fix *can* give; a README with one measured row per step shows what it *did* give.

## Sources and further reading

- [The Rust Performance Book](https://nnethercote.github.io/perf-book/) — especially the chapters "Heap Allocations", "Hashing" and "Standard Library Types".
- [samply on GitHub](https://github.com/mstange/samply) — the profiler used in milestone M3.
- [`std::collections::HashMap`](https://doc.rust-lang.org/std/collections/struct.HashMap.html) — the `entry` API and `get_mut` with borrowed keys.
- [The Cargo Book — Environment variables (`CARGO_BIN_EXE_<name>`)](https://doc.rust-lang.org/cargo/reference/environment-variables.html) — how integration tests find the binaries of the package.
- [hyperfine on GitHub](https://github.com/sharkdp/hyperfine) — options for comparing commands.

---

[🏠 Start](../START-HERE.md) · [← 19.9 Performance of async and I/O code](09-async-and-io-performance.md) · [📋 Phase 19](README.md) · [19.11 Checkpoint →](11-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
