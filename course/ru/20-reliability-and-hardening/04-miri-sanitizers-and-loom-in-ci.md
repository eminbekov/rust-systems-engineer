<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 20.3 Фаззинг подробно](03-fuzzing-in-depth.md) · [📋 Этап 20](README.md) · [20.5 Внедрение сбоев и хаос-тестирование →](05-fault-injection-and-chaos-testing.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/20-reliability-and-hardening/04-miri-sanitizers-and-loom-in-ci.md) <!-- nav -->

# Урок 20.4 — Miri, санитайзеры и loom в CI

> **Этап 20 — Надёжность и защита** · Урок 4 из 11 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Какой вид ошибок находит каждый инструмент: Miri, AddressSanitizer, ThreadSanitizer и loom, и почему проходящие тесты ничего не доказывают о неопределённом поведении.
- Как запускать весь набор тестов под Miri и как не пускать в него медленные тесты с помощью `cfg(miri)`.
- Как собирать и тестировать с AddressSanitizer и ThreadSanitizer на nightly-тулчейне, включая `-Zbuild-std`.
- Как запускать тесты loom с ограниченным поиском и как свести всё это в один workflow CI (непрерывной интеграции).

## Перед началом

- Вы прошли [Урок 20.3 — Фаззинг подробно](03-fuzzing-in-depth.md).
- Вы знаете неопределённое поведение и Miri из [Урока 06.6 — Небезопасный Rust и Miri](../06-rust-advanced/06-unsafe-rust-and-miri.md), ревью `unsafe` — из [Урока 10.12](../10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md), а loom — из [Урока 14.9 — Тестирование конкурентного кода с loom](../14-concurrency-and-async/09-testing-concurrent-code-with-loom.md).
- Нужен nightly-тулчейн с компонентами `miri` и `rust-src`. Nightly с `miri` вы установили в Уроке 06.6; `rust-src` (исходный код стандартной библиотеки, нужен для `-Zbuild-std` в Разделе 5) — новый компонент. Выполните `rustup toolchain install nightly --component miri --component rust-src`: команда добавит недостающее и сохранит то, что уже установлено.
- Создайте пакет для этого урока:

```bash
cd ~/projects/rust
cargo new --lib code/p20-checks
cd code/p20-checks
mkdir tests
```

## Новые слова

| Слово | Значение |
|---|---|
| **инструментирование** (instrumentation) | Дополнительный проверяющий код, который компилятор добавляет в вашу программу, например «перед каждым чтением памяти проверить, что этот адрес разрешён». |
| **ASan** (AddressSanitizer, санитайзер адресов) | Санитайзер, который находит неправильную работу с памятью: чтение за пределами выделенного блока, использование после освобождения, двойное освобождение. Замедляет программу примерно в два раза. |
| **TSan** (ThreadSanitizer, санитайзер потоков) | Санитайзер, который находит гонки данных: два потока одновременно используют одну память без синхронизации, и хотя бы один из них пишет. |
| **LSan** (LeakSanitizer, санитайзер утечек) | Санитайзер, который при завершении программы сообщает о памяти, которую выделили и так и не освободили. |
| **MSan** (MemorySanitizer, санитайзер памяти) | Санитайзер, который находит чтение неинициализированной памяти. Работает только в Linux. |
| **ABI** (Application Binary Interface, двоичный интерфейс приложений) | Низкоуровневые правила того, как скомпилированный код вызывает другой скомпилированный код: куда идут аргументы, как размещаются значения. Код, собранный с разными ABI, смешивать нельзя. |
| **`-Zbuild-std`** | Параметр nightly-версии Cargo, который компилирует стандартную библиотеку из исходников вместе с вашим кодом, с теми же флагами компилятора. |
| **целевой триплет** (target triple) | Имя платформы, под которую вы компилируете, например `aarch64-apple-darwin` (macOS на Apple Silicon) или `x86_64-unknown-linux-gnu` (64-битный Linux на процессорах Intel или AMD (Advanced Micro Devices)). |
| **`cfg(miri)`** | Условие, которое истинно только когда код выполняется под Miri. Используется, чтобы пропустить или уменьшить тесты, слишком медленные для него. |
| **вытеснение** (preemption) | Момент, когда loom даёт поработать другому потоку, прежде чем текущий дойдёт до естественной остановки. Ограничение числа вытеснений ограничивает поиск. |
| **матрица сборки** (build matrix) | Возможность CI запускать одно и то же задание несколько раз с разными значениями, например по разу для каждого санитайзера. |

## 1. Три инструмента, три вопроса

