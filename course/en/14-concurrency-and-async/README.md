<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 13.18 Checkpoint](../13-operating-systems/18-checkpoint.md) · [14.1 Why concurrency is hard: races and deadlocks →](01-why-concurrency-is-hard.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/README.md) <!-- nav -->

# Phase 14 — Concurrency and async in depth

> **Stage 3 — How computers really work** · ⏱ about 4–5 weeks at ~10 hours per week

## What this phase is about

In Phase 05 you started threads, sent messages through channels and shared a counter with `Arc<Mutex<T>>`. In Phase 06 you wrote your first `async` functions with Tokio and built a thread-pool web server. Those tools work, but so far you have used them as black boxes. This phase opens the boxes. You will see why concurrent code is hard even when the compiler accepts it, how locks, condition variables and atomics really work, and how an async runtime turns a `Future` into running code.

The first half is about threads and shared memory. You meet race conditions that safe Rust cannot prevent, deadlocks and how to avoid them, the costs of a contended lock, condition variables, thread pools and data parallelism with `rayon`. Then you go one level deeper: atomic operations, memory ordering on your weakly ordered arm64 Mac, a spin lock and lock-free data structures built by hand, and the `loom` model checker that tests concurrent code under every possible interleaving.

The second half is about async. You learn what `Future`, `poll` and `Waker` really are, build a tiny async runtime of your own, understand `Pin`, and then use Tokio in depth: tasks, channels, cancellation, timeouts and graceful shutdown. The phase ends with a concurrent job queue project and the checkpoint that closes Stage 3.

Every example is measured or tested on a real machine, and many of them show a bug first. That is on purpose: in concurrency, the most important skill is to see why "it worked when I tried it" is not proof.

## What you will be able to do

- [ ] Explain the difference between a data race and a race condition, and find and fix lost updates, check-then-act bugs and deadlocks.
- [ ] Choose between `Mutex`, `RwLock`, condition variables, channels, atomics and `rayon`, and measure the choice instead of guessing.
- [ ] Use atomic operations and compare-and-swap loops, and pick the correct memory ordering (`Relaxed`, `Acquire`/`Release`, `SeqCst`) with a reason.
- [ ] Build a spin lock and a lock-free ring buffer with `unsafe` code, justify every `unsafe` block, and check them with Miri and loom.
- [ ] Explain how `Future`, `poll`, `Waker` and `Pin` work, and write a small executor that runs futures and timers.
- [ ] Use Tokio tasks and channels correctly, with timeouts, cancellation, backpressure and graceful shutdown.
- [ ] Build and test a concurrent job queue with workers, retries and clean shutdown.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 14.1 | [Why concurrency is hard: races and deadlocks](01-why-concurrency-is-hard.md) | 135 min |
| 14.2 | [Mutexes and read-write locks in depth](02-mutexes-and-rwlocks-in-depth.md) | 135 min |
| 14.3 | [Condition variables](03-condition-variables.md) | 2.5 h |
| 14.4 | [Thread pools, scoped threads and rayon](04-thread-pools-scoped-threads-and-rayon.md) | 2.5 h |
| 14.5 | [Atomics](05-atomics.md) | 2.5 h |
| 14.6 | [Memory ordering](06-memory-ordering.md) | 135 min |
| 14.7 | [Building a spin lock](07-building-a-spin-lock.md) | 135 min |
| 14.8 | [Lock-free data structures](08-lock-free-data-structures.md) | 2.5 h |
| 14.9 | [Testing concurrent code with loom](09-testing-concurrent-code-with-loom.md) | 3 h |
| 14.10 | [How futures work: Future, poll and Waker](10-how-futures-work.md) | 2.5 h |
| 14.11 | [Build a tiny async runtime](11-build-a-tiny-async-runtime.md) | 135 min |
| 14.12 | [Pin and self-referential futures](12-pin-and-self-referential-futures.md) | 120 min |
| 14.13 | [Tokio in depth: tasks and channels](13-tokio-tasks-and-channels.md) | 3 h |
| 14.14 | [Cancellation, timeouts and graceful shutdown](14-cancellation-timeouts-and-graceful-shutdown.md) | 2.5 h |
| 14.15 | [Project: a concurrent job queue](15-project-concurrent-job-queue.md) | 8 h |
| 14.16 | [Checkpoint: Stage 3 complete](16-checkpoint-stage-3-complete.md) | 105 min |

## Projects in this phase

- **The phase package `code/p14-concurrency`** (Lessons 14.1 to 14.8): one package with a small program in `src/bin/` for every example, and a library part (`src/lib.rs`) where you build a spin lock, a ticket lock, a lock-free ring buffer and a lock-free stack, each with tests that also run under Miri.
- **Small packages in the async lessons** (`code/p14-loom`, `code/p14-futures`, `code/p14-tiny-runtime`, `code/p14-pin`, `code/p14-tokio`, `code/p14-shutdown`): each lesson has its own follow-along package with tests.
- **[Project 14.15 — A concurrent job queue](15-project-concurrent-job-queue.md)**: a Tokio job queue with a bounded channel (backpressure), several workers, retries, cancellation and graceful shutdown, built milestone by milestone with acceptance tests (`code/p14-jobqueue`).

## How to study this phase

1. **Run every example several times, in debug and in release mode.** Concurrency bugs depend on timing. A result that changes from run to run is the lesson, not a mistake in the course. Write down your own numbers next to the ones in the text.
2. **Break the code on purpose.** Change `Acquire` to `Relaxed`, remove a lock, swap the order of two locks, and watch the test, Miri or loom fail. A safety rule that you have seen fail is a rule you understand.
3. **Test invariants, not timing.** Every concurrent test in this phase starts many threads or tasks, waits for all of them, and then checks something that must always be true. Keep that shape in your own tests.
4. **Draw before you code.** For locks and orderings, draw the two threads as columns with arrows for "happens-before". Most ordering bugs are visible on paper.
5. **Prefer the simple tool.** After building your own primitives, use `Mutex`, channels, `rayon`, `crossbeam` and Tokio in real code. Write your own lock-free code only to learn, or when a measurement proves you need it.

---

[🏠 Start](../START-HERE.md) · [← 13.18 Checkpoint](../13-operating-systems/18-checkpoint.md) · [14.1 Why concurrency is hard: races and deadlocks →](01-why-concurrency-is-hard.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
