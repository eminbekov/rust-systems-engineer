<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 14.8 Неблокирующие структуры данных](08-lock-free-data-structures.md) · [📋 Этап 14](README.md) · [14.10 Как работают future: Future, poll и Waker →](10-how-futures-work.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/14-concurrency-and-async/09-testing-concurrent-code-with-loom.md) <!-- nav -->

# Урок 14.9 — Тестирование конкурентного кода с loom

> **Этап 14 — Конкурентность и асинхронность подробно** · Урок 9 из 16 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Почему обычный тест, даже повторённый тысячи раз, может проходить, хотя конкурентный код неправилен.
- Что такое средство проверки моделей (model checker) и как крейт `loom` прогоняет один тест при каждом возможном расписании потоков.
- Как подготовить крейт к loom: флаг `cfg(loom)`, модуль `sync` и отдельный файл тестов.
- Как loom находит потерянное обновление, неверный порядок памяти, спин-цикл, который никогда не заканчивается, и взаимную блокировку.
- Как держать модели loom маленькими и что loom проверить не может.

## Перед началом

- Вы прошли [Урок 14.8 — Неблокирующие структуры данных](08-lock-free-data-structures.md).
- Вы знаете атомарные операции и `fetch_add` из [Урока 14.5](05-atomics.md), а `Relaxed`, `Release` и `Acquire` — из [Урока 14.6 — Порядок памяти](06-memory-ordering.md).
- Вы знаете спин-циклы из [Урока 14.7 — Пишем спин-блокировку](07-building-a-spin-lock.md) и взаимные блокировки из [Урока 14.1](01-why-concurrency-is-hard.md).
- Вы знаете `Arc`, `thread::spawn` и `join` из [Урока 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md), а интеграционные тесты в `tests/` — из [Урока 05.6](../05-rust-intermediate/06-testing-in-depth.md).
- Код этого урока находится в `~/projects/rust/code/p14-loom`.

## Новые слова

| Слово | Значение |
|---|---|
| **чередование** (interleaving) | Один возможный порядок, в котором выполняются шаги нескольких потоков. У двух потоков по два шага уже шесть чередований. |
| **расписание** (schedule) | Решения «какой поток работает следующим», которые дают одно чередование. Их принимает планировщик операционной системы, и вы не можете ими управлять. |
| **потерянное обновление** (lost update) | Ошибка, когда два потока читают одно и то же старое значение, оба записывают новое, и одна из записей теряется. |
| **стресс-тест** (stress test) | Тест, который много раз запускает конкурентный код в надежде, что редкое плохое расписание случится хотя бы раз. |
| **средство проверки моделей** (model checker) | Инструмент, который не надеется: он пробует каждое возможное расписание (или каждое расписание в пределах ограничения) и проверяет ваши утверждения в каждом. |
| **loom** | Средство проверки моделей для конкурентного кода на Rust от проекта Tokio. Его типы заменяют атомарные типы, блокировки и потоки из `std` во время теста. |
| **модель** (`loom::model`) | Замыкание, которое loom запускает снова и снова, по одному разу на расписание. Каждый запуск называется *итерацией*. |
| **детерминированный** (deterministic) | Даёт один и тот же результат при одном и том же входе. Тесты loom должны быть детерминированными: без случайности, без часов, без настоящего ввода-вывода (I/O, input/output: файлы, сеть). |
| **вытеснение** (preemption) | Планировщик останавливает работающий поток посреди работы и запускает другой. |
| **предел вытеснений** (preemption bound) | Ограничение на число вытеснений, которые loom пробует в одном расписании. Меньший предел — меньше итераций. |
| **комбинаторный взрыв состояний** (state explosion) | Число возможных расписаний растёт очень быстро с каждым новым потоком или шагом, поэтому большие модели проверяются слишком долго. |
| **флаг `cfg`** (`cfg` flag) | Имя, которое вы включаете при компиляции (`--cfg loom`). Код с пометкой `#[cfg(loom)]` компилируется, только когда флаг включён. |
| **`RUSTFLAGS`** | Переменная окружения с дополнительными параметрами, которые Cargo передаёт компилятору Rust `rustc`. |

## 1. Почему обычных тестов недостаточно

Вот счётчик с ошибкой. `increment` читает значение, а затем записывает значение плюс один — в два отдельных шага. С этой ошибкой, *потерянным обновлением*, вы встречались в [Уроке 14.1](01-why-concurrency-is-hard.md): два потока могут прочитать одно и то же число.

Создайте библиотеку и замените содержимое `src/lib.rs`:

```bash
cd ~/projects/rust
cargo new --lib code/p14-loom
cd code/p14-loom
```

