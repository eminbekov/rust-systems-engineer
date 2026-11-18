<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 14.14 Отмена, тайм-ауты и корректное завершение](14-cancellation-timeouts-and-graceful-shutdown.md) · [📋 Этап 14](README.md) · [14.16 Контрольная точка: стадия 3 пройдена →](16-checkpoint-stage-3-complete.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/14-concurrency-and-async/15-project-concurrent-job-queue.md) <!-- nav -->

# Урок 14.15 — Проект: конкурентная очередь заданий

> **Этап 14 — Конкурентность и асинхронность подробно** · Урок 15 из 16 · ⏱ около 8 часов (4 занятия)

## Что вы узнаете

- Как объединить всё из этого этапа в одной библиотеке: ограниченный канал, рабочие задачи (workers), повторные попытки, токен отмены и корректное завершение.
- Как спроектировать асинхронный API (Application Programming Interface, программный интерфейс: типы и функции, которые предлагает ваша библиотека) вокруг трейта (`Job`) и дать ему ясный контракт.
- Как работать по приёмочным тестам с остановленными часами, чтобы время проверялось точно, а тесты никогда не были нестабильными.
- Как наращивать конкурентную программу маленькими вехами, по коммиту на каждую.

## Перед началом

- Вы прошли [Урок 14.14 — Отмена, тайм-ауты и корректное завершение](14-cancellation-timeouts-and-graceful-shutdown.md). Программа `service` оттуда — ближайший родственник этого проекта.
- Вы знаете ограниченные каналы `mpsc` и обратное давление из [Урока 14.13](13-tokio-tasks-and-channels.md), а правило «блокировка освобождается на `;`» — из [Урока 14.11](11-build-a-tiny-async-runtime.md).
- Вы знаете интеграционные тесты в `tests/` и общие помощники в `tests/common/mod.rs` из [Урока 05.6 — Тестирование подробно](../05-rust-intermediate/06-testing-in-depth.md).
- Ваш проект находится в `~/projects/rust/code/p14-jobqueue`. Работайте в ветке: `git switch -c p14-jobqueue`.

## Новые слова

| Слово | Значение |
|---|---|
| **очередь заданий** (job queue) | Часть программы, которая принимает единицы работы (задания), хранит их по порядку и отдаёт рабочим задачам на выполнение. Настоящие примеры: отправка писем, изменение размера картинок, построение отчётов. |
| **рабочая задача** (worker) | Задача, которая берёт задания из очереди и выполняет их одно за другим. |
| **повторная попытка** (retry) | Повторный запуск упавшего задания. Повторяются только *временные* ошибки (retryable, например тайм-аут); *фатальные* (fatal, неверный вход) — нет. |
| **экспоненциальная задержка** (exponential backoff) | Ожидание, которое растёт перед каждой новой попыткой: 100 мс, затем 200 мс, затем 400 мс. Оно даёт перегруженному сервису время прийти в себя. |
| **льготный период** (grace period) | Время, которое завершение ждёт, пока работа закончится нормально, прежде чем отменить оставшееся. |
| **приёмочный тест** (acceptance test) | Тест, который прилагается к задаче и решает, выполнена ли веха. |
| **RPITIT** (Return-Position `impl Trait` In Traits, `impl Trait` в позиции возвращаемого значения в трейтах) | Запись `-> impl Future<…>` как типа результата метода трейта. Позволяет методу трейта быть асинхронным без упаковки в `Box`. |

## 1. Как устроена очередь

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

Части и уроки, в которых вы их изучили:

1. **Ограниченный канал `mpsc`** хранит ждущие задания. `submit` ждёт, пока канал полон (обратное давление, Урок 14.13). `try_submit` никогда не ждёт.
2. **N рабочих задач** делят один получатель через `Arc<tokio::sync::Mutex<Receiver>>`, как в `service` из Урока 14.14.
3. **Повторные попытки**: каждое задание получает до `max_attempts` попыток с удваивающимся ожиданием между ними. Ожидание должно быть отменяемым.
4. **`CancellationToken`** (Урок 14.14) общий для всех рабочих задач и передаётся каждому заданию.
5. **`shutdown(grace)`** выбрасывает отправителя, поэтому рабочие задачи доделывают задания из очереди и останавливаются. Если это занимает больше `grace`, токен отменяется; тогда рабочие задачи останавливают выполняемые задания, а задания из очереди считают отменёнными, не запуская их.
6. **`Report`** считает, что произошло с каждым принятым заданием.

