<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 14.14 Cancellation, timeouts and graceful shutdown](14-cancellation-timeouts-and-graceful-shutdown.md) · [📋 Phase 14](README.md) · [14.16 Checkpoint: Stage 3 complete →](16-checkpoint-stage-3-complete.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/14-concurrency-and-async/15-project-concurrent-job-queue.md) <!-- nav -->

# Lesson 14.15 — Project: a concurrent job queue

> **Phase 14 — Concurrency and async in depth** · Lesson 15 of 16 · ⏱ about 8 hours (4 sittings)

## What you will learn

- How to combine everything from this phase in one library: a bounded channel, worker tasks, retries, a cancellation token and a graceful shutdown.
- How to design an async API (Application Programming Interface: the types and functions your library offers) around a trait (`Job`) and give it a clear contract.
- How to work against acceptance tests that use a paused clock, so that timing is checked exactly and the tests never flake.
- How to grow a concurrent program in small milestones, one commit each.

## Before you start

- You finished [Lesson 14.14 — Cancellation, timeouts and graceful shutdown](14-cancellation-timeouts-and-graceful-shutdown.md). The `service` program there is the closest relative of this project.
- You know bounded `mpsc` channels and backpressure from [Lesson 14.13](13-tokio-tasks-and-channels.md), and the "lock released at `;`" rule from [Lesson 14.11](11-build-a-tiny-async-runtime.md).
- You know integration tests in `tests/` and shared helpers in `tests/common/mod.rs` from [Lesson 05.6 — Testing in depth](../05-rust-intermediate/06-testing-in-depth.md).
- Your project goes in `~/projects/rust/code/p14-jobqueue`. Work on a branch: `git switch -c p14-jobqueue`.

## New words

| Word | Meaning |
|---|---|
| **job queue** | A program part that accepts units of work (jobs), keeps them in order, and lets workers run them. Real examples: sending emails, resizing images, building reports. |
| **worker** | A task that takes jobs from the queue and runs them, one after another. |
| **retry** | Running a failed job again. Only *retryable* errors (for example a timeout) are retried; *fatal* errors (bad input) are not. |
| **exponential backoff** | Waiting longer before each new retry: 100 ms, then 200 ms, then 400 ms. It gives a struggling service time to recover. |
| **grace period** | The time a shutdown waits for work to finish normally, before it cancels what is left. |
| **acceptance test** | A test that comes with the task and decides whether a milestone is done. |
| **RPITIT** (Return-Position `impl Trait` In Traits) | Writing `-> impl Future<…>` as the return type of a trait method. It lets a trait method be async without boxing. |

## 1. How the queue works

```text
 submit(job) ──► [ bounded channel, capacity N ] ──► worker 1 ─┐
 (waits when full:                    │           ──► worker 2 ─┼─► Report { succeeded,
  backpressure)                       │           ──► worker 3 ─┘            failed,
                                      │                 ▲                    cancelled,
 try_submit(job) ──► Err(Full) when full                │                    retries }
                                                        │
 shutdown(grace) ── 1. close the channel                │
                    2. wait up to `grace` for workers ──┘
                    3. then cancel the token: running jobs and queued jobs are cancelled
```

The pieces, and the lesson where you learned each one:

1. **A bounded `mpsc` channel** holds waiting jobs. `submit` waits while it is full (backpressure, Lesson 14.13). `try_submit` never waits.
2. **N worker tasks** share the one receiver through `Arc<tokio::sync::Mutex<Receiver>>`, as in the `service` of Lesson 14.14.
3. **Retries**: each job gets up to `max_attempts` attempts, with a doubling wait between them. The wait must be cancellable.
4. **A `CancellationToken`** (Lesson 14.14) is shared by all workers and passed to every job.
5. **`shutdown(grace)`** drops the sender, so the workers finish the queued jobs and stop. If that takes longer than `grace`, it cancels the token; the workers then stop the running jobs and count the queued ones as cancelled without starting them.
6. **A `Report`** counts what happened to every accepted job.

## The project

### What you will build