```rust
//! Small concurrent types, tested with loom.

use std::sync::atomic::{AtomicUsize, Ordering};

/// A counter that many threads can increment.
pub struct Counter {
    value: AtomicUsize,
}

impl Counter {
    pub fn new() -> Self {
        Counter {
            value: AtomicUsize::new(0),
        }
    }

    /// Adds one. BUG: the read and the write are two separate steps.
    pub fn increment(&self) {
        let current = self.value.load(Ordering::SeqCst);
        self.value.store(current + 1, Ordering::SeqCst);
    }

    pub fn get(&self) -> usize {
        self.value.load(Ordering::SeqCst)
    }
}

impl Default for Counter {
    fn default() -> Self {
        Self::new()
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::Arc;
    use std::thread;

    #[test]
    fn two_threads_increment_once_each() {
        for _ in 0..1000 {
            let counter = Arc::new(Counter::new());
            let other = Arc::clone(&counter);
            let handle = thread::spawn(move || other.increment());
            counter.increment();
            handle.join().unwrap();
            assert_eq!(counter.get(), 2);
        }
    }
}
```

**Что происходит:**

1. Тест 1000 раз повторяет одну и ту же маленькую историю: два потока, каждый увеличивает счётчик один раз, результат должен быть 2.
2. `thread::spawn` запускает второй поток. Пока операционная система его запускает, главный поток обычно уже заканчивает свой `increment`.
3. Поэтому два увеличения почти никогда не пересекаются, и ошибка остаётся скрытой.

Запустите `cargo test`:

```text
running 1 test
test tests::two_threads_increment_once_each ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.11s
```

Тест зелёный, а код неправильный. В этом главная проблема конкурентного кода: плохое расписание существует, но оно редкое и зависит от времени, от машины и от нагрузки.

Иногда стресс-тест всё же находит ошибку. Лакмусовый тест «данные плюс флаг с `Relaxed`» из [Урока 14.6](06-memory-ordering.md) увидел старые данные несколько тысяч раз за 100 000 раундов на Mac с M4, но только потому, что `DATA` и `READY` лежали в разных кэш-линиях; без этого выравнивания он не увидел ни одного случая, а машина x86-64 не показала бы ни одного вовсе. Тест, который падает «иногда, на некоторых машинах», — нестабильный тест, и когда он проходит, это не говорит, что код правильный.

## 2. Что делает loom

Посмотрите на два шага каждого потока в счётчике. Вот два из возможных чередований:

```text
Schedule A (correct)              Schedule B (lost update)
main:  load   -> 0                main:  load   -> 0
main:  store  1                   other: load   -> 0
other: load   -> 1                main:  store  1
other: store  2                   other: store  1
result: 2                         result: 1
```

Средство проверки моделей запускает ваш тест по одному разу для **каждого** такого расписания. Loom делает это так:

1. Вы используете типы loom (`loom::sync::atomic::AtomicUsize`, `loom::sync::Mutex`, `loom::thread::spawn`, `loom::sync::Arc`) вместо типов из `std`, но только во время тестирования.
2. Каждая операция с этими типами — точка, где loom может переключиться на другой поток. Кто работает, решает собственный планировщик loom, а не операционная система.
3. `loom::model(|| { ... })` запускает ваше замыкание, запоминает сделанные выборы, затем запускает его снова с другими выборами — пока не пройдёт все расписания.
4. Для атомарных типов loom ещё и моделирует модель памяти: загрузка с `Relaxed` или `Acquire` может вернуть *более старое* значение, если это разрешают правила из [Урока 14.6](06-memory-ordering.md). Поэтому loom находит ошибки порядка даже на машине, которая никогда бы их не показала.
5. Loom также замечает взаимные блокировки (все потоки заблокированы) и утечки объектов loom.

Loom детерминирован: если тест падает, он падает при каждом запуске, на любом компьютере.

## 3. Подготовка крейта к loom

Типы loom работают только внутри `loom::model`. Обычная сборка должна по-прежнему использовать `std`. В Уроке 14.6 типы loom использовал сам файл теста, поэтому loom мог быть зависимостью для разработки (dev-dependency). Здесь проверяемый код находится в вашей библиотеке, поэтому переключаться между loom и `std` должна сама библиотека. Обычное решение состоит из трёх частей.

**Часть 1: `Cargo.toml`.** Подключите loom, только когда включён флаг `loom`, и сообщите компилятору, что `loom` — известное имя флага:

```toml
[package]
name = "p14-loom"
version = "0.1.0"
edition = "2024"

[dependencies]

[target.'cfg(loom)'.dependencies]
loom = "0.7"

[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ["cfg(loom)"] }
```

`[target.'cfg(loom)'.dependencies]` означает «эта зависимость существует, только когда `cfg(loom)` истинно». Без части `[lints.rust]` компилятор предупреждает `unexpected cfg condition name: loom`, потому что проверяет, что каждое имя `cfg` известно.

**Часть 2: модуль `sync`.** Один файл решает, откуда берутся типы. Создайте `src/sync.rs`:

```rust
//! One place that decides where the sync types come from.

#[cfg(loom)]
pub(crate) use loom::sync::atomic::{AtomicUsize, Ordering};

#[cfg(not(loom))]
pub(crate) use std::sync::atomic::{AtomicUsize, Ordering};
```

В `src/lib.rs` добавьте в начало `mod sync;` и замените строку `use` на `use sync::{AtomicUsize, Ordering};`. Остальной код не меняется. `pub(crate)` делает имена видимыми во всём крейте, но не снаружи.

