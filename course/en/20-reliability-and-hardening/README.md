<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 19.11 Checkpoint](../19-performance-and-profiling/11-checkpoint.md) · [20.1 What reliability means →](01-what-reliability-means.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/20-reliability-and-hardening/README.md) <!-- nav -->

# Phase 20 — Reliability and hardening

> **Stage 5 — Mastery and career** · ⏱ about 3 weeks at ~10 hours per week

## What this phase is about

A program that works on your laptop is only the beginning. In production, disks fill up, networks lose packets, dependencies become slow, inputs are strange or hostile, and bugs that no test found finally run. *Reliability* is the art of keeping these faults from turning into failures that users see, and of making the failures that do happen small, loud and short. *Hardening* is the work of removing, one by one, the ways in which a program can fail or be attacked.

The phase starts with the language of reliability: faults and failures, availability in "nines", MTBF and MTTR (mean time between failures and mean time to recovery), and the four layers *prevent, detect, contain, recover*. Then you learn how a Rust service should treat errors and panics in production. The middle of the phase is about finding bugs before your users do, with tools that go far beyond normal tests: model-based fuzzing, Miri, AddressSanitizer, ThreadSanitizer and loom in CI (continuous integration), fault injection and chaos experiments with toxiproxy, and formal proofs with the model checker Kani.

The last part looks at the whole service: the supply chain of your dependencies and builds, overload and backpressure, and a production-readiness checklist that collects everything into questions you can ask of any service. In the project you take a real service and harden it step by step.

Every lesson uses a small package in `code/p20-<topic>`, shows real output from real commands, and plants a bug for the tools to find. Many tools need the nightly toolchain or Docker; each lesson says exactly what to install.

## What you will be able to do

- [ ] Explain reliability with faults, failures, failure modes and availability numbers, and compute the availability of chains and redundant copies.
- [ ] Decide what is an error and what is a bug, isolate panics in threads and tasks, report them with a panic hook, and choose between `unwind` and `abort`.
- [ ] Write model-based, stateful fuzz targets with a hand-written `Arbitrary`, manage the corpus, and set up continuous fuzzing in CI.
- [ ] Check `unsafe` and concurrent code with Miri, AddressSanitizer, ThreadSanitizer and loom, and run all of them in one CI workflow.
- [ ] Inject faults inside the code and on the network with toxiproxy, and write a repeatable chaos experiment with a stop button.
- [ ] Prove small, critical functions correct for all inputs with Kani, and turn counterexamples into regression tests.
- [ ] Audit and secure the supply chain of a Rust project, protect a service against overload, and review it with a production-readiness checklist.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 20.1 | [What reliability means](01-what-reliability-means.md) | 120 min |
| 20.2 | [Errors and panics in production](02-errors-and-panics-in-production.md) | 135 min |
| 20.3 | [Fuzzing in depth](03-fuzzing-in-depth.md) | 135 min |
| 20.4 | [Miri, sanitizers and loom in CI](04-miri-sanitizers-and-loom-in-ci.md) | 2.5 h |
| 20.5 | [Fault injection and chaos testing](05-fault-injection-and-chaos-testing.md) | 135 min |
| 20.6 | [Formal verification basics with Kani](06-formal-verification-with-kani.md) | 135 min |
| 20.7 | [Supply-chain security in depth](07-supply-chain-security-in-depth.md) | 135 min |
| 20.8 | [Overload, limits and backpressure](08-overload-limits-and-backpressure.md) | 120 min |
| 20.9 | [The production-readiness checklist](09-production-readiness-checklist.md) | 2.5 h |
| 20.10 | [Project: harden a service](10-project-harden-a-service.md) | 10 h |
| 20.11 | [Checkpoint](11-checkpoint.md) | 105 min |

## Projects in this phase

- **Small packages in the lessons** (`code/p20-reliability`, `code/p20-panics`, `code/p20-fuzz`, `code/p20-checks`, `code/p20-chaos`, `code/p20-kani`, and more): each lesson plants a bug, lets a tool find it, and ends with exercises that ask you to write a test or a proof.
- **[Project 20.10 — Harden a service](10-project-harden-a-service.md)**: take the URL (Uniform Resource Locator, a web address) shortener from Project 16.15 (or the small service given in the lesson) and add timeouts and limits, fuzz targets, Miri where it applies, supply-chain checks, a chaos test and the readiness checklist.

## How to study this phase

1. **Let the tool find the bug first.** Every lesson plants a bug. Run the tool *before* you read the explanation, and try to read its report yourself. Reading fuzzer, sanitizer and Kani reports is the real skill of this phase.
2. **Keep every finding as a test.** Whatever a fuzzer, Miri or Kani finds becomes a normal unit test. After this phase your test suites should contain tests named after bugs that tools found.
3. **Time-box the slow tools.** Fuzzers never finish, Miri and sanitizers are slow, and Kani grows quickly with its bounds. Use `-max_total_time`, `cfg(miri)` and small `unwind` values, and let CI do the long runs.
4. **Clean up after experiments.** Remove Docker containers (`docker rm -f course-…`), toxics and build folders (`cargo clean`) when you finish a lesson. A fault left behind is a fault for the next person.
5. **Apply it to your own code.** After each lesson, pick one earlier project, for example the key-value store from Phase 17, and add one fuzz target, one Miri run or one Kani proof to it.

---

[🏠 Start](../START-HERE.md) · [← 19.11 Checkpoint](../19-performance-and-profiling/11-checkpoint.md) · [20.1 What reliability means →](01-what-reliability-means.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