## Проект

### Что вы создадите

Библиотечный крейт `p14-jobqueue` с обобщённой `JobQueue<J: Job>`, приёмочные тесты в `tests/`, маленькую демонстрационную программу в `examples/demo.rs` и короткий `README.md`. Полного решения нет; у каждой вехи есть подсказки.

### Контракт

**Зависимости в `Cargo.toml`:**

```bash
cargo new --lib code/p14-jobqueue
cd code/p14-jobqueue
cargo add tokio@1 --features rt-multi-thread,macros,sync,time,signal
cargo add tokio-util@0.7
cargo add --dev tokio@1 --features test-util
```

**Публичный API.** Начните `src/lib.rs` ровно с этих типов и сигнатур. Тела `todo!()` и поле-заглушку замените своими; публичные имена, поля и сигнатуры должны остаться такими, потому что их используют тесты. Этот файл компилируется как есть:

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

**Правила поведения.** Их проверяют тесты:

1. `JobQueue::start` сразу запускает `workers` рабочих задач. Её нужно вызывать внутри рантайма Tokio.
2. Принятые задания получают идентификаторы `JobId(1)`, `JobId(2)`, … в порядке принятия.
3. `submit` ждёт, пока в очереди `capacity` заданий. `try_submit` вместо этого возвращает `Err(SubmitError::Full(job))` и отдаёт задание обратно.
4. Каждая рабочая задача выполняет одно задание за раз. Одновременно выполняется до `workers` заданий.
5. Задание запускается не больше `max_attempts` раз. `Ok(())` считается как `succeeded`. `Err(Fatal)` сразу считается как `failed`. `Err(Retryable)` повторяется после ожидания `backoff`, затем `2 × backoff`, затем `4 × backoff`, …; каждая повторная попытка добавляет 1 к `retries` в момент, когда она решена, **до** ожидания (поэтому задание, отменённое во время ожидания задержки, всё равно учитывает эту попытку). Если последняя попытка неудачна, задание считается `failed`.
6. `shutdown(grace)` больше не принимает заданий и возвращает `Report` только после того, как остановилась каждая рабочая задача.
7. Если очередь опустела до конца `grace`, `shutdown` возвращается, как только закончено последнее задание, и ничего не отменяется.
8. Когда `grace` заканчивается, общий токен отменяется. Выполняемая попытка или ожидание задержки тогда сразу заканчивается, и задание считается `cancelled`; повтора нет. Задания, ещё стоящие в очереди, считаются `cancelled` и никогда не запускаются. Если попытка и отмена заканчиваются в один момент, побеждает отмена.
9. Каждое принятое задание учитывается ровно один раз в `succeeded + failed + cancelled`.

**Приёмочные тесты.** Скопируйте эти файлы в `tests/`. Сначала тестовое задание с четырьмя вариантами поведения, в `tests/common/mod.rs` (`#![allow(dead_code)]` стоит потому, что каждый файл тестов использует только часть помощников):

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

Все тесты используют `#[tokio::test(start_paused = true)]` (Урок 14.14): время стоит и перескакивает к следующему таймеру, поэтому задание, которое «работает» час, не занимает реального времени, а `assert_eq!` для длительностей точен. Со скелетом выше тесты компилируются и падают с `not yet implemented`. Это ваша отправная точка.

### Пошаговый план

#### M1 — Одна рабочая задача и ограниченная очередь

**Цель:** `start` с одной рабочей задачей (`workers` пока можно игнорировать), `submit`, `try_submit` и `shutdown`, который ждёт, пока очередь опустеет. Пока без повторных попыток и без отмены (`grace` можно игнорировать).