**Часть 3: файл тестов только для loom.** Создайте `tests/loom.rs`:

```rust
#![cfg(loom)]

use loom::sync::Arc;
use loom::thread;
use p14_loom::Counter;

#[test]
fn two_threads_increment_once_each() {
    loom::model(|| {
        let counter = Arc::new(Counter::new());
        let other = Arc::clone(&counter);
        let handle = thread::spawn(move || other.increment());
        counter.increment();
        handle.join().unwrap();
        assert_eq!(counter.get(), 2);
    });
}
```

**Что происходит:**

1. `#![cfg(loom)]` в начале файла выключает весь файл в обычной сборке. `cargo test` всё равно его компилирует и сообщает для него `running 0 tests`.
2. Тело теста — та же история, что и раньше, но внутри `loom::model` и с `loom::sync::Arc` и `loom::thread`.
3. Цикла на 1000 повторений нет: loom сам повторяет замыкание, по одному разу на расписание.

Запустите тесты loom с флагом, в режиме release и только для файла тестов `loom`:

```bash
RUSTFLAGS="--cfg loom" cargo test --release --test loom
```

`--release` важен, потому что loom может запустить замыкание тысячи раз. `--test loom` важен, потому что модульные тесты в `src/lib.rs` не используют `loom::model` (в разделе «Частые ошибки» показано, что иначе произойдёт).

> 💡 **Совет:** изменение `RUSTFLAGS` заставляет Cargo пересобрать все зависимости. Если вы часто переключаетесь между обычными запусками и запусками loom, дайте loom собственную папку сборки: `CARGO_TARGET_DIR=target/loom RUSTFLAGS="--cfg loom" cargo test --release --test loom`. Тогда обе сборки остаются в кэше.

## 4. Loom находит потерянное обновление

Выполните команду. Тест падает с первой попытки:

```text
     Running tests/loom.rs (target/release/build/p14-loom/8533a2a354401770/out/loom-8533a2a354401770)

running 1 test
test two_threads_increment_once_each ... FAILED

failures:

---- two_threads_increment_once_each stdout ----

thread 'two_threads_increment_once_each' (4876607) panicked at tests/loom.rs:15:9:
assertion `left == right` failed
  left: 1
 right: 2
```

`left: 1` — это расписание B из раздела 2: одно обновление потерялось. Исправьте `increment`, чтобы чтение и запись были одним неделимым шагом:

```rust
    /// Adds one. The read and the write are one indivisible step.
    pub fn increment(&self) {
        self.value.fetch_add(1, Ordering::SeqCst);
    }
```

Запустите снова, на этот раз с `LOOM_LOG=info`, чтобы loom сообщил, сколько расписаний он проверил. `--nocapture` показывает журнал, который иначе скрывает система запуска тестов:

```bash
RUSTFLAGS="--cfg loom" LOOM_LOG=info cargo test --release --test loom -- --nocapture
```

```text
running 1 test
test two_threads_increment_once_each ...  INFO loom::model: Completed in 4 iterations
ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(Журнал также печатает строку на каждое переключение потока и цвета; здесь они опущены.) «Completed in 4 iterations» означает, что loom запустил замыкание четыре раза, и утверждение выполнилось во всех четырёх. Это не «прошло 1000 раз по везению», а «прошло в каждом существующем расписании».

> 🧪 **Тестирование:** обычный тест тоже оставьте. Он проверяет, что код работает с настоящими типами `std` и настоящими потоками, и `cargo test` запускает его при каждом изменении. Тест loom проверяет все расписания. Они отвечают на разные вопросы.

## 5. Loom находит ошибку порядка памяти

Теперь тип, в котором каждая отдельная операция атомарна, но *порядок* неверный. `Mailbox` хранит одно число: один поток кладёт его, другой забирает. Добавьте два имени в `src/sync.rs` (обе строки `use` становятся `{AtomicBool, AtomicU64, AtomicUsize, Ordering}`), затем создайте `src/mailbox.rs`:

```rust
use crate::sync::{AtomicBool, AtomicU64, Ordering};

/// A box for one number: one thread puts it, another thread takes it.
pub struct Mailbox {
    value: AtomicU64,
    ready: AtomicBool,
}

impl Mailbox {
    pub fn new() -> Self {
        Mailbox {
            value: AtomicU64::new(0),
            ready: AtomicBool::new(false),
        }
    }

    pub fn put(&self, value: u64) {
        self.value.store(value, Ordering::Relaxed);
        self.ready.store(true, Ordering::Relaxed); // BUG: should be Release
    }

    pub fn take(&self) -> Option<u64> {
        if self.ready.load(Ordering::Relaxed) {
            // BUG: should be Acquire
            Some(self.value.load(Ordering::Relaxed))
        } else {
            None
        }
    }
}