Обычные тесты отвечают на вопрос «даёт ли код правильный результат *в этом запуске*?». Коду с `unsafe` или с потоками нужны вопросы построже, потому что неправильная программа может дать правильный результат случайно.

| Инструмент | На какой вопрос отвечает | Как работает | Цена |
|---|---|---|---|
| **Miri** | Нарушил ли этот запуск какое-либо правило Rust (неопределённое поведение), включая гонки данных? | Интерпретирует вашу программу на модели памяти Rust | Часто в сотни раз медленнее; nightly; нельзя вызывать C через FFI (Foreign Function Interface, интерфейс внешних функций) |
| **ASan** | Читал или писал ли машинный код память, которая ему не принадлежит? | Инструментирование компилятором каждого обращения к памяти | Примерно в 2 раза медленнее; nightly |
| **TSan** | Были ли в этом запуске гонки двух потоков за память? | Инструментирование, которое записывает, какой поток и когда трогал что | В 5–15 раз медленнее; nightly; нужен `-Zbuild-std` |
| **loom** | Есть ли *хоть одно* расписание потоков, при котором мой тест падает? | Запускает тест снова и снова, по разу на каждое возможное расписание | Растёт очень быстро с числом потоков и шагов |

Miri и санитайзеры проверяют **пути, которые выполняют ваши тесты**, с одним расписанием за запуск (ASan, TSan) или несколькими (Miri). loom проверяет **все расписания**, но только маленьких моделей, которые вы пишете с типами loom. Они дополняют друг друга, и ни один не заменяет тесты: они делают тесты строже.

## 2. Код, который проходит свои тесты

Положите это в `src/lib.rs`:

```rust
//! Small pieces of unsafe and concurrent code, checked by Miri, sanitizers and loom.

/// Adds the values in pairs: `[1, 2, 3, 4]` gives `[3, 7]`.
/// A last value without a partner stays alone: `[1, 2, 3]` gives `[3, 3]`.
pub fn pair_sums(values: &[u32]) -> Vec<u32> {
    let mut sums = Vec::with_capacity(values.len().div_ceil(2));
    let start = values.as_ptr();
    let mut i = 0;
    while i < values.len() {
        // SAFETY: (wrong!) we assume that every value has a partner at `i + 1`.
        let sum = unsafe { *start.add(i) + *start.add(i + 1) };
        sums.push(sum);
        i += 2;
    }
    sums
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn even_number_of_values() {
        assert_eq!(pair_sums(&[1, 2, 3, 4]), vec![3, 7]);
    }

    #[test]
    fn odd_number_of_values() {
        let values = vec![1, 2, 3];
        assert_eq!(pair_sums(&values), vec![3, 3]);
    }
}
```

**Что происходит:**

1. Цикл читает `values[i]` и `values[i + 1]` через сырой указатель (Урок 06.6), без проверок границ.
2. При нечётном числе значений последний шаг читает один элемент **за концом** среза. Это неопределённое поведение. Комментарий `// SAFETY:` формулирует неверное предположение — именно то, что должен заметить ревьюер (Урок 10.12).
3. Второй тест даже проверяет нечётный случай.

```bash
cargo test -q
cargo test -q --release
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
...
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Оба проходят, и в отладочной, и в релизной сборке. В памяти сразу за тремя значениями случайно оказался `0`, поэтому `3 + 0` дало «правильный» ответ. На другой машине, с другим аллокатором или другой версией компилятора там может быть любое число или аварийное завершение. Именно для этого существуют инструменты ниже.

## 3. Miri на всём наборе тестов

В Уроке 06.6 вы запускали отдельные программы через `cargo +nightly miri run`. В настоящем проекте под Miri запускают **весь набор тестов**:

```bash
cargo +nightly miri test
```

```text
running 2 tests
test tests::even_number_of_values ... ok
test tests::odd_number_of_values ... error: Undefined Behavior: memory access failed: attempting to access 4 bytes, but got alloc47039+0xc which is at or beyond the end of the allocation of size 12 bytes
  --> src/lib.rs:11:44
   |
11 |         let sum = unsafe { *start.add(i) + *start.add(i + 1) };
   |                                            ^^^^^^^^^^^^^^^^^ Undefined Behavior occurred here
   |
   = help: this indicates a bug in the program: it performed an invalid operation, and caused Undefined Behavior
   = help: see https://doc.rust-lang.org/nightly/reference/behavior-considered-undefined.html for further information
help: alloc47039 was allocated here:
  --> src/lib.rs:29:22
   |
29 |         let values = vec![1, 2, 3];
   |                      ^^^^^^^^^^^^^