A library crate `p14-jobqueue` with a generic `JobQueue<J: Job>`, the acceptance tests in `tests/`, a small demo program in `examples/demo.rs`, and a short `README.md`. No full solution is given; each milestone has hints.

### Contract

**`Cargo.toml` dependencies:**

```bash
cargo new --lib code/p14-jobqueue
cd code/p14-jobqueue
cargo add tokio@1 --features rt-multi-thread,macros,sync,time,signal
cargo add tokio-util@0.7
cargo add --dev tokio@1 --features test-util
```

**Public API.** Start `src/lib.rs` with exactly these types and signatures. The `todo!()` bodies and the placeholder field are yours to replace; the public names, fields and signatures must stay as they are, because the tests use them. This file compiles as it is:

```rust
//! A concurrent job queue on Tokio: bounded queue, N workers, retries,
//! cancellation and graceful shutdown.

use std::future::Future;
use std::marker::PhantomData;
use std::time::Duration;
use tokio_util::sync::CancellationToken;

/// How the queue behaves.
#[derive(Debug, Clone)]
pub struct Config {
    /// Number of worker tasks (at least 1).
    pub workers: usize,
    /// How many jobs may wait in the queue (at least 1).
    pub capacity: usize,
    /// Attempts per job, the first one included (at least 1).
    pub max_attempts: u32,
    /// Wait before the 2nd attempt; it doubles before each further attempt.
    pub backoff: Duration,
}

/// Why one attempt of a job failed.
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum JobError {
    /// Try again later (for example a timeout).
    Retryable(String),
    /// Do not try again (for example invalid input).
    Fatal(String),
}

/// A unit of work.
pub trait Job: Send + Sync + 'static {
    /// Runs one attempt. `attempt` starts at 1. A long job should watch `cancel`.
    fn run(
        &self,
        attempt: u32,
        cancel: CancellationToken,
    ) -> impl Future<Output = Result<(), JobError>> + Send;
}

/// The number given to each accepted job: 1, 2, 3, …
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash, PartialOrd, Ord)]
pub struct JobId(pub u64);

/// Why a job was not accepted. The job is given back.
pub enum SubmitError<J> {
    /// The queue is full (only from `try_submit`).
    Full(J),
    /// The queue does not accept jobs any more.
    Closed(J),
}

// Written by hand, so that `J` does not have to implement `Debug`.
impl<J> std::fmt::Debug for SubmitError<J> {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            SubmitError::Full(_) => write!(f, "Full(..)"),
            SubmitError::Closed(_) => write!(f, "Closed(..)"),
        }
    }
}

/// What happened to all accepted jobs, returned by `shutdown`.
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq)]
pub struct Report {
    pub succeeded: u64,
    pub failed: u64,
    pub cancelled: u64,
    /// Extra attempts, over all jobs (a job that succeeded on attempt 3 adds 2).
    pub retries: u64,
}

pub struct JobQueue<J: Job> {
    // Replace this with your own fields.
    _jobs: PhantomData<J>,
}

impl<J: Job> JobQueue<J> {
    /// Starts the workers. Must be called inside a Tokio runtime.
    pub fn start(config: Config) -> Self {
        let _ = config;
        todo!()
    }

    /// Adds a job. Waits while the queue is full (backpressure).
    pub async fn submit(&self, job: J) -> Result<JobId, SubmitError<J>> {
        let _ = job;
        todo!()
    }

    /// Adds a job if there is room right now.
    pub fn try_submit(&self, job: J) -> Result<JobId, SubmitError<J>> {
        let _ = job;
        todo!()
    }

    /// Stops accepting jobs, lets the workers finish the queue for at most
    /// `grace`, then cancels whatever is left, and reports.
    pub async fn shutdown(self, grace: Duration) -> Report {
        let _ = grace;
        todo!()
    }
}
```

**Behaviour rules.** The tests check these:

1. `JobQueue::start` spawns `workers` worker tasks at once. It must be called inside a Tokio runtime.
2. Accepted jobs get the ids `JobId(1)`, `JobId(2)`, … in the order they were accepted.
3. `submit` waits while the queue holds `capacity` jobs. `try_submit` returns `Err(SubmitError::Full(job))` instead, giving the job back.
4. Each worker runs one job at a time. Up to `workers` jobs run at the same time.
5. A job is run at most `max_attempts` times. `Ok(())` counts as `succeeded`. `Err(Fatal)` counts as `failed` at once. `Err(Retryable)` is retried after a wait of `backoff`, then `2 × backoff`, then `4 × backoff`, …; each retry adds 1 to `retries` at the moment it is decided, **before** the wait (so a job that is cancelled during a backoff wait still counts that retry). When the last attempt fails, the job counts as `failed`.
6. `shutdown(grace)` accepts no more jobs and returns the `Report` only after every worker has stopped.
7. If the queue is drained before `grace` ends, `shutdown` returns as soon as the last job is done, with nothing cancelled.
8. When `grace` ends, the shared token is cancelled. A running attempt, or a backoff wait, then ends at once and the job counts as `cancelled`; it is not retried. Jobs still in the queue count as `cancelled` and are never started. If an attempt and the cancellation finish at the same moment, cancellation wins.
9. Every accepted job is counted exactly once in `succeeded + failed + cancelled`.

**Acceptance tests.** Copy these files into `tests/`. First, a test job with four behaviours, in `tests/common/mod.rs` (`#![allow(dead_code)]` is there because each test file uses only some of the helpers):

```rust
//! A job type for the acceptance tests.
#![allow(dead_code)]

use p14_jobqueue::{Config, Job, JobError};
use std::future::Future;
use std::sync::Arc;
use std::sync::atomic::{AtomicU32, Ordering};
use std::time::Duration;
use tokio_util::sync::CancellationToken;

#[derive(Debug, Clone)]
pub enum Kind {
    /// Sleeps for the duration, then succeeds.
    Work(Duration),
    /// Fails with `Retryable` on the first `failures` attempts, then succeeds.
    FailThenSucceed { failures: u32 },
    /// Fails with `Fatal`.
    Fatal,
    /// Never finishes by itself; returns only when cancelled.
    Endless,
}

#[derive(Debug)]
pub struct TestJob {
    pub kind: Kind,
    /// How many attempts were started.
    pub runs: Arc<AtomicU32>,
}

impl TestJob {
    pub fn new(kind: Kind) -> Self {
        TestJob {
            kind,
            runs: Arc::new(AtomicU32::new(0)),
        }
    }

    pub fn work(ms: u64) -> Self {
        TestJob::new(Kind::Work(Duration::from_millis(ms)))
    }

    pub fn runs(&self) -> u32 {
        self.runs.load(Ordering::SeqCst)
    }
}

impl Job for TestJob {
    fn run(
        &self,
        attempt: u32,
        cancel: CancellationToken,
    ) -> impl Future<Output = Result<(), JobError>> + Send {
        self.runs.fetch_add(1, Ordering::SeqCst);
        let kind = self.kind.clone();
        async move {
            match kind {
                Kind::Work(duration) => {
                    tokio::time::sleep(duration).await;
                    Ok(())
                }
                Kind::FailThenSucceed { failures } if attempt <= failures => {
                    Err(JobError::Retryable(format!("attempt {attempt} failed")))
                }
                Kind::FailThenSucceed { .. } => Ok(()),
                Kind::Fatal => Err(JobError::Fatal("bad input".to_string())),
                Kind::Endless => {
                    cancel.cancelled().await;
                    Err(JobError::Retryable("cancelled".to_string()))
                }
            }
        }
    }
}

/// A configuration for tests: no retries, no backoff.
pub fn config(workers: usize, capacity: usize) -> Config {
    Config {
        workers,
        capacity,
        max_attempts: 1,
        backoff: Duration::ZERO,
    }
}

/// A grace period so long that it never ends in a test.
pub const FOREVER: Duration = Duration::from_secs(3600);
```

`tests/m1_queue.rs`:

```rust
mod common;

use common::{FOREVER, TestJob, config};
use p14_jobqueue::{JobId, JobQueue, Report, SubmitError};
use std::time::Duration;
use tokio::time::Instant;

#[tokio::test(start_paused = true)]
async fn m1_runs_every_submitted_job() {
    let queue = JobQueue::start(config(1, 4));
    for expected in 1..=3 {
        let id = queue.submit(TestJob::work(10)).await.unwrap();
        assert_eq!(id, JobId(expected));
    }
    let report = queue.shutdown(FOREVER).await;
    assert_eq!(
        report,
        Report {
            succeeded: 3,
            ..Report::default()
        }
    );
}

#[tokio::test(start_paused = true)]
async fn m1_submit_waits_while_the_queue_is_full() {
    let queue = JobQueue::start(config(1, 1));
    let start = Instant::now();
    queue.submit(TestJob::work(100)).await.unwrap(); // the worker takes it
    queue.submit(TestJob::work(100)).await.unwrap(); // waits in the queue
    queue.submit(TestJob::work(100)).await.unwrap(); // must wait for a free place
    assert_eq!(start.elapsed(), Duration::from_millis(100));
    queue.shutdown(FOREVER).await;
}

#[tokio::test(start_paused = true)]
async fn m1_try_submit_reports_a_full_queue() {
    let queue = JobQueue::start(config(1, 1));
    queue.submit(TestJob::work(50)).await.unwrap();
    tokio::time::sleep(Duration::from_millis(1)).await; // the worker takes job 1
    assert_eq!(queue.try_submit(TestJob::work(50)).unwrap(), JobId(2));
    match queue.try_submit(TestJob::work(50)) {
        Err(SubmitError::Full(_job)) => {} // the job is given back
        other => panic!("expected Full, got {:?}", other.map(|_| ())),
    }
    let report = queue.shutdown(FOREVER).await;
    assert_eq!(report.succeeded, 2);
}
```

`tests/m2_workers.rs`:

```rust
mod common;

use common::{FOREVER, TestJob, config};
use p14_jobqueue::JobQueue;
use std::time::Duration;
use tokio::time::Instant;

#[tokio::test(start_paused = true)]
async fn m2_workers_run_jobs_in_parallel() {
    let queue = JobQueue::start(config(4, 8));
    let start = Instant::now();
    for _ in 0..8 {
        queue.submit(TestJob::work(100)).await.unwrap();
    }
    let report = queue.shutdown(FOREVER).await;
    assert_eq!(report.succeeded, 8);
    // 8 jobs of 100 ms on 4 workers: two rounds.
    assert_eq!(start.elapsed(), Duration::from_millis(200));
}

#[tokio::test(start_paused = true)]
async fn m2_one_worker_runs_jobs_one_after_another() {
    let queue = JobQueue::start(config(1, 8));
    let start = Instant::now();
    for _ in 0..3 {
        queue.submit(TestJob::work(100)).await.unwrap();
    }
    queue.shutdown(FOREVER).await;
    assert_eq!(start.elapsed(), Duration::from_millis(300));
}
```

`tests/m3_retries.rs`:

```rust
mod common;

use common::{FOREVER, Kind, TestJob};
use p14_jobqueue::{Config, JobQueue, Report};
use std::sync::Arc;
use std::time::Duration;
use tokio::time::Instant;

fn retry_config() -> Config {
    Config {
        workers: 1,
        capacity: 4,
        max_attempts: 3,
        backoff: Duration::from_millis(100),
    }
}

#[tokio::test(start_paused = true)]
async fn m3_retries_with_exponential_backoff() {
    let queue = JobQueue::start(retry_config());
    let job = TestJob::new(Kind::FailThenSucceed { failures: 2 });
    let runs = Arc::clone(&job.runs);
    let start = Instant::now();
    queue.submit(job).await.unwrap();
    let report = queue.shutdown(FOREVER).await;
    assert_eq!(
        report,
        Report {
            succeeded: 1,
            retries: 2,
            ..Report::default()
        }
    );
    assert_eq!(runs.load(std::sync::atomic::Ordering::SeqCst), 3);
    // Waits of 100 ms and then 200 ms before attempts 2 and 3.
    assert_eq!(start.elapsed(), Duration::from_millis(300));
}

#[tokio::test(start_paused = true)]
async fn m3_fatal_errors_are_not_retried() {
    let queue = JobQueue::start(retry_config());
    let job = TestJob::new(Kind::Fatal);
    let runs = Arc::clone(&job.runs);
    queue.submit(job).await.unwrap();
    let report = queue.shutdown(FOREVER).await;
    assert_eq!(report.failed, 1);
    assert_eq!(report.retries, 0);
    assert_eq!(runs.load(std::sync::atomic::Ordering::SeqCst), 1);
}

#[tokio::test(start_paused = true)]
async fn m3_gives_up_after_max_attempts() {
    let queue = JobQueue::start(retry_config());
    let job = TestJob::new(Kind::FailThenSucceed { failures: 10 });
    let runs = Arc::clone(&job.runs);
    queue.submit(job).await.unwrap();
    let report = queue.shutdown(FOREVER).await;
    assert_eq!(
        report,
        Report {
            failed: 1,
            retries: 2,
            ..Report::default()
        }
    );
    assert_eq!(runs.load(std::sync::atomic::Ordering::SeqCst), 3);
}
```

`tests/m4_shutdown.rs`:

```rust
mod common;

use common::{Kind, TestJob, config};
use p14_jobqueue::{Config, JobQueue, Report};
use std::time::Duration;
use tokio::time::{Instant, timeout};

/// Fails the test (instead of hanging) if `shutdown` never returns.
const HANG_LIMIT: Duration = Duration::from_secs(60);

#[tokio::test(start_paused = true)]
async fn m4_shutdown_finishes_queued_work_within_the_grace_period() {
    let queue = JobQueue::start(config(2, 8));
    let start = Instant::now();
    for _ in 0..4 {
        queue.submit(TestJob::work(100)).await.unwrap();
    }
    let report = queue.shutdown(Duration::from_secs(10)).await;
    assert_eq!(report.succeeded, 4);
    assert_eq!(start.elapsed(), Duration::from_millis(200));
}

#[tokio::test(start_paused = true)]
async fn m4_shutdown_cancels_what_is_left_after_the_grace_period() {
    let queue = JobQueue::start(config(1, 4));
    queue.submit(TestJob::new(Kind::Endless)).await.unwrap();
    queue.submit(TestJob::work(10)).await.unwrap();
    queue.submit(TestJob::work(10)).await.unwrap();
    tokio::time::sleep(Duration::from_millis(1)).await; // the worker starts the endless job
    let start = Instant::now();
    let report = timeout(HANG_LIMIT, queue.shutdown(Duration::from_millis(500)))
        .await
        .expect("shutdown hung: is the work cancelled after the grace period?");
    assert_eq!(start.elapsed(), Duration::from_millis(500));
    assert_eq!(
        report,
        Report {
            cancelled: 3, // the endless job and the two queued ones
            ..Report::default()
        }
    );
}

#[tokio::test(start_paused = true)]
async fn m4_cancellation_also_stops_a_backoff_wait() {
    let queue = JobQueue::start(Config {
        workers: 1,
        capacity: 1,
        max_attempts: 5,
        backoff: Duration::from_secs(3600),
    });
    queue
        .submit(TestJob::new(Kind::FailThenSucceed { failures: 10 }))
        .await
        .unwrap();
    let start = Instant::now();
    let report = timeout(HANG_LIMIT, queue.shutdown(Duration::from_secs(1)))
        .await
        .expect("shutdown hung: is the backoff wait cancelled?");
    assert_eq!(start.elapsed(), Duration::from_secs(1));
    assert_eq!(
        report,
        Report {
            cancelled: 1,
            retries: 1,
            ..Report::default()
        }
    );
}
```

All tests use `#[tokio::test(start_paused = true)]` (Lesson 14.14): time stands still and jumps to the next timer, so a job that "works" for an hour takes no real time, and `assert_eq!` on durations is exact. With the skeleton above, the tests compile and fail with `not yet implemented`. That is your starting point.