impl Default for Mailbox {
    fn default() -> Self {
        Self::new()
    }
}
```

В `src/lib.rs` добавьте `mod mailbox;` и `pub use mailbox::Mailbox;`. Затем добавьте тест в `tests/loom.rs`:

```rust
#[test]
fn taker_sees_the_value_that_was_put() {
    use p14_loom::Mailbox;
    loom::model(|| {
        let mailbox = Arc::new(Mailbox::new());
        let sender = Arc::clone(&mailbox);
        let handle = thread::spawn(move || sender.put(42));
        if let Some(value) = mailbox.take() {
            assert_eq!(value, 42);
        }
        handle.join().unwrap();
    });
}
```

**Что происходит:**

1. Отправитель записывает `42`, затем устанавливает `ready`.
2. Получатель проверяет `ready`. Только если там `true`, он читает значение, и значение должно быть `42`.
3. С `Relaxed` две атомарные переменные ничем не связаны. Поток может увидеть `ready == true` и при этом всё ещё видеть старое значение `0`. Именно это показал стресс-тест на arm64 в разделе 1.

```text
---- taker_sees_the_value_that_was_put stdout ----

thread 'taker_sees_the_value_that_was_put' (4883659) panicked at tests/loom.rs:27:13:
assertion `left == right` failed
  left: 0
 right: 42
```

Loom сразу нашёл устаревшее значение, на любой машине. Исправление — пара release/acquire из [Урока 14.6](06-memory-ordering.md): `put` записывает `ready` с `Ordering::Release`, а `take` загружает его с `Ordering::Acquire`. Загрузка, которая видит опубликованное (released) `true`, видит и всё, что отправитель записал до этого. После исправления тест проходит за 6 итераций.

## 6. Спин-циклы и взаимные блокировки

**Спин-цикл** (spin loop) ждёт, проверяя флаг снова и снова. Вот `Latch` — «ворота», которые сначала закрыты (`src/latch.rs`; добавьте `mod latch;` и `pub use latch::Latch;` в `src/lib.rs`):

```rust
use crate::sync::{AtomicBool, Ordering, hint};

/// A gate that starts closed. Threads wait until someone opens it.
pub struct Latch {
    open: AtomicBool,
}

impl Latch {
    pub fn new() -> Self {
        Latch {
            open: AtomicBool::new(false),
        }
    }

    pub fn open(&self) {
        self.open.store(true, Ordering::Release);
    }

    pub fn wait(&self) {
        while !self.open.load(Ordering::Acquire) {
            hint::spin_loop();
        }
    }
}

impl Default for Latch {
    fn default() -> Self {
        Self::new()
    }
}
```

`hint` тоже берётся из модуля `sync`: добавьте `pub(crate) use loom::hint;` под `#[cfg(loom)]` и `pub(crate) use std::hint;` под `#[cfg(not(loom))]`.

Зачем? Планировщик loom *намеренно* несправедлив: он пробует каждое расписание, включая «ждущий поток работает вечно, а другой не запускается никогда». `loom::hint::spin_loop()` говорит loom: «я не могу продвинуться, пока не поработает другой поток», и loom в этом месте переключает потоки. Если вместо этого вызвать `std::hint::spin_loop()`, loom видит бесконечный цикл:

```text
thread 'wait_returns_after_open' (4904556) panicked at /Users/you/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/loom-0.7.3/src/rt/path.rs:175:9:
Model exceeded maximum number of branches. This is often caused by an algorithm requiring the processor to make progress, e.g. spin locks.
```

То же правило действует для спин-блокировки из [Урока 14.7](07-building-a-spin-lock.md): внутри цикла ожидания вызывайте `spin_loop` из своего модуля `sync` (или `loom::thread::yield_now()` под `cfg(loom)`).

Loom находит и **взаимные блокировки**. Когда каждый поток ждёт блокировку, которую держит другой поток, loom останавливается с таким сообщением (вы получите его в упражнении 3):

```text
deadlock; threads = [(Id(0), Blocked(Location(None))), (Id(1), Blocked(Location(None)))]
```

## 7. Держите модели маленькими и знайте ограничения

Число расписаний растёт очень быстро. Вот настоящие числа для исправленного теста `Counter` (`LOOM_LOG=info`):

| Модель | Итерации |
|---|---|
| 2 потока, по одному увеличению | 4 |
| 3 потока, по одному увеличению | 253 |
| 3 потока, `LOOM_MAX_PREEMPTIONS=1` | 10 |
| 3 потока, `LOOM_MAX_PREEMPTIONS=2` | 40 |
| 3 потока, `LOOM_MAX_PREEMPTIONS=3` | 106 |

Один лишний поток превратил 4 в 253. Тест с четырьмя потоками и циклами может идти часами. Правила, которые сохраняют пользу от loom:

- **Два-три потока, по нескольку операций.** Проверяйте самую маленькую историю, которая показывает свойство («два `put`, один `take`»), а не всю программу.
- **Для больших моделей задайте предел вытеснений:** `LOOM_MAX_PREEMPTIONS=2` или `3`. В документации loom сказано, что 2 или 3 достаточно, чтобы поймать большинство ошибок. Проверка тогда уже не полная, но гораздо быстрее.
- **Всегда `--release`.**

Чего loom не умеет:

- Он видит только типы loom. `std::sync::Mutex` или `std::sync::Arc`, спрятанные в вашем коде, для него невидимы, как и расписания вокруг них.
- Он не может смоделировать каждую перестановку, которую разрешает `Relaxed` (например, когда две операции в одном потоке меняются местами), поэтому несколько редких ошибок остаются возможными.
- Загрузки и записи с `SeqCst` он считает `AcqRel`, а это слабее. Это может давать ложные тревоги для кода, которому действительно нужен `SeqCst` (барьер `SeqCst` моделируется правильно).
- Внутри модели нет настоящего времени, случайности и ввода-вывода. Замените их простыми фейками (тестовые двойники были на Этапе 09).

> 🔒 **Безопасность:** ошибки конкурентности в `unsafe`-коде (Уроки 14.7 и 14.8) — это ошибки безопасности памяти: гонка данных на `UnsafeCell` — неопределённое поведение. У loom есть свой `loom::cell::UnsafeCell`, который сообщает о двух несинхронизированных доступах как о «causality violation» (нарушении причинности). Используйте его, а обычные тесты также запускайте под Miri (`cargo +nightly miri test`, [Урок 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md)): Miri проверяет правила `unsafe`, но не перебирает все расписания.

## Частые ошибки

**1. loom в `dev-dependencies`.** Сам код библиотеки (`src/sync.rs`) использует loom, поэтому зависимости только для тестов недостаточно:

```text
error[E0433]: cannot find module or crate `loom` in this scope
 --> src/sync.rs:4:16
  |
4 | pub(crate) use loom::sync::atomic::{AtomicUsize, Ordering};
  |                ^^^^ use of unresolved module or unlinked crate `loom`
```

Исправление: используйте `[target.'cfg(loom)'.dependencies]`, а не `[target.'cfg(loom)'.dev-dependencies]`. (В Уроке 14.6 хватало зависимости для разработки, потому что loom использовал только файл теста.)

**2. Запуск всех тестов с флагом loom.** `RUSTFLAGS="--cfg loom" cargo test --release` без `--test loom` запускает и модульные тесты из `src/lib.rs`. С флагом `Counter` использует атомарные типы loom, но модульный тест не вызывает `loom::model`:

```text
thread 'tests::two_threads_increment_once_each' (5338353) panicked at /Users/you/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/loom-0.7.3/src/rt/scheduler.rs:128:13:
cannot access Loom execution state from outside a Loom model. are you accessing a Loom synchronization primitive from outside a Loom test (a call to `model` or `check`)?
```

Исправление: всегда добавляйте `--test loom` (или поставьте `#[cfg(not(loom))]` на модуль модульных тестов).

**3. Прямой импорт типов `std`.** Если какой-то файл пишет `use std::sync::atomic::AtomicBool;` вместо `use crate::sync::AtomicBool;`, loom не видит эту атомарную переменную и не может перебрать её расписания. Тест может пройти, хотя ошибка на месте. Исправление: каждый тип синхронизации в крейте берётся из `crate::sync`.

**4. `std::hint::spin_loop()` или пустой цикл в модели.** Вы получите «Model exceeded maximum number of branches». Исправление: используйте `spin_loop` из своего модуля `sync`.

## Попробуйте сами

1. Пройдите разделы 1–6 в `code/p14-loom`. Начиная с раздела 3, после каждого шага запускайте обе команды:

   ```bash
   cargo test
   RUSTFLAGS="--cfg loom" cargo test --release --test loom
   ```

2. Ненадолго верните ошибку в `Counter::increment`, посмотрите, как падает loom, затем снова исправьте.
3. В `Mailbox::put` верните только `Release` обратно на `Relaxed`, а `Acquire` в `take` оставьте. Предскажите результат, затем запустите. (Loom снова падает с `left: 0`: release и acquire работают только в паре.)
4. Запустите `cargo clippy --all-targets` и `cargo fmt`, затем сделайте коммит: `git add code/p14-loom && git commit -m "p14-loom: loom tests for counter, mailbox and latch"`.

## Упражнения

### Упражнение 1 — Три потока (легко)

Напишите второй тест loom для исправленного `Counter`: главный поток и два запущенных потока увеличивают счётчик по одному разу; результат должен быть 3. Сколько итераций сообщает loom? Затем запустите его с `LOOM_MAX_PREEMPTIONS=1`.

<details>
<summary>Подсказка</summary>

Запустите два потока в `map` по `0..2`, соберите дескрипторы в `Vec`, увеличьте счётчик в главном потоке, затем дождитесь (`join`) каждого дескриптора.

</details>

<details>
<summary>Ответ</summary>

```rust
#[test]
fn three_threads_increment_once_each() {
    loom::model(|| {
        let counter = Arc::new(Counter::new());
        let handles: Vec<_> = (0..2)
            .map(|_| {
                let other = Arc::clone(&counter);
                thread::spawn(move || other.increment())
            })
            .collect();
        counter.increment();
        for handle in handles {
            handle.join().unwrap();
        }
        assert_eq!(counter.get(), 3);
    });
}
```