...
error: test failed, to rerun pass `--lib`
```

**Как это читать:** выделенный блок — 12 байт (три значения `u32`), а программа попыталась прочитать 4 байта по смещению `0xc` (12), ровно на конце. Miri показывает строку неправильного чтения *и* строку, где память была выделена. Первый тест под Miri прошёл: чётное число значений никогда не доходит до бага. Miri проверяет только пути, которые выполняют ваши тесты.

Когда набор тестов растёт, появляются две практические проблемы:

- **Некоторые тесты слишком медленные для Miri.** Тест, который обрабатывает миллион значений, нативно занимает миллисекунды, а под Miri — минуты. Пометьте его, чтобы Miri его пропускал:

```rust
    #[test]
    #[cfg_attr(miri, ignore)] // a million values: seconds natively, many minutes under Miri
    fn a_million_values() {
        let values = vec![1; 1_000_000];
        assert_eq!(pair_sums(&values).len(), 500_000);
    }
```

  `cfg_attr(miri, ignore)` означает «добавить атрибут `#[ignore]`, только когда `cfg(miri)` истинно». Под Miri строка результата тогда говорит `1 ignored`. Часто лучше оставить тест, но уменьшить его под Miri с помощью `cfg!(miri)` — булева значения, которое можно использовать в обычном коде (Упражнение 3).
- **Некоторые тесты вызывают код, который Miri не может выполнить**, например библиотеки на C через FFI (Урок 06.10) или сетевые вызовы. Пропускайте их так же.

> 💡 **Совет:** `MIRIFLAGS` передаёт параметры в Miri. `MIRIFLAGS="-Zmiri-many-seeds=0..16" cargo +nightly miri test` запускает каждый тест с 16 разными случайными начальными значениями (seed), что меняет расписания потоков и адреса, которые выбирает Miri. Используйте это для тестов с потоками.

## 4. AddressSanitizer: ошибки памяти в машинном коде

Miri точен, но медленен и не может выполнять код на C. **ASan** проверяет настоящую скомпилированную программу почти на полной скорости. Он включается флагом компилятора на nightly:

```bash
RUSTFLAGS="-Zsanitizer=address" cargo +nightly test -q --target aarch64-apple-darwin
```

```text
running 2 tests
=================================================================
==24999==ERROR: AddressSanitizer: heap-buffer-overflow on address 0x60200000201c at pc 0x000100e44374 bp 0x00016f3d21b0 sp 0x00016f3d21a8
READ of size 4 at 0x60200000201c thread T2
    #0 0x000100e44370 in _RNvCskEp1fDRXMd0_10p20_checks9pair_sums lib.rs:11
    #1 0x000100e42d90 in _RNvNtCskEp1fDRXMd0_10p20_checks5testss_20odd_number_of_values lib.rs:30
...
0x60200000201c is located 0 bytes after 12-byte region [0x602000002010,0x60200000201c)
allocated by thread T2 here:
...
    #7 0x000100e42b54 in _RNvNtCskEp1fDRXMd0_10p20_checks5testss_20odd_number_of_values lib.rs:29
```

**Что происходит:**

1. `RUSTFLAGS` передаёт компилятору дополнительные флаги для этой одной команды. `-Zsanitizer=address` добавляет проверки ASan; флаги `-Z` существуют только в nightly.
2. `--target aarch64-apple-darwin` явно называет **целевой триплет** вашего Mac. При явной цели Cargo применяет `RUSTFLAGS` только к вашему коду и его зависимостям, но не к скриптам сборки и процедурным макросам, которые выполняются на машине сборки и не должны инструментироваться. В CI на Linux триплет — `x86_64-unknown-linux-gnu`.
3. Отчёт: **heap-buffer-overflow** (переполнение буфера в куче), чтение 4 байт «0 bytes after a 12-byte region». Тот же баг, что нашёл Miri, теперь найден в скомпилированной программе.
4. Имена вроде `_RNvCskEp1fDRXMd0_10p20_checks9pair_sums` — *искажённые* (mangled) имена символов: закодированная компилятором форма `p20_checks::pair_sums`. Читайте понятную часть ближе к концу, а также файл и строку.

В Linux тем же флагом доступны родственные санитайзеры: `-Zsanitizer=leak` (LSan, утечки памяти) и `-Zsanitizer=memory` (MSan, чтение неинициализированной памяти; нужен `-Zbuild-std`, см. ниже).

> ⚡ **Производительность:** ASan замедляет программу примерно в два раза и использует больше памяти, поэтому с ним можно запускать *весь* набор тестов и даже фаззеры: `cargo fuzz` по умолчанию включает ASan (Урок 09.8). Miri для этого слишком медленный; используйте его на модульных тестах кода, в котором есть `unsafe`.

