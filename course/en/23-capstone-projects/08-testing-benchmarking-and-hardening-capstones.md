<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 23.7 Capstone option 5: your own idea](07-capstone-your-own-idea.md) · [📋 Phase 23](README.md) · [23.9 Presenting your work →](09-presenting-your-work.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/23-capstone-projects/08-testing-benchmarking-and-hardening-capstones.md) <!-- nav -->

# Lesson 23.8 — Testing, benchmarking and hardening your capstone

> **Phase 23 — Capstone projects** · Lesson 8 of 10 · ⏱ about 120 minutes

## What you will learn

- What the hardening stage of a capstone is, and how to find the gaps in your tests with a *test matrix* built from your design document.
- How a property-based round-trip test finds a real bug that example tests miss, and how proptest keeps the failing case for ever.
- How to build one CI pipeline for a whole capstone, with fast jobs for every push and slow jobs (simulation, fuzzing) that run longer at night.
- How to run benchmarks that other people can trust, and how to write them up in `docs/results.md`.
- A hardening checklist that ties together the reliability, security and performance lessons of Phases 19–20.

## Before you start

- You are in the middle of a capstone: the design document is written (Lesson 23.2) and at least M2 of your option ([23.3](03-capstone-distributed-key-value-store.md)–[23.7](07-capstone-your-own-idea.md)) is done.
- This lesson collects tools that earlier lessons taught in detail: property-based tests ([Lesson 09.4](../09-testing-debugging-benchmarking/04-property-based-testing.md)), mutation testing ([Lesson 09.7](../09-testing-debugging-benchmarking/07-code-coverage-and-mutation-testing.md)), fuzzing ([Lessons 09.8](../09-testing-debugging-benchmarking/08-fuzzing.md) and [20.3](../20-reliability-and-hardening/03-fuzzing-in-depth.md)), CI ([Lesson 09.12](../09-testing-debugging-benchmarking/12-continuous-integration.md)), load testing ([Lesson 19.2](../19-performance-and-profiling/02-load-testing.md)), fault injection ([Lesson 20.5](../20-reliability-and-hardening/05-fault-injection-and-chaos-testing.md)) and the production-readiness checklist ([Lesson 20.9](../20-reliability-and-hardening/09-production-readiness-checklist.md)). Here we apply them to a whole system; we do not teach them again.
- `actionlint` is installed (Lesson 09.12).

## New words

| Word | Meaning |
|---|---|
| **hardening** | The work of removing, one by one, the ways in which a program can fail, be overloaded or be attacked. |
| **test matrix** | A table that maps every requirement and every failure mode of the design document to the test that proves it, and shows which are still missing. |
| **round-trip property** | The rule "decode(encode(x)) == x for every x". It is one of the most useful properties for any format or protocol. |
| **shrinking** | What a property-testing tool does after it finds a failing input: it tries smaller and simpler inputs until it finds the smallest one that still fails. |
| **regression file** | A file in which proptest saves the seeds of failing cases, so that they are tried again first in every later run. |
| **fast lane / slow lane** | CI jobs that run on every push in a few minutes (fast) and jobs that run longer, after the fast ones or at night (slow). |
| **benchmark methodology** | The written description of *how* you measured: hardware, software versions, workload, warm-up, repetitions and the exact commands. |
| **noise** | Variation between measurements that is not caused by your code: other programs, CPU frequency changes, thermal throttling. |
| **headroom** | The distance between the load at which you run and the load at which the system starts to fail its goals. |
| **p50 / p99 / p99.9** | Percentiles of latency (Lesson 19.1): half of the requests are faster than p50 (the median), 99% are faster than p99, 99.9% are faster than p99.9. |
| **UTC** (Coordinated Universal Time) | The world's reference time, without time zones or summer time; CI schedules use it. |
| **CPU** (Central Processing Unit) | The processor; "CPU use" is how much of its time a program takes. |
| **VM** (Virtual Machine) | A whole computer simulated by software; the course's Linux VM is called `dev` (Lesson 11.1). |
| **TCP** (Transmission Control Protocol) | The internet protocol that gives two programs a reliable, ordered stream of bytes. |
| **HTTP** (HyperText Transfer Protocol) | The request-and-response protocol of the web; HTTPS is HTTP over an encrypted connection. |

## 1. The hardening stage