```text
 INFO loom::model: Completed in 253 iterations
```

С `LOOM_MAX_PREEMPTIONS=1` получается `Completed in 10 iterations`. Почему это работает: каждый лишний поток умножает число мест, где loom может переключиться. Предел вытеснений отсекает расписания, которым нужно много принудительных переключений.

</details>

### Упражнение 2 — Ровно один победитель (средне)

`Ticket` может забрать только один поток:

```rust
use loom::sync::atomic::{AtomicBool, Ordering};

struct Ticket {
    taken: AtomicBool,
}

impl Ticket {
    fn try_claim(&self) -> bool {
        if !self.taken.load(Ordering::Acquire) {
            self.taken.store(true, Ordering::Release);
            true
        } else {
            false
        }
    }
}
```

Напишите тест loom (в новом файле `tests/loom_ticket.rs` с `#![cfg(loom)]`), где два потока вызывают `try_claim`, и проверьте, что ровно один из них получает `true`. Посмотрите, как тест падает, затем исправьте `try_claim` одной атомарной операцией.

<details>
<summary>Подсказка</summary>

`handle.join().unwrap()` возвращает значение, которое вернуло замыкание потока. «Ровно один `true`» для двух булевых значений — это `a != b`. Для исправления посмотрите на `swap`: он записывает новое значение и возвращает старое за один шаг.

</details>

<details>
<summary>Ответ</summary>

```rust
#![cfg(loom)]

use loom::sync::Arc;
use loom::sync::atomic::{AtomicBool, Ordering};
use loom::thread;

struct Ticket {
    taken: AtomicBool,
}

impl Ticket {
    fn try_claim(&self) -> bool {
        !self.taken.swap(true, Ordering::AcqRel)
    }
}

#[test]
fn exactly_one_thread_wins() {
    loom::model(|| {
        let ticket = Arc::new(Ticket {
            taken: AtomicBool::new(false),
        });
        let other = Arc::clone(&ticket);
        let handle = thread::spawn(move || other.try_claim());
        let main_won = ticket.try_claim();
        let other_won = handle.join().unwrap();
        assert!(
            main_won != other_won,
            "main_won={main_won}, other_won={other_won}"
        );
    });
}
```

С ошибочной версией тест падает с `main_won=true, other_won=true`: оба потока загрузили `false` раньше, чем кто-либо записал `true`. Исправленная версия проходит (`Completed in 4 iterations`). Почему это работает: `swap` читает старое значение и записывает `true` одним неделимым шагом, поэтому старое `false` может увидеть только один поток.

</details>

### Упражнение 3 — Найдите взаимную блокировку (сложно)

Два банковских счёта, у каждого `loom::sync::Mutex<i64>`. `transfer(from, to, amount)` блокирует `from`, затем `to`. Один поток переводит 10 со счёта A на B, а главный поток в это время переводит 20 с B на A. Напишите тест loom, посмотрите на сообщение о взаимной блокировке, затем исправьте `transfer`, чтобы тест проходил и проверял итоговые балансы (A = 110, B = 90).

<details>
<summary>Подсказка</summary>

Исправление из [Урока 14.1](01-why-concurrency-is-hard.md): все потоки должны брать блокировки в одном общем порядке. Дайте каждому счёту `id` и всегда блокируйте сначала меньший id. Затем решите, какой баланс уменьшается.

</details>

<details>
<summary>Ответ</summary>

```rust
#![cfg(loom)]

use loom::sync::{Arc, Mutex};
use loom::thread;

struct Account {
    id: u32,
    balance: Mutex<i64>,
}

fn transfer(from: &Account, to: &Account, amount: i64) {
    // Always lock the account with the smaller id first.
    let (first, second) = if from.id < to.id {
        (from, to)
    } else {
        (to, from)
    };
    let mut first_balance = first.balance.lock().unwrap();
    let mut second_balance = second.balance.lock().unwrap();
    if first.id == from.id {
        *first_balance -= amount;
        *second_balance += amount;
    } else {
        *second_balance -= amount;
        *first_balance += amount;
    }
}

#[test]
fn two_transfers_in_opposite_directions() {
    loom::model(|| {
        let a = Arc::new(Account {
            id: 1,
            balance: Mutex::new(100),
        });
        let b = Arc::new(Account {
            id: 2,
            balance: Mutex::new(100),
        });
        let (a2, b2) = (Arc::clone(&a), Arc::clone(&b));
        let handle = thread::spawn(move || transfer(&a2, &b2, 10));
        transfer(&b, &a, 20);
        handle.join().unwrap();
        assert_eq!(*a.balance.lock().unwrap(), 110);
        assert_eq!(*b.balance.lock().unwrap(), 90);
    });
}
```

Версия, которая сначала блокирует `from`, останавливается с `deadlock; threads = [(Id(0), Blocked(...)), (Id(1), Blocked(...))]`. Исправленная версия проходит за 5 итераций. Почему это работает: при едином порядке блокировок ни один поток не может держать «вторую» блокировку, ожидая «первую», поэтому цикл ждущих потоков невозможен.

</details>