Теперь исправьте баг. Лучшее исправление `unsafe`-кода — часто вообще без `unsafe`. `chunks(2)` даёт срезы по два значения и последний срез из одного значения для нечётной длины:

```rust
pub fn pair_sums(values: &[u32]) -> Vec<u32> {
    values.chunks(2).map(|pair| pair.iter().sum()).collect()
}
```

Снова выполните `cargo +nightly miri test`: оба теста проходят.

## 5. ThreadSanitizer: гонки данных

Создайте `src/counter.rs` с «быстрым» счётчиком запросов, который обходится без синхронизации, и добавьте `pub mod counter;` в начало `src/lib.rs`:

```rust
//! A request counter shared by many threads.

use std::cell::UnsafeCell;

/// BUG: a "fast" counter without any synchronization.
pub struct RacyCounter {
    value: UnsafeCell<u64>,
}

// SAFETY: (wrong!) we claim that sharing between threads is fine.
unsafe impl Sync for RacyCounter {}

impl RacyCounter {
    pub fn new() -> Self {
        RacyCounter {
            value: UnsafeCell::new(0),
        }
    }

    pub fn increment(&self) {
        // SAFETY: (wrong!) two threads can be here at the same time.
        unsafe { *self.value.get() += 1 };
    }

    pub fn get(&self) -> u64 {
        // SAFETY: (wrong!) another thread may be writing right now.
        unsafe { *self.value.get() }
    }
}

impl Default for RacyCounter {
    fn default() -> Self {
        Self::new()
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn two_threads_count_requests() {
        let counter = RacyCounter::new();
        thread::scope(|s| {
            s.spawn(|| {
                for _ in 0..100 {
                    counter.increment();
                }
            });
            s.spawn(|| {
                for _ in 0..100 {
                    counter.increment();
                }
            });
        });
        assert_eq!(counter.get(), 200);
    }
}
```

**Что происходит:**

1. `UnsafeCell` (Урок 06.4) позволяет менять значение через общую ссылку; `unsafe impl Sync` (Урок 06.6) обещает компилятору, что это безопасно между потоками. Обещание ложное: `+= 1` — это чтение и запись, и два потока могут их перемешать.
2. `thread::scope` (Урок 14.4) запускает два потока, которые заимствуют `counter`, и ждёт обоих.

Запустите тест обычным образом пять раз: `for i in 1 2 3 4 5; do cargo test -q counter; done`. Он проходит каждый раз. Сто увеличений выполняются так быстро, что два потока редко пересекаются. Теперь запустите ThreadSanitizer так же, как ASan:

```bash
RUSTFLAGS="-Zsanitizer=thread" cargo +nightly test -q --target aarch64-apple-darwin counter
```

```text
error: mixing `-Zsanitizer` will cause an ABI mismatch in crate `p20_checks`
 |
 = help: the `-Zsanitizer` flag modifies the ABI so Rust crates compiled with different values of this flag cannot be used together safely
 = note: `-Zsanitizer=thread` in this crate is incompatible with `-Zsanitizer` being unset in dependency `std`
 = help: unset `-Zsanitizer` in this crate or set `-Zsanitizer=thread` in `std`
...
```

TSan должен видеть *каждое* обращение к памяти, включая обращения внутри стандартной библиотеки, а стандартная библиотека, которая поставляется с тулчейном, собрана без TSan. Они не подходят друг другу: у них разный **ABI**. Решение — собрать стандартную библиотеку из исходников с тем же флагом, это и делает `-Zbuild-std` (нужен компонент `rust-src`):

```bash
RUSTFLAGS="-Zsanitizer=thread" cargo +nightly test -q -Zbuild-std --target aarch64-apple-darwin counter
```

```text
running 1 test
==================
WARNING: ThreadSanitizer: data race (pid=26447)
  Read of size 8 at 0x00016b355368 by thread T3:
    #0 _RNvMs_NtCs1JKWEFB9EXy_10p20_checks7counterNtB4_11RacyCounter9increment counter.rs:22 (p20_checks-a99037034d792b30:arm64+0x100003450)
...
  Previous write of size 8 at 0x00016b355368 by thread T2:
    #0 _RNvMs_NtCs1JKWEFB9EXy_10p20_checks7counterNtB4_11RacyCounter9increment counter.rs:22 (p20_checks-a99037034d792b30:arm64+0x1000034d4)
...
SUMMARY: ThreadSanitizer: data race counter.rs:22 in _RNvMs_NtCs1JKWEFB9EXy_10p20_checks7counterNtB4_11RacyCounter9increment
==================
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 2 filtered out; finished in 0.17s

ThreadSanitizer: reported 1 warnings
error: test failed, to rerun pass `--lib`
```