During the milestones you wrote tests for what each milestone promised. The hardening stage (about 15% of your time, Lesson 23.1) looks at the **whole** system and asks: which promise or failure mode has no test yet?

Build a **test matrix** from your design document. Every requirement (Section 4 of the document) and every row of the failure-mode table (Section 7) becomes a row:

| ID | Requirement or failure | Test that proves it | Kind | Status |
|---|---|---|---|---|
| R1 | p99 `GET` < 5 ms at 10,000 QPS (queries per second) on one node | `bench/get_latency.sh` with `oha` | load test | done (4.1 ms) |
| R2 | acknowledged writes survive `kill -9` | `m1_acknowledged_writes_survive_kill` | crash test | done |
| R3 | old leader cannot serve linearizable reads | `sim::partitioned_leader_refuses_reads` | simulation | done |
| F4 | follower disk full | — | fault injection | **missing** |
| F5 | corrupt log record on disk | `fuzz/log_decoder` | fuzzing | done, 1 h |
| F6 | client sends a 4 GiB length field | — | unit + fuzz | **missing** |

The ID (identifier) column numbers the rows: R for requirements, F for failure modes, so that a test, an issue or a commit message can refer to "F6". The rows marked **missing** are your work for this stage, ordered by risk. The same table goes into the README in Lesson 23.9, because it is the honest answer to "how do you know it works?".

> 🧪 **Testing:** Run `cargo mutants` (Lesson 09.7) on the core crate once during hardening. Each *surviving mutant* — a change to your code that no test noticed — points to a missing assertion. You do not need to kill all of them; you need to look at them.

## 2. A property that finds a real bug

Example tests check the cases you thought of. Property tests check cases you did not think of. The most useful property for any format, protocol or file layout is the **round trip**: encoding and then decoding gives back exactly what you started with.

Here is a small frame format with a bug that example tests do not find. Create a library package for practice:

```bash
cd ~/projects/rust
cargo new --lib code/p23-harden
cd code/p23-harden
cargo add --dev proptest@1
```

`src/lib.rs`:

```rust
//! A tiny frame format: kind (1 byte), payload length (u16, big-endian), payload.

#[derive(Debug, Clone, PartialEq)]
pub struct Frame {
    pub kind: u8,
    pub payload: Vec<u8>,
}

pub fn encode(frame: &Frame) -> Vec<u8> {
    let len = frame.payload.len() as u16;
    let mut out = vec![frame.kind];
    out.extend_from_slice(&len.to_be_bytes());
    out.extend_from_slice(&frame.payload);
    out
}

pub fn decode(bytes: &[u8]) -> Result<Frame, String> {
    let [kind, hi, lo, rest @ ..] = bytes else {
        return Err("frame too short".to_string());
    };
    let len = u16::from_be_bytes([*hi, *lo]) as usize;
    if rest.len() != len {
        return Err(format!("length says {len} bytes, got {}", rest.len()));
    }
    Ok(Frame {
        kind: *kind,
        payload: rest.to_vec(),
    })
}

#[cfg(test)]
mod tests {
    use super::*;
    use proptest::prelude::*;

    #[test]
    fn a_small_frame_round_trips() {
        let frame = Frame {
            kind: 1,
            payload: b"hello".to_vec(),
        };
        assert_eq!(decode(&encode(&frame)), Ok(frame));
    }

    proptest! {
        #[test]
        fn every_frame_round_trips(kind in any::<u8>(), len in 0usize..70_000, fill in any::<u8>()) {
            let frame = Frame { kind, payload: vec![fill; len] };
            let decoded = decode(&encode(&frame));
            prop_assert!(decoded == Ok(frame), "a frame with {} payload bytes did not round-trip", len);
        }
    }
}
```

**What happens:**

1. `encode` writes the kind, the payload length as two big-endian bytes, and the payload. `as u16` converts the length — and **silently cuts off** every bit above 16 if the payload is longer than 65,535 bytes.
2. `decode` uses a slice pattern (`[kind, hi, lo, rest @ ..]`, Lesson 06.3) to take the first three bytes, and checks that the rest has exactly the announced length.
3. The example test uses a 5-byte payload. It passes, and it would pass for any payload you would naturally write by hand.
4. The property test generates 256 random frames: any kind, a length from 0 to 69,999 and a fill byte. Generating `len` and `fill` instead of a random `Vec` keeps the failure report short (a random 65,536-byte vector would be printed byte by byte).