### Step-by-step plan

#### M1 — One worker and a bounded queue

**Goal:** `start` with one worker (you may ignore `workers` for now), `submit`, `try_submit` and a `shutdown` that waits until the queue is empty. No retries, no cancellation yet (`grace` may be ignored).

Guidance: the queue's fields are a `tokio::sync::mpsc::Sender<(JobId, J)>`, a counter for the next id, the handle(s) of the worker task(s), and a shared `Report` (`Arc<std::sync::Mutex<Report>>` is fine: it is locked only for a moment, never across an `.await`). The worker loops over `recv()` and runs each job once. `shutdown` drops the sender (so `recv()` returns `None` once the queue is empty), waits for the worker, and returns a copy of the report.

<details>
<summary>Hint</summary>

To give the id only to jobs that were really accepted, first reserve a place in the channel and then create the id: `let permit = self.sender.reserve().await` waits for a free place (and fails only when the channel is closed); `permit.send((id, job))` cannot fail. For `try_submit` there is `self.sender.try_reserve()`, whose error `TrySendError::Full(())` tells you the queue is full. `shutdown(self, …)` takes `self` by value, so you can move the fields out with `let JobQueue { sender, … } = self;` and `drop(sender)`.

</details>

**Done when:**

```bash
cargo test --test m1_queue
```

```text
running 3 tests
test m1_runs_every_submitted_job ... ok
test m1_submit_waits_while_the_queue_is_full ... ok
test m1_try_submit_reports_a_full_queue ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Then `cargo clippy --all-targets` shows no warnings, and you commit: `git add code/p14-jobqueue && git commit -m "p14-jobqueue: M1 bounded queue with one worker"`.

#### M2 — N workers

**Goal:** `config.workers` workers that take jobs from the same queue and run them at the same time.

Guidance: an `mpsc::Receiver` has one owner. Put it into `Arc<tokio::sync::Mutex<…>>` and give every worker a clone of the `Arc`. Keep the worker handles in a `tokio::task::JoinSet<()>`, so `shutdown` can wait for all of them.

<details>
<summary>Hint</summary>

Take the next job in its own statement: `let next = queue.lock().await.recv().await;`. The `;` drops the lock guard before the job runs. If you write `while let Some(job) = queue.lock().await.recv().await { … }`, the guard lives for the whole loop body and your workers run one at a time; `m2_workers_run_jobs_in_parallel` then fails with `left: 800ms, right: 200ms`.

</details>

**Done when:** `cargo test --test m2_workers` prints `test result: ok. 2 passed; 0 failed`, the M1 tests still pass (`cargo test`), and you commit `p14-jobqueue: M2 N workers`.

#### M3 — Retries with exponential backoff

**Goal:** rules 5 and 9: up to `max_attempts` attempts, `Fatal` errors not retried, a doubling wait between attempts, the `retries` counter.

Guidance: write a helper `async fn run_with_retries(job: &J, …) -> Outcome`, where `Outcome` is your own private `enum` (`Succeeded`, `Failed`, later `Cancelled`). The worker then adds one to the right counter. `attempt` starts at 1, and the first wait is `config.backoff`.

<details>
<summary>Hint</summary>

A `for attempt in 1..=config.max_attempts` loop with a `delay` variable that starts at `config.backoff` and is doubled (`delay *= 2`) after each wait. A match guard keeps the cases short: `Err(JobError::Retryable(_)) if attempt == config.max_attempts => return Outcome::Failed`. Use `tokio::time::sleep` for the wait, never `std::thread::sleep` (Lesson 14.13), or the paused clock cannot help you and the worker thread blocks.

</details>

**Done when:** `cargo test --test m3_retries` prints `test result: ok. 3 passed; 0 failed`, all earlier tests pass, and you commit `p14-jobqueue: M3 retries with backoff`.

#### M4 — Cancellation and the grace period

**Goal:** rules 6 to 9: `shutdown(grace)` waits at most `grace`, then cancels running attempts, backoff waits and queued jobs.

Guidance: create one `CancellationToken` in `start`, give each worker a clone, and pass `token.clone()` to `job.run`. In `run_with_retries`, race the attempt and the backoff sleep against `token.cancelled()` with `tokio::select!`. In the worker, a job taken from the queue after cancellation is only counted, never run. In `shutdown`, wrap "wait for all workers" in `tokio::time::timeout(grace, …)`; if it times out, cancel the token and wait for the workers again (now they stop quickly).

<details>
<summary>Hint</summary>

The `Endless` test job returns `Err(Retryable)` as soon as it sees the cancellation, so both `select!` branches can be ready in the same poll. `select!` picks a random ready branch, so the test would pass sometimes and fail sometimes. Write `biased;` as the first line inside `tokio::select! { … }` and put the `cancelled()` branch first: then cancellation always wins (rule 8). If a test fails with "shutdown hung", your shutdown never cancels, or a worker does not notice the token.

</details>

**Done when:** `cargo test` runs all four test files and every one ends with `test result: ok.`; run it ten times in a row (`for i in $(seq 1 10); do cargo test -q || break; done`) and it never fails; you commit `p14-jobqueue: M4 cancellation and grace period`.

#### M5 — Demo, README and quality gate

**Goal:** a program that uses the library like a real user, a README, and a clean quality gate.

Create `examples/demo.rs` (Cargo builds files in `examples/` as separate programs that may use your library and its dev-dependencies):

```rust
use p14_jobqueue::{Config, Job, JobError, JobQueue};
use std::future::Future;
use std::time::Duration;
use tokio_util::sync::CancellationToken;