**Как это читать:** поток T3 прочитал счётчик в `counter.rs:22`, когда поток T2 записал те же 8 байт в той же строке, и между этими действиями нет ничего, что их упорядочивает. Это гонка данных. Обратите внимание на конец: собственное утверждение теста прошло (`1 passed`), но TSan заставил процесс упасть при завершении, поэтому команда завершается ошибкой, и задание CI тоже упало бы. Первая сборка занимает минуту-две, потому что компилируется стандартная библиотека.

Miri находит ту же гонку без всяких дополнительных флагов:

```bash
cargo +nightly miri test counter
```

```text
test counter::tests::two_threads_count_requests ... error: Undefined Behavior: Data race detected between (1) non-atomic write on thread `unnamed-2` and (2) non-atomic read on thread `unnamed-3` at alloc42582
  --> src/counter.rs:22:18
   |
22 |         unsafe { *self.value.get() += 1 };
   |                  ^^^^^^^^^^^^^^^^^^^^^^ (2) just happened here
...
```

Исправление — атомарный счётчик из Урока 14.5. Замените `RacyCounter` и его блоки `impl` (тест оставьте, но пусть он использует `AtomicCounter::new()`):

```rust
/// The fixed counter: one atomic read-modify-write per increment.
pub struct AtomicCounter {
    value: crate::sync::AtomicU64,
}

impl AtomicCounter {
    pub fn new() -> Self {
        AtomicCounter {
            value: crate::sync::AtomicU64::new(0),
        }
    }

    pub fn increment(&self) {
        self.value.fetch_add(1, crate::sync::Ordering::Relaxed);
    }

    pub fn get(&self) -> u64 {
        self.value.load(crate::sync::Ordering::Relaxed)
    }
}

impl Default for AtomicCounter {
    fn default() -> Self {
        Self::new()
    }
}
```

`crate::sync` — модуль-переключатель из Урока 14.9; мы добавим его в следующем разделе, чтобы loom мог подменить атомарный тип.

## 6. loom в CI

Настройте loom точно как в Уроке 14.9. Добавьте в `Cargo.toml`:

```toml
[target.'cfg(loom)'.dependencies]
loom = "0.7"

[lints.rust]
unexpected_cfgs = { level = "warn", check-cfg = ["cfg(loom)"] }
```

Создайте `src/sync.rs` и добавьте `mod sync;` в `src/lib.rs` рядом с `pub mod counter;`:

```rust
//! One place that decides where the sync types come from.

#[cfg(loom)]
pub(crate) use loom::sync::atomic::{AtomicU64, Ordering};

#[cfg(not(loom))]
pub(crate) use std::sync::atomic::{AtomicU64, Ordering};
```

В `src/counter.rs` замените строку `#[cfg(test)]` над модулем тестов на `#[cfg(all(test, not(loom)))]`: этот тест использует `std::thread`, который не должен выполняться внутри сборки loom. Затем создайте `tests/loom.rs`:

```rust
#![cfg(loom)]

use loom::sync::Arc;
use loom::thread;
use p20_checks::counter::AtomicCounter;

#[test]
fn two_threads_never_lose_an_increment() {
    loom::model(|| {
        let counter = Arc::new(AtomicCounter::new());
        let other = Arc::clone(&counter);
        let handle = thread::spawn(move || other.increment());
        counter.increment();
        handle.join().unwrap();
        assert_eq!(counter.get(), 2);
    });
}
```

```bash
LOOM_MAX_PREEMPTIONS=3 RUSTFLAGS="--cfg loom" CARGO_TARGET_DIR=target/loom cargo test --release --test loom
```

```text
running 1 test
test two_threads_never_lose_an_increment ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Что происходит:**

1. `RUSTFLAGS="--cfg loom"` переключает модуль `sync` на типы loom, а `--test loom` запускает только тестовый файл loom (Урок 14.9).
2. `LOOM_MAX_PREEMPTIONS=3` говорит loom исследовать только расписания, в которых не больше трёх принудительных переключений потоков. Большинству настоящих багов нужно одно-два, а без лимита поиск для модели побольше может идти часами. В CI лимит делает время задания предсказуемым.
3. `CARGO_TARGET_DIR=target/loom` держит сборку loom отдельно от обычной, чтобы ни одной не приходилось всё пересобирать (совет из Урока 14.9).

> 🧪 **Тестирование:** loom не нужен nightly, и он находит логические гонки между *атомарными* операциями, о которых не сообщают ни TSan, ни Miri, потому что атомарные операции никогда не образуют гонку в смысле TSan. TSan и Miri находят гонки на *неатомарной* памяти. Вам нужны проверки обоих видов.

## 7. Один workflow для всех глубоких проверок

Эти проверки медленнее, чем `cargo test`, поэтому они получают свой workflow рядом с контролем качества из Урока 09.12. Сохраните его как `.github/workflows/deep-checks.yml`:

```yaml
name: Deep checks