```bash
cargo test
```

```text
running 2 tests
. 1/2
tests::every_frame_round_trips --- FAILED

failures:

---- tests::every_frame_round_trips stdout ----
proptest: Saving this and future failures in /Users/you/projects/rust/code/p23-harden/proptest-regressions/lib.txt
proptest: If this test was run on a CI system, you may wish to add the following line to your copy of the file. (You may need to create it.)
cc 577adc906d1865d9ceef5fb33e28dd8300da7a36149e355ea2c4f1ed7514e9ac

thread 'tests::every_frame_round_trips' (6666320) panicked at src/lib.rs:45:5:
Test failed: a frame with 65536 payload bytes did not round-trip at src/lib.rs:50.
minimal failing input: kind = 0, len = 65536, fill = 0
	successes: 75
	local rejects: 0
	global rejects: 0
```

(Your path, seed and number of successes will differ.) Read it from the bottom of the report up:

1. After 75 passing cases, proptest found a failing frame, then **shrank** it: `kind = 0`, `fill = 0`, and `len = 65536` — the smallest length that fails. That number tells you the bug at once: 65,536 is 2¹⁶, the first length that does not fit in a `u16`.
2. The seed of the failing case is saved in `proptest-regressions/lib.txt`. Commit this file: every later run on every machine tries this case first, so the bug can never come back unnoticed.

The fix is to refuse what the format cannot express, instead of producing a wrong frame:

```rust
pub fn encode(frame: &Frame) -> Result<Vec<u8>, String> {
    let len = u16::try_from(frame.payload.len())
        .map_err(|_| format!("payload of {} bytes is too long", frame.payload.len()))?;
    let mut out = vec![frame.kind];
    out.extend_from_slice(&len.to_be_bytes());
    out.extend_from_slice(&frame.payload);
    Ok(out)
}
```

and the property changes with it — a frame either round-trips, or `encode` refuses it because it is too long:

```rust
proptest! {
    #[test]
    fn every_frame_round_trips(kind in any::<u8>(), len in 0usize..70_000, fill in any::<u8>()) {
        let frame = Frame { kind, payload: vec![fill; len] };
        match encode(&frame) {
            Ok(bytes) => prop_assert_eq!(decode(&bytes), Ok(frame)),
            Err(_) => prop_assert!(len > usize::from(u16::MAX), "only too long payloads may fail"),
        }
    }
}
```

(The example test now calls `encode(&frame).unwrap()`.) Both tests pass:

```text
test tests::a_small_frame_round_trips ... ok
test tests::every_frame_round_trips ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🔒 **Security:** `as` between integer types never fails; it cuts or wraps. In a protocol, a wrong length field is how one message is read as two, or how a reader allocates gigabytes. Use `try_from` for every length that crosses a boundary, and let Clippy's `cast_possible_truncation` lint (in `clippy::pedantic`) point out the casts you missed.

## 3. One pipeline: a fast lane and a slow lane

A capstone has tests that take seconds and tests that take an hour. If everything runs on every push, CI becomes so slow that you stop waiting for it. Split the work:

- **Fast lane** (every push, a few minutes): format, Clippy, unit and integration tests, the `done-when.sh` checks of Lesson 23.7.
- **Slow lane** (after the fast lane passes, and longer every night): simulation with many seeds, fuzzing of every target, benchmark builds.

`.github/workflows/ci.yml`:

```yaml
name: ci

on:
  push:
    branches: [main]
  pull_request:
  schedule:
    - cron: "0 2 * * *" # every night at 02:00 UTC

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

env:
  CARGO_TERM_COLOR: always

jobs:
  gate:
    name: fmt, clippy, tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install stable --profile minimal --component rustfmt,clippy
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --check
      - run: cargo clippy --all-targets -- -D warnings
      - run: cargo test --all-targets
      - run: ./scripts/done-when.sh

  simulation:
    name: simulation with fixed seeds
    needs: gate
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install stable --profile minimal
      - uses: Swatinem/rust-cache@v2
      - run: cargo test --release --test simulation -- --include-ignored

  fuzz:
    name: fuzz every target
    needs: gate
    runs-on: ubuntu-latest
    timeout-minutes: 60
    env:
      FUZZ_SECONDS: ${{ github.event_name == 'schedule' && '600' || '60' }}
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install nightly --profile minimal
      - run: cargo install --locked cargo-fuzz
      - name: Fuzz each target for a while
        run: |
          for target in $(cargo +nightly fuzz list); do
            cargo +nightly fuzz run "$target" -- -max_total_time="$FUZZ_SECONDS"
          done

  benches:
    name: benchmarks still compile
    needs: gate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install stable --profile minimal
      - uses: Swatinem/rust-cache@v2
      - run: cargo bench --no-run