struct Resize {
    image: u32,
}

impl Job for Resize {
    fn run(
        &self,
        attempt: u32,
        _cancel: CancellationToken,
    ) -> impl Future<Output = Result<(), JobError>> + Send {
        let image = self.image;
        async move {
            tokio::time::sleep(Duration::from_millis(50)).await;
            if image == 7 {
                return Err(JobError::Fatal(format!("image {image} is broken")));
            }
            if image.is_multiple_of(3) && attempt == 1 {
                return Err(JobError::Retryable(format!("image {image}: timeout")));
            }
            println!("image {image}: resized (attempt {attempt})");
            Ok(())
        }
    }
}

#[tokio::main]
async fn main() {
    let queue = JobQueue::start(Config {
        workers: 3,
        capacity: 4,
        max_attempts: 3,
        backoff: Duration::from_millis(100),
    });
    for image in 1..=10 {
        queue.submit(Resize { image }).await.unwrap();
    }
    let report = queue.shutdown(Duration::from_secs(5)).await;
    println!("{report:?}");
}
```

Write `README.md` with: what the crate does (three sentences), the `Config` fields, the rules for retries and shutdown in your own words, and how to run the tests and the demo.

**Done when:**

```bash
cargo run --example demo
```

prints nine "resized" lines (their order changes from run to run) and ends with exactly this line:

```text
Report { succeeded: 9, failed: 1, cancelled: 0, retries: 3 }
```

and these commands pass without warnings or diffs:

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```

Commit `p14-jobqueue: M5 demo and README`, and merge your branch into `main` as you learned in Phase 01.

### Stretch goals

- **Ctrl+C:** in the demo, submit jobs in a loop and call `shutdown` when `tokio::signal::ctrl_c()` completes (Lesson 14.14). Print the report and exit with `ExitCode::FAILURE` if anything was cancelled.
- **Many producers:** add `pub fn submitter(&self) -> Submitter<J>`, a cloneable handle with its own `submit`, so several tasks can add jobs. What should `submit` on a `Submitter` return after `shutdown` started? Write a test for it.
- **Job status:** keep a `HashMap<JobId, Status>` (`Queued`, `Running { attempt }`, `Done(Outcome)`) and add `async fn status(&self, id: JobId) -> Option<Status>`.
- **Jitter:** add a random part to each backoff wait, so that many failing jobs do not all retry at the same moment. Keep the tests deterministic by making the random source part of `Config`.
- **loom:** extract the "count every job exactly once" logic into a small synchronous type with atomics and check it with loom (Lesson 14.9).