### Упражнение 4 — Проверьте спин-блокировку из Урока 14.7 (сложно)

Loom может проверить блокировку, только если видит и данные внутри неё. Добавьте `SpinLock<T>` в `p14-loom` (`src/spin_lock.rs` и `pub mod spin_lock;` в `src/lib.rs`). Используйте `UnsafeCell` из своего модуля `sync` и дайте блокировке API с замыканием (API — Application Programming Interface, программный интерфейс: методы, которые предлагает тип) вместо guard: `with_lock(&self, f: impl FnOnce(&mut T) -> R) -> R` блокирует, выполняет `f` со значением и разблокирует. Напишите тест loom в `tests/loom_spin.rs`: два потока под блокировкой прибавляют по 1; результат должен быть 2. Затем замените `Acquire` в операции захвата на `Relaxed` и прочитайте отчёт loom.

<details>
<summary>Подсказка</summary>

У `loom::cell::UnsafeCell` нет `get()`; вместо него `with_mut(|ptr: *mut T| …)` даёт сырой указатель на время замыкания, и loom записывает этот доступ. Дайте своей `std`-версии `UnsafeCell` в `src/sync.rs` такой же метод (документация loom показывает эту «прослойку»). Если loom останавливается с «Model exceeded maximum number of branches», хотя вы используете `spin_loop` из своего модуля `sync`: цикл со `swap` из Урока 14.7 *записывает* `true` при каждой неудачной попытке, и loom не может закончить его исследование. Ждите обычными `load` и пробуйте `compare_exchange`, только когда блокировка выглядит свободной (блокировка «test and test-and-set»).

</details>

<details>
<summary>Ответ</summary>

`src/sync.rs` получает `UnsafeCell` из loom и маленькую замену из `std` с тем же API:

```rust
//! One place that decides where the sync types come from.

#[cfg(loom)]
pub(crate) use loom::cell::UnsafeCell;
#[cfg(loom)]
pub(crate) use loom::hint;
#[cfg(loom)]
pub(crate) use loom::sync::atomic::{AtomicBool, AtomicU64, AtomicUsize, Ordering};

#[cfg(not(loom))]
pub(crate) use std::hint;
#[cfg(not(loom))]
pub(crate) use std::sync::atomic::{AtomicBool, AtomicU64, AtomicUsize, Ordering};

/// The same small API as `loom::cell::UnsafeCell`, built on the std type.
#[cfg(not(loom))]
#[derive(Debug)]
pub(crate) struct UnsafeCell<T>(std::cell::UnsafeCell<T>);

#[cfg(not(loom))]
impl<T> UnsafeCell<T> {
    pub(crate) fn new(value: T) -> Self {
        UnsafeCell(std::cell::UnsafeCell::new(value))
    }

    pub(crate) fn with_mut<R>(&self, f: impl FnOnce(*mut T) -> R) -> R {
        f(self.0.get())
    }
}
```

`src/spin_lock.rs`:

```rust
//! The spin lock from Lesson 14.7, with a closure API that loom can check.

use crate::sync::{AtomicBool, Ordering, UnsafeCell, hint};

pub struct SpinLock<T> {
    locked: AtomicBool,
    value: UnsafeCell<T>,
}

// SAFETY: only the thread that holds the lock reaches `value`.
unsafe impl<T: Send> Sync for SpinLock<T> {}

impl<T> SpinLock<T> {
    pub fn new(value: T) -> Self {
        SpinLock {
            locked: AtomicBool::new(false),
            value: UnsafeCell::new(value),
        }
    }

    /// Locks, runs `f` with the value, unlocks.
    pub fn with_lock<R>(&self, f: impl FnOnce(&mut T) -> R) -> R {
        // Wait with plain loads; try to take the lock only when it looks free.
        while self
            .locked
            .compare_exchange(false, true, Ordering::Acquire, Ordering::Relaxed)
            .is_err()
        {
            while self.locked.load(Ordering::Relaxed) {
                hint::spin_loop();
            }
        }
        // SAFETY: we hold the lock, so no other thread uses the value now.
        let result = self.value.with_mut(|ptr| f(unsafe { &mut *ptr }));
        self.locked.store(false, Ordering::Release);
        result
    }
}
```

`tests/loom_spin.rs`:

```rust
#![cfg(loom)]

use loom::sync::Arc;
use loom::thread;
use p14_loom::spin_lock::SpinLock;

#[test]
fn two_threads_increment_under_the_lock() {
    loom::model(|| {
        let lock = Arc::new(SpinLock::new(0_u32));
        let other = Arc::clone(&lock);
        let handle = thread::spawn(move || other.with_lock(|n| *n += 1));
        lock.with_lock(|n| *n += 1);
        handle.join().unwrap();
        assert_eq!(lock.with_lock(|n| *n), 2);
    });
}
```

```text
 INFO loom::model: Completed in 17 iterations
```

С `Ordering::Relaxed` в качестве порядка успеха у `compare_exchange` loom сообщает:

```text
thread 'two_threads_increment_under_the_lock' (6021419) panicked at /Users/you/.cargo/registry/src/index.crates.io-1949cf8c6b5b557f/loom-0.7.3/src/rt/location.rs:115:9:
Causality violation: Concurrent write accesses to `UnsafeCell`.
```

Почему это работает: `Acquire` у успешного `compare_exchange` образует пару с записью `Release` при разблокировке, поэтому доступ второго потока к ячейке происходит после доступа первого. С `Relaxed` такого порядка нет, и `UnsafeCell` из loom видит две неупорядоченные записи: гонку данных, которую loom называет нарушением причинности (causality violation). API с замыканием держит каждый доступ к значению внутри `with_mut`, где loom может его видеть.

</details>

## Вопросы для самопроверки

1. Почему тест, который повторяется в цикле 1000 раз, может проходить, хотя в коде есть гонка?

<details>
<summary>Ответ</summary>

Плохое расписание редкое и зависит от времени. Запуск потока медленный, поэтому потоки в тесте почти никогда не пересекаются. Цикл лишь повторяет самые вероятные расписания, а редкие не пробует.

</details>

2. Что делает `loom::model` с замыканием, которое вы ему передаёте?

<details>
<summary>Ответ</summary>

Запускает его много раз. Каждый запуск следует другому расписанию потоков loom и для атомарных типов может возвращать другие допустимые значения. Он останавливается, когда проверены все расписания (в пределах предела вытеснений), или при первой панике.

</details>

3. Зачем крейту, который использует loom, модуль `sync` и флаг `cfg(loom)`?

<details>
<summary>Ответ</summary>

Типы loom работают только внутри модели loom, поэтому обычная сборка должна использовать `std`. Модуль `sync` переключает все типы синхронизации в одном месте: типы loom, когда включён `--cfg loom`, и типы `std` в остальных случаях.

</details>

4. Что обычно означает сообщение «Model exceeded maximum number of branches»?

<details>
<summary>Ответ</summary>

Цикл ждёт другой поток, но не сообщает об этом loom, поэтому loom исследует вариант «этот поток крутится вечно». Используйте в цикле `loom::hint::spin_loop()` или `loom::thread::yield_now()` (через свой модуль `sync`).

</details>

5. Ваши тесты loom проходят. Доказывает ли это, что код правильный? Назовите одно ограничение.

<details>
<summary>Ответ</summary>

Нет. Loom проверяет только то, что видят его типы и что проверяют ваши утверждения. Ограничения: типы `std`, спрятанные в коде, невидимы; некоторые перестановки `Relaxed` не моделируются; `SeqCst` считается `AcqRel`; модель покрывает только ту маленькую историю, которую вы написали.

</details>

## Итоги

- Ошибки конкурентности прячутся в редких расписаниях. Обычные тесты и стресс-тесты пробуют лишь некоторые расписания, и результат зависит от машины.
- Loom — средство проверки моделей: `loom::model` прогоняет тест по одному разу на каждое возможное расписание, моделирует модель памяти для атомарных типов и находит потерянные обновления, ошибки порядка, бесконечные спин-циклы и взаимные блокировки.
- Подготовьте крейт: `[target.'cfg(loom)'.dependencies]`, запись `check-cfg` в линтах, модуль `sync`, который выбирает типы loom или `std`, и файл `tests/loom.rs` с пометкой `#![cfg(loom)]`.
- Запускайте командой `RUSTFLAGS="--cfg loom" cargo test --release --test loom`; `LOOM_LOG=info` показывает число итераций.
- Спин-циклы должны использовать `spin_loop` или `yield_now` из loom, иначе loom никогда не закончит.
- Держите модели крошечными (два-три потока), для больших используйте `LOOM_MAX_PREEMPTIONS` и сохраняйте обычные тесты и Miri.

## Источники и что почитать

- [loom на docs.rs](https://docs.rs/loom/0.7.3/loom/) — официальная документация: настройка, уступание потоков, отладка с `LOOM_LOG`, контрольные файлы и ограничения.
- [loom на GitHub](https://github.com/tokio-rs/loom) — README со списком неподдерживаемых возможностей.
- [Rust Atomics and Locks — Chapter 3. Memory Ordering](https://marabos.nl/atomics/memory-ordering.html) — книга Mara Bos, бесплатно онлайн; правила, которые моделирует loom.
- [The rustc book — Checking conditional configurations](https://doc.rust-lang.org/rustc/check-cfg.html) — почему предупреждает `unexpected_cfgs` и как `check-cfg` объявляет свой флаг.
- [The Cargo Book — Environment variables](https://doc.rust-lang.org/cargo/reference/environment-variables.html) — `RUSTFLAGS` и `CARGO_TARGET_DIR`.
- [Miri](https://github.com/rust-lang/miri) — интерпретатор, который обнаруживает неопределённое поведение; хороший напарник для loom.

---

[🏠 Начало](../START-HERE.md) · [← 14.8 Неблокирующие структуры данных](08-lock-free-data-structures.md) · [📋 Этап 14](README.md) · [14.10 Как работают future: Future, poll и Waker →](10-how-futures-work.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