```

**What happens:**

1. Three triggers: pushes to `main`, pull requests, and a `schedule` with a `cron` expression (minute, hour, day of month, month, day of week) that starts the workflow every night.
2. `gate` is the fast lane. The other jobs say `needs: gate`, so they start only when the fast lane is green; there is no point in fuzzing code that does not compile.
3. `simulation` runs one test target named `simulation` (a file `tests/simulation.rs`) in release mode and includes the tests marked `#[ignore]` — a common way to keep slow tests out of the normal `cargo test`.
4. `fuzz` chooses its duration with an expression: 600 seconds per target at night, 60 seconds otherwise. `cargo fuzz list` prints the names of all fuzz targets, so a new target is fuzzed without changing the workflow.
5. `benches` only compiles the benchmarks (`--no-run`). Shared CI machines are too noisy for real measurements (Section 4), but a benchmark that no longer compiles should be noticed at once.

GitHub's runners cannot be started from your Mac, but the file can be checked. `actionlint` (Lesson 09.12) prints nothing and exits with 0:

```bash
actionlint .github/workflows/ci.yml
```

A typo in a job name (`needs: gaet`) is the kind of mistake that otherwise appears only after a push:

```text
.github/workflows/bad.yml:33:3: job "simulation" needs job "gaet" which does not exist in this workflow [job-needs]
   |
33 |   simulation:
   |   ^~~~~~~~~~~
```

> 💡 **Tip:** When the nightly run finds a fuzz crash or a failing seed, turn it into a normal test before you fix it (a unit test with the crashing input, or the seed in the fixed-seed list). The fast lane then protects you from that bug for ever.

## 4. Benchmarks you can trust

A number without its conditions cannot be checked, compared or repeated. Before you measure, write down the **methodology**:

1. **Hardware and software:** CPU model and number of cores, memory, disk type, operating system and kernel version, Rust version, build profile (always `--release`).
2. **Workload:** the exact command, request or message size, number of connections, rate (an open-model rate such as `oha -q 5000` avoids coordinated omission, Lesson 19.1), and duration.
3. **Warm-up:** discard the first seconds, while caches fill and connections open.
4. **Repetitions:** run each measurement at least three times and report the median and the spread. If two runs differ by more than about 10%, find the noise before you trust any of them.
5. **Baseline:** compare with something: the version before your change, a direct path without your system (the proxy of Lesson 23.4), or a real system doing the same job.
6. **Explanation:** a profile (Lesson 19.3) at the interesting load, and one paragraph that says where the time goes.

Then `docs/results.md` follows a fixed shape:

```markdown
# Results

## Setup
- Machine: Apple M4 Pro, 12 cores, 24 GB; macOS 26; Rust 1.100.0, --release
- Linux runs: VM `dev` (Lima, Ubuntu 26.04, 4 vCPU, 8 GB)
- Commit: 3f2c9e1 (tag m5)

## Workload
oha -z 60s -q 5000 -c 64 --rand-regex-url 'http://127.0.0.1:7001/v1/kv/key-[0-9]{1,3}'
1 KiB values, 1,000 keys, 3 nodes on one machine, 10 s warm-up discarded

## Numbers (median of 3 runs)
| Operation | Rate | p50 | p99 | p99.9 | Errors |
|---|---|---|---|---|---|
| GET linearizable | 5,000/s | 0.41 ms | 1.9 ms | 4.8 ms | 0 |
| PUT, batching off | 5,000/s | 2.3 ms | 7.5 ms | 15 ms | 0 |
| PUT, batching on | 5,000/s | 1.1 ms | 3.2 ms | 6.0 ms | 0 |

## Where the time goes
(flame graph: profile-put.svg) About 60% of PUT time is fsync ...

## Limits and honesty
- All nodes run on one machine, so network latency is not realistic.
- ...
```