Подсказки к работе: поля очереди — это `tokio::sync::mpsc::Sender<(JobId, J)>`, счётчик для следующего идентификатора, дескриптор(ы) рабочей задачи и общий `Report` (`Arc<std::sync::Mutex<Report>>` подойдёт: он блокируется лишь на мгновение и никогда через `.await`). Рабочая задача в цикле вызывает `recv()` и выполняет каждое задание один раз. `shutdown` выбрасывает отправителя (поэтому `recv()` вернёт `None`, когда очередь опустеет), ждёт рабочую задачу и возвращает копию отчёта.

<details>
<summary>Подсказка</summary>

Чтобы выдавать идентификатор только реально принятым заданиям, сначала зарезервируйте место в канале, а затем создайте идентификатор: `let permit = self.sender.reserve().await` ждёт свободного места (и падает, только если канал закрыт); `permit.send((id, job))` упасть не может. Для `try_submit` есть `self.sender.try_reserve()`, чья ошибка `TrySendError::Full(())` сообщает, что очередь полна. `shutdown(self, …)` принимает `self` по значению, поэтому поля можно вынуть через `let JobQueue { sender, … } = self;` и сделать `drop(sender)`.

</details>

**Готово, когда:**

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

Затем `cargo clippy --all-targets` не показывает предупреждений, и вы делаете коммит: `git add code/p14-jobqueue && git commit -m "p14-jobqueue: M1 bounded queue with one worker"`.

#### M2 — N рабочих задач

**Цель:** `config.workers` рабочих задач, которые берут задания из одной очереди и выполняют их одновременно.

Подсказки к работе: у `mpsc::Receiver` один владелец. Положите его в `Arc<tokio::sync::Mutex<…>>` и дайте каждой рабочей задаче клон `Arc`. Храните дескрипторы рабочих задач в `tokio::task::JoinSet<()>`, чтобы `shutdown` мог дождаться их всех.

<details>
<summary>Подсказка</summary>

Берите следующее задание отдельной инструкцией: `let next = queue.lock().await.recv().await;`. `;` уничтожает guard блокировки до того, как задание начнёт выполняться. Если написать `while let Some(job) = queue.lock().await.recv().await { … }`, guard живёт всё тело цикла, и ваши рабочие задачи работают по одной; тогда `m2_workers_run_jobs_in_parallel` падает с `left: 800ms, right: 200ms`.

</details>

**Готово, когда:** `cargo test --test m2_workers` печатает `test result: ok. 2 passed; 0 failed`, тесты M1 по-прежнему проходят (`cargo test`), и вы делаете коммит `p14-jobqueue: M2 N workers`.

#### M3 — Повторные попытки с экспоненциальной задержкой

**Цель:** правила 5 и 9: до `max_attempts` попыток, `Fatal` не повторяется, удваивающееся ожидание между попытками, счётчик `retries`.

Подсказки к работе: напишите помощника `async fn run_with_retries(job: &J, …) -> Outcome`, где `Outcome` — ваш собственный приватный `enum` (`Succeeded`, `Failed`, позже `Cancelled`). Затем рабочая задача добавляет единицу к нужному счётчику. `attempt` начинается с 1, а первое ожидание — `config.backoff`.

<details>
<summary>Подсказка</summary>

Цикл `for attempt in 1..=config.max_attempts` с переменной `delay`, которая начинается с `config.backoff` и удваивается (`delay *= 2`) после каждого ожидания. Охранное условие (match guard) делает случаи короткими: `Err(JobError::Retryable(_)) if attempt == config.max_attempts => return Outcome::Failed`. Для ожидания используйте `tokio::time::sleep`, никогда `std::thread::sleep` (Урок 14.13), иначе остановленные часы вам не помогут, а поток рабочей задачи заблокируется.

</details>

**Готово, когда:** `cargo test --test m3_retries` печатает `test result: ok. 3 passed; 0 failed`, все прежние тесты проходят, и вы делаете коммит `p14-jobqueue: M3 retries with backoff`.

#### M4 — Отмена и льготный период

**Цель:** правила 6–9: `shutdown(grace)` ждёт не дольше `grace`, затем отменяет выполняемые попытки, ожидания задержки и задания в очереди.