on:
  push:
    branches: [main]
  pull_request:

permissions:
  contents: read

jobs:
  miri:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install nightly --profile minimal --component miri
      - run: cargo +nightly miri setup
      - run: cargo +nightly miri test

  sanitizers:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    strategy:
      fail-fast: false
      matrix:
        sanitizer: [address, thread]
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install nightly --profile minimal --component rust-src
      - name: Test with the ${{ matrix.sanitizer }} sanitizer
        env:
          RUSTFLAGS: -Zsanitizer=${{ matrix.sanitizer }}
        run: cargo +nightly test -Zbuild-std --target x86_64-unknown-linux-gnu --lib --tests

  loom:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v7
      - name: Explore thread schedules with loom
        env:
          RUSTFLAGS: --cfg loom
          LOOM_MAX_PREEMPTIONS: 3
        run: cargo test --release --test loom
```

**Что происходит:**

1. Три задания выполняются параллельно на свежих машинах с Linux. Каждое останавливается максимум через 30 минут.
2. `cargo +nightly miri setup` один раз готовит собственную копию стандартной библиотеки для Miri, чтобы шаг с тестами не смешивал эту работу со своим выводом.
3. Задание `sanitizers` — **матрица сборки**: оно выполняется дважды, один раз с `address` и один раз с `thread`. `fail-fast: false` позволяет второму запуску продолжиться, когда первый падает, чтобы вы увидели оба отчёта.
4. В CI на Linux задание использует `-Zbuild-std` для обоих санитайзеров, как рекомендует официальная документация, и `--lib --tests`, чтобы с санитайзером собирались только модульные и интеграционные тесты (документационные тесты компилирует другой инструмент, который не получает `RUSTFLAGS`).
5. Задание loom выполняется на stable Rust: loom — обычный крейт.

```bash
actionlint .github/workflows/deep-checks.yml && echo "actionlint: no problems"
```

```text
actionlint: no problems
```

(Сам workflow выполняется на машинах GitHub с Linux, которые этот курс использовать не может. Каждая команда из него была выполнена в macOS с целью `aarch64-apple-darwin`; в Linux меняется только целевой триплет.)

> 🔒 **Безопасность:** каждый блок `unsafe` — это обещание, что человек всё проверил. Эти инструменты проверяют обещание на каждом pull request, ещё долго после того, как ревьюер занялся другим. Для крейтов с `unsafe`-кодом зелёное задание Miri — один из самых сильных признаков качества, который вы можете показать.

## Частые ошибки

**1. «Мои тесты проходят, значит, небезопасный код правильный».** Раздел 2 показал тест, который проходит в отладочной и релизной сборках, читая при этом за концом буфера. Это находит только инструмент, проверяющий правила.

**2. Забыть `--target` с санитайзерами.** Без явной цели `RUSTFLAGS` применяется и к скриптам сборки, и к процедурным макросам, которые тогда не собираются или падают непонятным образом. Всегда передавайте `--target <триплет>`.

**3. TSan без `-Zbuild-std`.** Вы получаете `error: mixing -Zsanitizer will cause an ABI mismatch` (раздел 5). Добавьте `-Zbuild-std` и убедитесь, что для nightly установлен `rust-src`.

**4. Запускать под Miri всё подряд.** Набор тестов, который нативно занимает десять секунд, под Miri может идти часами. Помечайте медленные тесты `#[cfg_attr(miri, ignore)]` или уменьшайте их через `cfg!(miri)` и запускайте Miri на коде, в котором действительно есть `unsafe`.

## Попробуйте сами

1. Создайте `code/p20-checks` с ошибочной `pair_sums` и двумя её тестами. Запустите `cargo test` и `cargo +nightly miri test`.
2. Выполните команду с ASan и найдите в отчёте слова `heap-buffer-overflow` и `lib.rs:11`.
3. Исправьте `pair_sums` через `chunks(2)` и добавьте тест с `#[cfg_attr(miri, ignore)]`. Снова запустите Miri: `2 passed; 0 failed; 1 ignored`.
4. Добавьте `src/counter.rs` с `RacyCounter`. Запустите тест обычным образом пять раз, затем с TSan (сначала без `-Zbuild-std`, чтобы увидеть ошибку), затем под Miri.
5. Замените его на `AtomicCounter`, добавьте `src/sync.rs` и `tests/loom.rs` и выполните команду loom.
6. Сохраните workflow и проверьте его через `actionlint`. Затем добавьте `leak` в матрицу санитайзеров (LeakSanitizer доступен на `x86_64-unknown-linux-gnu`) и проверьте снова.