(The numbers above show the *shape* of the table, not real measurements. In the workload line, `--rand-regex-url` makes `oha` build every request URL (Uniform Resource Locator, the web address) from the pattern, so each request reads a random key from `key-0` to `key-999`.) The "Limits and honesty" section is not optional. Every benchmark has conditions that make it better or worse than reality; saying so is what makes the other numbers believable.

> ⚡ **Performance:** Report percentiles, not averages, and always next to the rate at which they were measured. "p99 = 3 ms" means nothing without "at 5,000 requests per second"; at twice the rate it may be 300 ms. Find the rate at which your latency goal is first missed and report it as the system's capacity — then you also know your **headroom**.

## 5. The hardening checklist

The production-readiness checklist of Lesson 20.9 asks these questions of a service. For a capstone, go through them once during hardening and write the answer (or "not applicable, because …") into your README:

| Area | Question | How you check it |
|---|---|---|
| Input | Is every length, size and count from outside checked before use? | unit tests with extreme values; fuzzing (Lessons 10.4, 20.3) |
| Panics | Can outside input cause a panic (`unwrap`, indexing, `as` casts)? | `clippy::unwrap_used` on non-test code; fuzzing |
| Limits | Does every queue, buffer, connection count and request size have a limit? | a test at the limit for each (Lesson 20.8) |
| Time | Does every network call and every lock wait have a timeout? | fault injection with a slow peer (Lesson 20.5) |
| Crashes | Is every acknowledged change durable, and is recovery tested? | `kill -9` tests (Lesson 17.2) |
| Shutdown | Does `SIGTERM` finish in-flight work and release resources? | an end-to-end shutdown test (Lesson 14.14) |
| `unsafe` | Does every `unsafe` block have a `// SAFETY:` comment and a Miri run? | review and `cargo miri test` (Lessons 10.12, 20.4) |
| Supply chain | Are dependencies checked for advisories and licences? | `cargo deny check`, `cargo audit` in CI (Lessons 10.13, 20.7) |
| Visibility | Can you tell from metrics and logs alone what the system is doing? | look at a dashboard during a fault test (Lesson 21.9) |

## Common mistakes

**1. Benchmarking a debug build.** Debug builds are often ten times slower and have different bottlenecks. Always `--release`, and say so in the methodology.

**2. One run, one number.** The number may be noise. Fix: three runs, the median, and the spread.

**3. Running the benchmark client and the system on one busy machine without saying so.** They compete for CPU, and the numbers describe the competition. Fix: separate machines (or the VM for one side), or a clear note under "Limits and honesty".

**4. Deleting `proptest-regressions/`.** The file looks like noise, so someone adds it to `.gitignore`, and an old bug returns silently. Fix: commit it.

**5. Slow tests in the fast lane.** CI takes forty minutes, so you stop waiting for it and start pushing on red. Fix: Section 3 — fast lane under about five minutes, everything else after it or at night.

**6. "Hardening" as a last-week activity only.** A week of tests cannot find what a design never considered. Fix: the test matrix starts with the design document; hardening fills the gaps, it does not start the work.

## Try it yourself

1. Create `code/p23-harden` from Section 2 with the buggy `encode`, run `cargo test`, and read the shrunk input in the report.
2. Open `proptest-regressions/lib.txt` and find the line `# shrinks to kind = 0, len = 65536, fill = 0`.
3. Apply the fix and the new property from Section 2, and run `cargo test` again: both tests pass, and proptest has tried the saved case first.
4. Copy the workflow from Section 3 into your capstone as `.github/workflows/ci.yml`, adapt the names of your test target and scripts, and run `actionlint` on it.
5. Write the test matrix of your capstone (Section 1) and mark the missing rows.

## Exercises

### Exercise 1 — Fill a test matrix (easy)

Your proxy capstone (option 2) has these rows in its failure-mode table: "backend is slow", "backend crashes in the middle of a response", "client sends headers very slowly", "configuration file is invalid on reload". For each, name the kind of test (unit, integration, fault injection, fuzz, load) and one concrete check.

<details>
<summary>Hint</summary>

Think about what the test must *control*: a backend you can make slow or kill, a client you can make slow, a file you can make invalid.

</details>

<details>
<summary>Answer</summary>