Подсказки к работе: создайте один `CancellationToken` в `start`, дайте каждой рабочей задаче клон и передавайте `token.clone()` в `job.run`. В `run_with_retries` устройте гонку попытки и сна задержки с `token.cancelled()` через `tokio::select!`. В рабочей задаче задание, взятое из очереди после отмены, только учитывается и никогда не запускается. В `shutdown` оберните «дождаться всех рабочих задач» в `tokio::time::timeout(grace, …)`; если время вышло, отмените токен и снова дождитесь рабочих задач (теперь они остановятся быстро).

<details>
<summary>Подсказка</summary>

Тестовое задание `Endless` возвращает `Err(Retryable)`, как только видит отмену, поэтому обе ветки `select!` могут стать готовыми в одном и том же опросе. `select!` выбирает случайную готовую ветку, и тест проходил бы то так, то эдак. Напишите `biased;` первой строкой внутри `tokio::select! { … }` и поставьте ветку `cancelled()` первой: тогда отмена всегда побеждает (правило 8). Если тест падает с «shutdown hung», ваше завершение никогда не отменяет работу или рабочая задача не замечает токен.

</details>

**Готово, когда:** `cargo test` запускает все четыре файла тестов, и каждый заканчивается на `test result: ok.`; запустите его десять раз подряд (`for i in $(seq 1 10); do cargo test -q || break; done`), и он ни разу не падает; вы делаете коммит `p14-jobqueue: M4 cancellation and grace period`.

#### M5 — Демо, README и контроль качества

**Цель:** программа, которая использует библиотеку как настоящий пользователь, README и чистый контроль качества.

Создайте `examples/demo.rs` (Cargo собирает файлы в `examples/` как отдельные программы, которые могут использовать вашу библиотеку и её зависимости для разработки):

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

Напишите `README.md`: что делает крейт (три предложения), поля `Config`, правила повторных попыток и завершения своими словами и как запускать тесты и демо.

**Готово, когда:**

```bash
cargo run --example demo
```

печатает девять строк «resized» (их порядок меняется от запуска к запуску) и заканчивается ровно этой строкой:

```text
Report { succeeded: 9, failed: 1, cancelled: 0, retries: 3 }
```

