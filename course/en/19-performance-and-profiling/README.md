<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 18.15 Checkpoint: Stage 4 complete](../18-distributed-systems/15-checkpoint-stage-4-complete.md) · [19.1 Latency, throughput and percentiles →](01-latency-throughput-and-percentiles.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/19-performance-and-profiling/README.md) <!-- nav -->

# Phase 19 — Performance and profiling

> **Stage 5 — Mastery and career** · ⏱ about 3 weeks at ~10 hours per week

## What this phase is about

A correct program is not always a fast enough program. A service that answers in 2 ms most of the time but in 2 seconds for one user in a hundred feels broken; a tool that needs an hour where it could need a minute wastes everyone's day. In this phase you learn to make Rust programs faster in the way professionals do it: by measuring, never by guessing.

You start with the numbers that describe speed: latency, throughput and percentiles, and you put real load on a web service to see where it breaks. Then you learn to find *where* the time goes with profilers: `samply` and flame graphs on your Mac, `perf` on Linux, and the heap profiler `dhat` for memory. With those tools you fix real hot spots: fewer allocations, a better data layout, a faster hasher.

The second half goes closer to the hardware and the build: SIMD (Single Instruction, Multiple Data) instructions that work on many numbers at once, compiler settings such as LTO (link-time optimization) and PGO (profile-guided optimization), and the performance of async and I/O code. At the end you take a slow but correct program and make it at least ten times faster without changing a single byte of its output.

## What you will be able to do

- [ ] Measure latency and throughput honestly, report percentiles instead of averages, and avoid coordinated omission.
- [ ] Load-test a service with `oha`, find its saturation point and its capacity for a latency goal.
- [ ] Profile a program with `samply` on macOS and `perf` on Linux, and read call trees and flame graphs.
- [ ] Measure heap allocations and peak memory with `dhat`, and write tests that fail when code allocates too much.
- [ ] Choose data layouts, maps and hashers for speed, and protect those choices with tests and compile-time checks.
- [ ] Use auto-vectorization and NEON intrinsics, and tune builds with profiles, LTO, `target-cpu` and PGO.
- [ ] Find and fix slow I/O and blocking code in async programs.
- [ ] Take a slow program through the full loop (freeze the output, measure, profile, change, test, measure) to a 10× speed-up.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 19.1 | [Latency, throughput and percentiles](01-latency-throughput-and-percentiles.md) | 135 min |
| 19.2 | [Load testing services](02-load-testing.md) | 120 min |
| 19.3 | [Profilers and flame graphs](03-profilers-and-flame-graphs.md) | 135 min |
| 19.4 | [Profiling on Linux with perf](04-profiling-on-linux-with-perf.md) | 90 min |
| 19.5 | [Memory profiling and fewer allocations](05-memory-profiling-and-allocations.md) | 105 min |
| 19.6 | [Data layout for speed](06-data-layout-for-speed.md) | 135 min |
| 19.7 | [SIMD: one instruction, many data](07-simd.md) | 135 min |
| 19.8 | [Build settings for speed: profiles, LTO and PGO](08-build-settings-for-speed.md) | 2.5 h |
| 19.9 | [Performance of async and I/O code](09-async-and-io-performance.md) | 135 min |
| 19.10 | [Project: make a slow program fast](10-project-make-a-slow-program-fast.md) | 8 h |
| 19.11 | [Checkpoint](11-checkpoint.md) | 105 min |

## Projects in this phase

- `code/p19-latency`: percentiles, histograms and a coordinated-omission simulation (Lesson 19.1).
- `code/p19-load`: a small `axum` service with fast, slow and blocking endpoints, loaded with `oha` (Lesson 19.2).
- `code/p19-wordfreq`: a word counter that you profile on macOS and Linux and speed up step by step (Lessons 19.3 to 19.6).
- `code/p19-layout`, `code/p19-simd`, `code/p19-build`, `code/p19-io`: measured experiments for data layout, SIMD, build settings and I/O (Lessons 19.6 to 19.9).
- **Project 19.10 — make a slow program fast:** a log analyzer `p19-logstat` that must become at least 10 times faster with byte-for-byte identical output.

## How to study this phase

- Always build in release mode for measurements, and use the `profiling` profile (release plus debug info) for profilers. Debug builds give misleading numbers.
- Before every optimization, have a test that pins down the current output, and keep the old version to compare with. A faster program with a different answer is a bug.
- Change one thing at a time and measure it with `hyperfine` or `criterion`. Write the numbers into your commit messages or a notes file; your Mac's numbers will differ from the lessons, but the ratios should be similar.
- Close heavy programs while you measure, run every measurement more than once, and ignore differences smaller than the noise.
- Linux lessons (`perf`, `strace`) run in the VM (virtual machine) `dev` or a local Docker container. Load-test and profile only your own programs.

---

[🏠 Start](../START-HERE.md) · [← 18.15 Checkpoint: Stage 4 complete](../18-distributed-systems/15-checkpoint-stage-4-complete.md) · [19.1 Latency, throughput and percentiles →](01-latency-throughput-and-percentiles.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