| Failure | Kind | Check |
|---|---|---|
| backend is slow | integration with a controllable `test-backend` | a backend told to sleep 3 s with `timeout_ms = 1000` gives `504` after about 1 s |
| backend crashes mid-response | fault injection | the backend closes the connection after half the body; the client gets an error (not a truncated `200`), and a `GET` is not retried after bytes were sent |
| client sends headers slowly | integration with a slow raw TCP client | after `header_timeout_ms`, the proxy closes the connection and the connection count drops |
| invalid configuration on reload | unit test of `load` + integration | `POST /reload` with a broken file returns `400`, and requests continue with the old routes |

</details>

### Exercise 2 — Write a test: the decoder never panics on bad input (medium)

Add two properties to the fixed `code/p23-harden`: (a) `decode` never panics for any byte vector of up to 64 bytes; (b) every strict prefix of a valid encoded frame is an `Err` (a frame that was cut in the middle must never decode as a shorter, valid frame).

<details>
<summary>Hint</summary>

For (a), the test body only calls `decode` and ignores the result: a panic fails the test. For (b), generate a payload length and a cut position; `prop::sample::Index` gives you an index that is always valid for a slice of any length: `cut.index(bytes.len())` is in `0..bytes.len()`.

</details>

<details>
<summary>Answer</summary>

Inside the existing `proptest! { … }` block:

```rust
#[test]
fn decode_never_panics(bytes in prop::collection::vec(any::<u8>(), 0..64)) {
    let _ = decode(&bytes);
}

#[test]
fn every_cut_frame_is_an_error(len in 0usize..300, cut in any::<prop::sample::Index>()) {
    let bytes = encode(&Frame { kind: 7, payload: vec![1; len] }).unwrap();
    let cut_at = cut.index(bytes.len());
    prop_assert!(decode(&bytes[..cut_at]).is_err());
}
```

```text
test tests::a_small_frame_round_trips ... ok
test tests::every_cut_frame_is_an_error ... ok
test tests::every_frame_round_trips ... ok
test tests::decode_never_panics ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Why it works: (a) is a cheap, always-running cousin of a fuzz target — it runs in every `cargo test`, while the fuzzer explores much further at night. (b) checks the property that matters most for a stream protocol: a frame that is not complete yet must be "not enough data", never a different message. It passes because `decode` requires the rest to have *exactly* the announced length.

</details>

### Exercise 3 — A supply-chain job (medium)

Add a job `supply-chain` to the workflow of Section 3 that installs `cargo-deny` and `cargo-audit` and runs `cargo deny check` and `cargo audit`. It should not wait for `gate`. Check the result with `actionlint`.

<details>
<summary>Hint</summary>

Follow the shape of the other jobs. Without a `needs:` line, a job starts at once, in parallel with `gate`. Remember the empty line between jobs and the indentation of two spaces per level.

</details>

<details>
<summary>Answer</summary>

At the end of `jobs:`:

```yaml
  supply-chain:
    name: licences, bans and advisories
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: cargo install --locked cargo-deny cargo-audit
      - run: cargo deny check
      - run: cargo audit