и эти команды проходят без предупреждений и различий:

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```

Сделайте коммит `p14-jobqueue: M5 demo and README` и слейте ветку в `main`, как вы учились на Этапе 01.

### Дополнительные цели

- **Ctrl+C:** в демо отправляйте задания в цикле и вызывайте `shutdown`, когда завершится `tokio::signal::ctrl_c()` (Урок 14.14). Напечатайте отчёт и выйдите с `ExitCode::FAILURE`, если что-то было отменено.
- **Много производителей:** добавьте `pub fn submitter(&self) -> Submitter<J>` — клонируемый дескриптор со своим `submit`, чтобы задания могли добавлять несколько задач. Что должен возвращать `submit` у `Submitter` после начала `shutdown`? Напишите для этого тест.
- **Состояние задания:** храните `HashMap<JobId, Status>` (`Queued`, `Running { attempt }`, `Done(Outcome)`) и добавьте `async fn status(&self, id: JobId) -> Option<Status>`.
- **Джиттер:** добавьте случайную часть к каждому ожиданию задержки, чтобы много падающих заданий не повторялись в один и тот же момент. Сохраните детерминированность тестов, сделав источник случайности частью `Config`.
- **loom:** вынесите логику «учесть каждое задание ровно один раз» в маленький синхронный тип с атомарными переменными и проверьте его через loom (Урок 14.9).

## Частые ошибки

**1. Блокировка получателя удерживается, пока выполняется задание.** Тогда рабочие задачи работают по очереди, а не параллельно. Освобождайте блокировку до выполнения задания (подсказка M2).

**2. `std::thread::sleep` для задержки.** Он блокирует поток рабочей задачи, не замечает токен и остановленные часы. Используйте `tokio::time::sleep` внутри `select!`.

**3. Отменённое задание считается упавшим.** Когда токен отменён, задание может вернуть ошибку именно из-за отмены. По правилу 8 это `cancelled`, и `biased;` делает это надёжным.

**4. Guard `std::sync::Mutex` через `.await`.** Мьютекс отчёта допустим, только если вы блокируете его, меняете число и уничтожаете guard в одной инструкции. Никогда не делайте `.await`, держа его (Урок 06.9).

**5. `shutdown`, который ждёт вечно.** Без `timeout(grace, …)` одно бесконечное задание блокирует всю программу. Тесты M4 ловят это через `HANG_LIMIT`.

## Вопросы для самопроверки

1. Зачем канал ограничен и что делает `submit`, когда он полон?

<details>
<summary>Ответ</summary>

Чтобы производители не могли заполнять память быстрее, чем рабочие задачи выполняют задания. Когда канал полон, `submit` ждёт, пока рабочая задача не возьмёт задание и не освободится место: обратное давление. `try_submit` вместо ожидания возвращает `Full`.

</details>

2. Почему каждая рабочая задача должна освобождать блокировку получателя до выполнения задания?

<details>
<summary>Ответ</summary>

Пока одна рабочая задача держит блокировку, никакая другая не может взять задание. Если блокировка держится во время задания, одновременно выполняется только одно задание, что бы ни было указано в `workers`.

</details>

3. Почему ожидание задержки должно быть внутри `select!` с `token.cancelled()`?

<details>
<summary>Ответ</summary>

Ожидание задержки может быть долгим (в тесте — час). Без `select!` завершению пришлось бы его ждать; с ним отмена сразу заканчивает ожидание, и задание считается отменённым.

</details>

4. Почему тесты могут использовать `assert_eq!` для длительностей вроде `200ms`?

<details>
<summary>Ответ</summary>

Они выполняются с остановленными часами: время движется, только когда каждая задача ждёт таймер, и перескакивает ровно на следующий срок. Поэтому прошедшее время точное и одинаковое при каждом запуске.

</details>

5. Что меняет `biased;` в `tokio::select!` и зачем он нужен в M4?

<details>
<summary>Ответ</summary>

Обычно `select!` выбирает случайную ветку среди готовых; с `biased;` он проверяет их в порядке записи. Когда попытка заканчивается из-за отмены, обе ветки готовы одновременно; если первой проверять `cancelled()`, задание каждый раз считается отменённым, а не только иногда.

</details>

## Итоги

- Очередь заданий объединяет ограниченный канал `mpsc` (обратное давление), N рабочих задач с общим получателем, повторные попытки с экспоненциальной задержкой, `CancellationToken` и завершение с льготным периодом.
- Обобщённая библиотека с трейтом `Job` и ясным контрактом позволяет тестам использовать собственный тип задания с точным поведением.
- Приёмочные тесты с остановленными часами проверяют время точно и выполняются за миллисекунды.
- Освобождайте блокировки до выполнения работы, используйте в асинхронном коде только сон Tokio и делайте каждое ожидание отменяемым.
- Каждое принятое задание попадает ровно в одну корзину: succeeded, failed или cancelled.

## Источники и что почитать

- [Tokio — Graceful Shutdown](https://tokio.rs/tokio/topics/shutdown) — шаблон завершения, который реализует этот проект.
- [tokio::sync::mpsc::Sender::reserve](https://docs.rs/tokio/1.54.1/tokio/sync/mpsc/struct.Sender.html) — `reserve`, `try_reserve` и разрешения (permits).
- [tokio::select! — fairness and `biased;`](https://docs.rs/tokio/1.54.1/tokio/macro.select.html) — как выбираются ветки.
- [tokio::time::pause](https://docs.rs/tokio/1.54.1/tokio/time/fn.pause.html) — остановленные часы, которые используют тесты.
- [The Rust Reference — Return-position impl Trait in traits](https://doc.rust-lang.org/reference/types/impl-trait.html) — правила, на которых держится `-> impl Future` в трейте `Job`.
- [AWS Architecture Blog — Exponential Backoff And Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) — почему задержка и джиттер важны для повторных попыток.

---

[🏠 Начало](../START-HERE.md) · [← 14.14 Отмена, тайм-ауты и корректное завершение](14-cancellation-timeouts-and-graceful-shutdown.md) · [📋 Этап 14](README.md) · [14.16 Контрольная точка: стадия 3 пройдена →](16-checkpoint-stage-3-complete.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