## Упражнения

### Упражнение 1 — Выберите инструмент (легко)

Какой инструмент (Miri, ASan, TSan, loom) вы использовали бы первым для каждой проблемы и почему?

1. Библиотека на C, которую вы вызываете через FFI, иногда падает с испорченной кучей.
2. Неблокирующий стек на `AtomicPtr` под нагрузкой иногда теряет элемент.
3. Маленькая `unsafe`-функция, которая превращает `&[u8]` в `&[u32]` приведением указателя.

<details>
<summary>Подсказка</summary>

Спросите: участвует ли код на C? Атомарны ли операции? Код маленький и на чистом Rust?

</details>

<details>
<summary>Ответ</summary>

1. **ASan.** Miri не может выполнять код на C; ASan инструментирует машинный код и почти на полной скорости ловит переполнения кучи и использование после освобождения.
2. **loom.** Баг — логическая гонка между атомарными операциями; loom исследует все расписания маленькой модели. (Miri с множеством начальных значений тоже может помочь.)
3. **Miri.** Маленький `unsafe`-код на чистом Rust: Miri точно проверяет выравнивание, границы и корректность получившихся ссылок.

</details>

### Упражнение 2 — Тест, который доходит до бага (средне)

В этой функции ошибка на единицу, но её тесты проходят, даже под Miri. Напишите ещё один тест, после которого `cargo +nightly miri test` сообщит о неопределённом поведении. Затем исправьте функцию.

```rust
/// Returns the value at `index`, or 0 if there is no such value.
pub fn get_or_zero(values: &[u32], index: usize) -> u32 {
    if index <= values.len() {
        // SAFETY: we checked that `index` is inside `values`.
        unsafe { *values.as_ptr().add(index) }
    } else {
        0
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn reads_an_existing_value() {
        assert_eq!(get_or_zero(&[5, 6], 0), 5);
    }

    #[test]
    fn far_away_index_gives_zero() {
        assert_eq!(get_or_zero(&[5, 6], 9), 0);
    }
}
```

<details>
<summary>Подсказка</summary>

Проверьте граничный случай (Урок 05.6): какой индекс находится ровно на границе проверки?

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod edge_tests {
    use super::*;

    #[test]
    fn index_equal_to_len_gives_zero() {
        let values = vec![5, 6];
        assert_eq!(get_or_zero(&values, 2), 0);
    }
}
```

С обычным `cargo test` этот тест тоже проходит (байт после вектора случайно оказался 0). Под Miri:

```text
test edge_tests::index_equal_to_len_gives_zero ... error: Undefined Behavior: memory access failed: attempting to access 4 bytes, but got alloc40367+0x8 which is at or beyond the end of the allocation of size 8 bytes
  --> src/lib.rs:5:18
```

Исправление — `if index < values.len()`. После него все три теста проходят под Miri. Почему это работает: Miri может судить только о выполняемых путях; тест граничного случая заставляет выполниться плохой путь. (Безопасной версии, `values.get(index).copied().unwrap_or(0)`, `unsafe` не нужен вовсе.)

</details>

### Упражнение 3 — Меньше под Miri вместо пропуска (средне)

Вместо того чтобы пропускать тест с миллионом значений под Miri, напишите тест, который под Miri использует 1 000 значений, а иначе 1 000 000, и проверяет, что каждая сумма равна 2.

<details>
<summary>Подсказка</summary>

`cfg!(miri)` — выражение типа `bool`: `if cfg!(miri) { … } else { … }`.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn many_values_but_fewer_under_miri() {
        let count = if cfg!(miri) { 1_000 } else { 1_000_000 };
        let values = vec![1; count];
        let sums = pair_sums(&values);
        assert_eq!(sums.len(), count / 2);
        assert!(sums.iter().all(|&sum| sum == 2));
    }
```

```text
test tests::many_values_but_fewer_under_miri ... ok
```

Почему это работает: тест по-прежнему выполняется под Miri, поэтому путь кода всё так же проверяется на неопределённое поведение, но со входом, достаточно маленьким, чтобы закончиться примерно за секунду.

</details>