## Common mistakes

**1. The receiver lock is held while a job runs.** Workers then take turns instead of running in parallel. Release the lock before running the job (M2 hint).

**2. `std::thread::sleep` for the backoff.** It blocks a worker thread, ignores the token and the paused clock. Use `tokio::time::sleep` inside `select!`.

**3. Counting a cancelled job as failed.** When the token is cancelled, a job may return an error because of the cancellation. Rule 8 says that is `cancelled`, and `biased;` makes it reliable.

**4. Holding a `std::sync::Mutex` guard across an `.await`.** The report mutex is fine only if you lock it, change a number and drop the guard in one statement. Never `.await` while holding it (Lesson 06.9).

**5. `shutdown` that waits forever.** Without `timeout(grace, …)` a single endless job blocks the whole program. The M4 tests catch this with `HANG_LIMIT`.

## Quiz

1. Why is the channel bounded, and what does `submit` do when it is full?

<details>
<summary>Answer</summary>

So that producers cannot fill memory faster than workers can run jobs. When it is full, `submit` waits until a worker takes a job and a place is free: backpressure. `try_submit` returns `Full` instead of waiting.

</details>

2. Why does every worker need to release the receiver's lock before it runs a job?

<details>
<summary>Answer</summary>

While one worker holds the lock, no other worker can take a job. If the lock is held during the job, only one job runs at a time, whatever `workers` says.

</details>

3. Why must the backoff wait be inside a `select!` with `token.cancelled()`?

<details>
<summary>Answer</summary>

A backoff wait can be long (in the test, an hour). Without the `select!`, a shutdown would have to wait for it; with it, cancellation ends the wait at once and the job counts as cancelled.

</details>

4. Why can the tests use `assert_eq!` on durations such as `200ms`?

<details>
<summary>Answer</summary>

They run with a paused clock: time moves only when every task waits for a timer, and it jumps exactly to the next deadline. Elapsed times are therefore exact and the same in every run.

</details>

5. What does `biased;` change in `tokio::select!`, and why does M4 need it?

<details>
<summary>Answer</summary>

Normally `select!` picks a random branch among those that are ready; with `biased;` it checks them in written order. When an attempt ends because of the cancellation, both branches are ready at once; checking `cancelled()` first makes the job count as cancelled every time, not only sometimes.

</details>

## Summary

- The job queue combines a bounded `mpsc` channel (backpressure), N workers sharing one receiver, retries with exponential backoff, a `CancellationToken`, and a shutdown with a grace period.
- A generic library with a `Job` trait and a clear contract lets tests use their own job type with exact behaviours.
- Acceptance tests with a paused clock check timing exactly and run in milliseconds.
- Release locks before running work, use only Tokio's sleep in async code, and make every wait cancellable.
- Every accepted job ends in exactly one bucket: succeeded, failed or cancelled.

## Sources and further reading

- [Tokio — Graceful Shutdown](https://tokio.rs/tokio/topics/shutdown) — the shutdown pattern this project implements.
- [tokio::sync::mpsc::Sender::reserve](https://docs.rs/tokio/1.54.1/tokio/sync/mpsc/struct.Sender.html) — `reserve`, `try_reserve` and permits.
- [tokio::select! — fairness and `biased;`](https://docs.rs/tokio/1.54.1/tokio/macro.select.html) — how branches are chosen.
- [tokio::time::pause](https://docs.rs/tokio/1.54.1/tokio/time/fn.pause.html) — the paused clock used by the tests.
- [The Rust Reference — Return-position impl Trait in traits](https://doc.rust-lang.org/reference/types/impl-trait.html) — the rules behind `-> impl Future` in the `Job` trait.
- [AWS Architecture Blog — Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) — why backoff and jitter matter for retries.

---

[🏠 Start](../START-HERE.md) · [← 14.14 Cancellation, timeouts and graceful shutdown](14-cancellation-timeouts-and-graceful-shutdown.md) · [📋 Phase 14](README.md) · [14.16 Checkpoint: Stage 3 complete →](16-checkpoint-stage-3-complete.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