```

`actionlint .github/workflows/ci.yml` prints nothing and exits with 0. Why it works: supply-chain checks do not depend on your code compiling, so they run in parallel and report a new advisory even on a day when the fast lane is red. Because the workflow also runs every night, a newly published advisory for one of your dependencies shows up within a day, even if you did not push anything. (`cargo deny check` needs a `deny.toml` in the repository, as in Lesson 10.13.)

</details>

### Exercise 4 — Read a results table critically (hard)

A friend's `docs/results.md` says: "Average latency 0.8 ms, 120,000 requests per second. Measured with `oha -c 500 -z 10s` on my laptop against a debug build, client and server on the same machine, one run." List every problem with this measurement and how to fix each one.

<details>
<summary>Hint</summary>

Go through the six points of the methodology in Section 4, and remember what Lesson 19.1 said about averages and about closed-model load generators.

</details>

<details>
<summary>Answer</summary>

1. **Debug build:** numbers describe the wrong program. Use `--release`.
2. **Average only:** it hides the tail. Report p50, p99 and p99.9.
3. **Closed model (`-c 500`) without a rate:** the client waits for answers, so a slow server slows the client down and hides latency (coordinated omission, Lesson 19.1). Use a fixed rate (`-q`) and report latency at that rate.
4. **10 seconds, no warm-up, one run:** too short and possibly noise. Discard a warm-up, run 60 s or more, three times, report the median and spread.
5. **Client and server on one laptop:** they compete for CPU, and nothing says so. Separate them or state it under "Limits and honesty".
6. **No hardware, versions, payload size or commit:** nobody can repeat it. Write the setup section.
7. **No baseline and no profile:** 120,000 per second compared with what, and limited by what? Add a comparison and a flame graph.

</details>

## Quiz

1. What is a test matrix, and where do its rows come from?

<details>
<summary>Answer</summary>

A table that maps each requirement and each failure mode of the design document to the test that proves it, with a status. Its rows come from the requirements section and the failure-mode table of the design document.

</details>

2. Why did the example test not find the `as u16` bug, and how did the property test find it so precisely?

<details>
<summary>Answer</summary>

The example used a 5-byte payload, far from the 65,535-byte limit. The property test tried lengths up to 69,999, found a failing one, and shrank it to the smallest failing length, 65,536 — exactly 2¹⁶, which points straight at the 16-bit cast.

</details>

3. Why should `proptest-regressions/` be committed?

<details>
<summary>Answer</summary>

It holds the seeds of cases that failed before. Every run, on every machine and in CI, tries them first, so an old bug cannot return unnoticed.

</details>

4. What belongs in the fast lane of CI and what in the slow lane, and why split them?

<details>
<summary>Answer</summary>

Fast lane: format, Clippy, unit and integration tests, quick acceptance checks — minutes, on every push. Slow lane: simulation with many seeds, fuzzing, benchmark builds — after the fast lane and longer at night. Splitting keeps feedback fast without giving up the slow, deep checks.

</details>

5. Why does CI only *compile* the benchmarks instead of running them?

<details>
<summary>Answer</summary>

Shared CI machines are noisy: other jobs run on the same hardware, so the numbers are not reliable. Compiling them still catches benchmarks that no longer build; real measurements happen on known hardware with the written methodology.

</details>

6. Name four things a benchmark methodology must state.

<details>
<summary>Answer</summary>

Any four of: hardware and software versions, build profile, the exact workload (command, size, connections, rate, duration), warm-up, number of repetitions with median and spread, the baseline, and the limits of the setup.

</details>

7. What is headroom, and how do you find it?

<details>
<summary>Answer</summary>

The distance between the load you run at and the load at which the system first misses its goals. You find it by increasing a fixed rate step by step until the latency goal (for example p99) is missed; that rate is the capacity, and the gap to the normal load is the headroom.

</details>

## Summary

- Hardening looks at the whole system: build a test matrix from the design document's requirements and failure modes, and fill the missing rows by risk.
- Round-trip properties find bugs that examples miss; proptest shrinks failures to a minimal case (here `len = 65536`) and saves the seed in `proptest-regressions/`, which you commit.
- `as` casts silently cut lengths; use `try_from` at every boundary.
- One pipeline with a fast lane (format, Clippy, tests, acceptance checks) and a slow lane (simulation seeds, fuzzing, benchmark builds, longer at night), checked with `actionlint`.
- Trustworthy benchmarks have a written methodology: setup, workload with a fixed rate, warm-up, repetitions, a baseline, a profile and an honest list of limits.
- The hardening checklist asks about input, panics, limits, timeouts, crashes, shutdown, `unsafe`, the supply chain and visibility — and the answers go into your README.

## Sources and further reading

- [The proptest book](https://proptest-rs.github.io/proptest/intro.html) — strategies, shrinking and the regression file.
- [cargo-mutants](https://mutants.rs/) — finding the assertions your tests are missing.
- [Workflow syntax for GitHub Actions](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions) — `on.schedule`, `needs`, `timeout-minutes` and expressions.
- [actionlint](https://github.com/rhysd/actionlint) — the checks it runs on workflow files, including ShellCheck on `run:` scripts.
- [Criterion.rs user guide](https://bheisler.github.io/criterion.rs/book/) and [oha](https://github.com/hatoo/oha) — micro-benchmarks and HTTP load generation.
- [The USE Method](https://www.brendangregg.com/usemethod.html) — utilisation, saturation and errors: a checklist for explaining where a system's limit is.
- [cargo-deny](https://embarkstudios.github.io/cargo-deny/) — the checks of the supply-chain job.

---

[🏠 Start](../START-HERE.md) · [← 23.7 Capstone option 5: your own idea](07-capstone-your-own-idea.md) · [📋 Phase 23](README.md) · [23.9 Presenting your work →](09-presenting-your-work.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
