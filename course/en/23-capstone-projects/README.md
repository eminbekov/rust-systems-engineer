<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.30 Checkpoint: your specialization](../22-specializations/30-checkpoint-your-specialization.md) · [23.1 How capstone projects work →](01-how-capstone-projects-work.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/23-capstone-projects/README.md) <!-- nav -->

# Phase 23 — Capstone projects

> **Stage 5 — Mastery and career** · ⏱ about 10–14 weeks at ~10 hours per week

## What this phase is about

Every project so far was an exercise: the lesson gave you the contract, and each project practised the ideas of one phase. A *capstone project* is different. It is a complete system that you design, build, test, measure, document and present yourself, and it uses almost everything you learned in this course at once: storage, networking, concurrency, distributed systems, performance, reliability and operations.

In this phase you build **two** capstones, one after the other, each in 4–7 weeks of about 10 hours. One should go deep into the specialization you chose in Phase 22; the other should show range in a different area. You choose from four reference options — a distributed key-value store, an HTTP (HyperText Transfer Protocol) reverse proxy, a message broker and an observability pipeline — or bring your own idea and scope it with the method of Lesson 23.7.

The lessons do not give solutions. They give what a senior engineer would give you at work: a reference architecture, a contract, milestones with "Done when" checks, a quality bar and the real systems to study. Around them, the phase teaches the work that turns code into a portfolio project: a design document before the code, a test matrix and a CI (Continuous Integration) pipeline during hardening, honest benchmarks, and a README, demo and talk that other engineers remember.

## What you will be able to do

- [ ] Choose and scope a capstone with a pitch, non-goals, a cut list and a spike for the biggest risk.
- [ ] Write a design document with checkable requirements, estimates, a failure-mode table, alternatives and ADRs (Architecture Decision Records).
- [ ] Set up a capstone repository with a walking skeleton, acceptance tests per milestone and Git tags.
- [ ] Build a complete system milestone by milestone against a contract, with contract tests or a reference model.
- [ ] Harden it: a test matrix, property-based and failure tests, fuzzing, and one CI pipeline with fast and slow lanes.
- [ ] Measure it with a written methodology and report percentiles, baselines and limits honestly.
- [ ] Present it with a README that is tested, a write-up, a two-minute demo and a ten-minute talk, and publish a release.
- [ ] Review a capstone with a rubric and a fresh-clone test, and fix what the review finds.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 23.1 | [How capstone projects work](01-how-capstone-projects-work.md) | 135 min |
| 23.2 | [Writing a design document](02-writing-a-design-document.md) | 120 min |
| 23.3 | [Capstone option 1: a distributed key-value store](03-capstone-distributed-key-value-store.md) | 50 h |
| 23.4 | [Capstone option 2: a high-performance HTTP reverse proxy](04-capstone-http-reverse-proxy.md) | 50 h |
| 23.5 | [Capstone option 3: a message broker](05-capstone-message-broker.md) | 50 h |
| 23.6 | [Capstone option 4: an observability pipeline](06-capstone-observability-pipeline.md) | 50 h |
| 23.7 | [Capstone option 5: your own idea](07-capstone-your-own-idea.md) | 50 h |
| 23.8 | [Testing, benchmarking and hardening your capstone](08-testing-benchmarking-and-hardening-capstones.md) | 120 min |
| 23.9 | [Presenting your work](09-presenting-your-work.md) | 90 min |
| 23.10 | [Checkpoint: capstone review](10-checkpoint-capstone-review.md) | 90 min |

(The times are for reading and doing the lesson itself. The capstones are built over the weeks between the lessons.)

## Projects in this phase

- **Practice packages in the lessons:** `code/p23-skeleton` (the walking skeleton every capstone starts from), `code/p23-estimate` (tested back-of-the-envelope estimates), and the small contract packages of the option lessons (`code/p23-kv-contract`, `code/p23-proxy-config`, `code/p23-broker-model`, `code/p23-pipe-model`), plus `code/p23-harden` and `code/p23-present`.
- **Two capstones**, each in its own Git repository under `code/p23-<name>`, chosen from:
  - [Option 1 — a distributed key-value store](03-capstone-distributed-key-value-store.md) (`code/p23-kv`), growing out of Projects 17.14 and 18.14;
  - [Option 2 — an HTTP reverse proxy](04-capstone-http-reverse-proxy.md) (`code/p23-proxy`), growing out of Project 15.15;
  - [Option 3 — a message broker](05-capstone-message-broker.md) (`code/p23-broker`);
  - [Option 4 — an observability pipeline](06-capstone-observability-pipeline.md) (`code/p23-pipe`);
  - [Option 5 — your own idea](07-capstone-your-own-idea.md), scoped with a one-page proposal.

## How to study this phase

1. **Read 23.1–23.7 first, then choose.** Skim all five options before you decide; write your choice and the order in a short note. Start the first capstone only after its design document is written.
2. **Interleave the method lessons with the work.** Read 23.8 when your first capstone reaches about M3, and 23.9 when its milestones are almost done. Do 23.10 at the end of each capstone, not only at the end of the phase.
3. **Turn every "Done when" into a test before the milestone starts.** A milestone is done when CI is green and the commit is tagged — not when it works on your laptop.
4. **Keep a work log and protect the quality bar.** When you are late, cut scope with your cut list; never cut tests, measurements or documentation.
5. **Clean up as you go.** Remove `course-` containers and `kind` clusters after experiments, run `cargo clean` in practice packages, and keep at most one cluster or database running at a time.

---

[🏠 Start](../START-HERE.md) · [← 22.30 Checkpoint: your specialization](../22-specializations/30-checkpoint-your-specialization.md) · [23.1 How capstone projects work →](01-how-capstone-projects-work.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