## Вопросы для самопроверки

1. Почему тест с неопределённым поведением может проходить и в отладочной, и в релизной сборке?

<details>
<summary>Ответ</summary>

Неопределённое поведение не обязано приводить к падению. Программа может прочитать память, в которой случайно лежит безобидное значение. Результат — удача, и он может измениться на другой машине, с другим аллокатором или версией компилятора.

</details>

2. Что проверяет Miri, чего не проверяет обычный тест, и каково его главное ограничение?

<details>
<summary>Ответ</summary>

Он проверяет каждое обращение к памяти и каждое заимствование выполняемого кода на соответствие правилам Rust (границы, выравнивание, алиасинг, инициализация, гонки данных). Ограничения: он проверяет только выполняемые пути, он очень медленный и не может выполнять чужой код, например библиотеки на C.

</details>

3. Зачем сборке с санитайзером нужен `--target <триплет>`?

<details>
<summary>Ответ</summary>

Чтобы `RUSTFLAGS` с `-Zsanitizer=…` применялся только к коду, собираемому для цели, а не к скриптам сборки и процедурным макросам, которые выполняются на машине сборки во время компиляции.

</details>

4. Что означает ошибка «mixing -Zsanitizer will cause an ABI mismatch» и как её исправить?

<details>
<summary>Ответ</summary>

Ваш крейт собран с санитайзером, а заранее собранная стандартная библиотека — нет, и они несовместимы. Соберите стандартную библиотеку из исходников с теми же флагами: добавьте `-Zbuild-std` (с компонентом `rust-src`).

</details>

5. TSan сообщил о гонке, но напечатал `1 passed`. Проходит ли задание CI?

<details>
<summary>Ответ</summary>

Нет. После предупреждений TSan заставляет тестовый процесс упасть при завершении, поэтому `cargo test` возвращает ошибку, и задание падает.

</details>

6. Какой инструмент находит потерянное обновление между двумя вызовами `load` и `store` на `AtomicU64`?

<details>
<summary>Ответ</summary>

loom (и, возможно, Miri с множеством начальных значений). TSan о нём не сообщает, потому что атомарные операции никогда не образуют гонку данных; баг — логическая гонка между корректными атомарными операциями.

</details>

## Итоги

- Проходящие тесты ничего не доказывают о неопределённом поведении: неправильный `unsafe`-код часто случайно даёт правильный ответ.
- **Miri** (`cargo +nightly miri test`) точно проверяет выполняемые пути, включая гонки данных; медленные тесты пропускайте или уменьшайте через `#[cfg_attr(miri, ignore)]` и `cfg!(miri)`.
- **ASan** (`RUSTFLAGS="-Zsanitizer=address" cargo +nightly test --target <триплет>`) находит ошибки памяти в скомпилированном коде, в том числе на C, примерно на половине скорости.
- **TSan** находит гонки данных в скомпилированном коде и требует `-Zbuild-std`; сообщив о гонке, он проваливает запуск.
- **loom** исследует все расписания маленькой модели; `LOOM_MAX_PREEMPTIONS` ограничивает поиск в CI.
- Выносите глубокие проверки в отдельный workflow CI: задание Miri, матрица санитайзеров и задание loom, каждое с тайм-аутом, проверенные через `actionlint`.

## Источники и что почитать

- [Miri](https://github.com/rust-lang/miri) — официальный README: установка, `cargo miri test`, параметры `MIRIFLAGS` и советы для CI.
- [The Unstable Book — sanitizer](https://doc.rust-lang.org/nightly/unstable-book/compiler-flags/sanitizer.html) — все санитайзеры, которые поддерживает Rust, на каких целях и рекомендация про `-Zbuild-std`.
- [AddressSanitizer](https://clang.llvm.org/docs/AddressSanitizer.html) и [ThreadSanitizer](https://clang.llvm.org/docs/ThreadSanitizer.html) — как работают санитайзеры LLVM (набора инструментов компилятора, на котором построен Rust) и как читать их отчёты.
- [loom — документация](https://docs.rs/loom/0.7.3/loom/) — `loom::model`, `LOOM_MAX_PREEMPTIONS` и другие переменные окружения.
- [The Cargo Book — Unstable features: build-std](https://doc.rust-lang.org/cargo/reference/unstable.html) — что делает `-Zbuild-std` и его параметры.

---

[🏠 Начало](../START-HERE.md) · [← 20.3 Фаззинг подробно](03-fuzzing-in-depth.md) · [📋 Этап 20](README.md) · [20.5 Внедрение сбоев и хаос-тестирование →](05-fault-injection-and-chaos-testing.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
