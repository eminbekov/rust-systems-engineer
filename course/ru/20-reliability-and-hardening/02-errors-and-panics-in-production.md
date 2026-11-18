<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 20.1 Что такое надёжность](01-what-reliability-means.md) · [📋 Этап 20](README.md) · [20.3 Фаззинг подробно →](03-fuzzing-in-depth.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/20-reliability-and-hardening/02-errors-and-panics-in-production.md) <!-- nav -->

# Урок 20.2 — Ошибки и паники в продакшене

> **Этап 20 — Надёжность и защита** · Урок 2 из 11 · ⏱ около 135 минут

## Что вы узнаете

- Чем ожидаемая ошибка отличается от бага и почему паниковать должны только баги.
- Что паника делает с потоком, с `Mutex` и с задачей Tokio и как не дать одной панике уронить весь сервис.
- Как сделать каждую панику заметной с помощью *хука паники* (panic hook): одна структурированная строка лога и счётчик.
- Когда выбирать `panic = "unwind"`, а когда `panic = "abort"`, и что каждый вариант означает для очистки ресурсов и перезапусков.
- Как линты Clippy не пускают `unwrap`, `expect` и индексацию в продакшен-код.

## Перед началом

- Вы прошли [Урок 20.1 — Что такое надёжность](01-what-reliability-means.md).
- Вы знаете `Result`, `?` и `panic!` из [Урока 04.10 — Обработка ошибок](../04-structs-enums-collections-errors/10-error-handling.md) и то, почему злоумышленник любит паники, из [Урока 10.12 — Безопасность памяти и ревью небезопасного кода](../10-security-fundamentals/12-memory-safety-and-reviewing-unsafe.md).
- Вы знаете потоки и `Mutex` из [Урока 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md), а задачи Tokio — из [Урока 14.13 — Tokio подробно](../14-concurrency-and-async/13-tokio-tasks-and-channels.md).
- Создайте пакет для этого урока:

```bash
cd ~/projects/rust
cargo new --lib code/p20-panics
cd code/p20-panics
cargo add tokio@1 --features rt-multi-thread,macros
mkdir src/bin
```

## Новые слова

| Слово | Значение |
|---|---|
| **ожидаемая ошибка** (expected error) | Проблема, которая — обычная часть работы: отсутствующий файл, плохой ввод пользователя, тайм-аут. Её возвращают как `Err` и обрабатывают. |
| **баг** (bug) | Ошибка в программе: случилось то, что, по предположению кода, случиться не может. В Rust это обычно проявляется как паника. |
| **раскрутка стека** (unwinding) | Что Rust по умолчанию делает после паники: идёт назад по стеку паникующего потока и вызывает `Drop` для каждого значения на пути. |
| **аварийное прерывание** (abort) | Немедленное завершение всего процесса, без раскрутки стека и без вызова `Drop`. |
| **отравленная блокировка** (poisoned lock) | `Mutex`, владелец которого запаниковал, удерживая его. Последующие вызовы `lock()` возвращают `Err(PoisonError)`, предупреждая, что данные могли остаться изменёнными наполовину. |
| **полезная нагрузка паники** (panic payload) | Значение, которое несёт паника, обычно сообщение в виде `&str` или `String`. |
| **хук паники** (panic hook) | Функция, которую Rust вызывает при каждой панике, до начала раскрутки стека. Хук по умолчанию печатает сообщение `thread '…' panicked at …`. |
| **`catch_unwind`** | Функция из `std::panic`, которая выполняет замыкание и возвращает `Err(payload)` вместо того, чтобы выпустить панику наружу. |
| **изоляция** (isolation) | Разделение частей программы, чтобы отказ одной части не распространялся на другие. |
| **супервизор** (supervisor) | Программа, которая запускает другую программу и перезапускает её, когда та завершается, например systemd (Урок 11.6) или Kubernetes (Этап 21). |
| **линт** (lint) | Автоматическая проверка исходного кода на шаблоны, которые часто оказываются ошибками. Clippy — главный линтер Rust. |

## 1. Ошибки и баги — разные вещи

Сервис сталкивается с двумя видами проблем:

- **Ожидаемые ошибки.** Пользователь присылает слишком длинное имя. Запрос к базе данных превышает тайм-аут. Файла нет. Это часть нормальной жизни, поэтому код возвращает `Err(...)`, вызывающий решает, что делать, а пользователь получает понятный ответ, например «400 Bad Request».
- **Баги.** Код читает элемент 4 из списка с четырьмя элементами. Он делит на счётчик, равный нулю. Случилось то, что «никогда не может случиться». Rust останавливает текущий поток **паникой**, потому что продолжение с невозможными данными могло бы сделать хуже.

Отсюда правило для продакшен-кода: **каждая проблема, которая может случиться при правильном использовании, — это `Err`. Паника означает, что есть баг, который нужно исправить.** В Уроке 04.10 вы изучили инструменты для первого вида. Этот урок — о втором виде: баги *будут* попадать в продакшен, и надёжный сервис должен их пережить, сообщить о них и продолжать обслуживать всех остальных.

## 2. Что делает паника

По умолчанию паника **раскручивает стек** только того потока, в котором она случилась. Остальные потоки продолжают работать. Создайте `src/bin/thread_panic.rs`:

```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let orders = Arc::new(Mutex::new(vec![10, 20, 30]));

    let worker_orders = Arc::clone(&orders);
    let worker = thread::spawn(move || {
        let mut list = worker_orders.lock().unwrap();
        list.push(40);
        let fifth = list[4]; // BUG: there are only four orders
        println!("fifth order: {fifth}");
    });

    let result = worker.join();
    println!("worker finished with an error: {}", result.is_err());

    match orders.lock() {
        Ok(list) => println!("orders: {list:?}"),
        Err(poisoned) => {
            println!("the lock is poisoned");
            println!("orders anyway: {:?}", poisoned.into_inner());
        }
    }
}
```

```bash
cargo run -q --bin thread_panic
```

```text
thread '<unnamed>' (5911757) panicked at src/bin/thread_panic.rs:11:25:
index out of bounds: the len is 4 but the index is 4
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
worker finished with an error: true
the lock is poisoned
orders anyway: [10, 20, 30, 40]
```

**Что происходит:**

1. Рабочий поток блокирует список, добавляет `40` и читает индекс 4 из списка с четырьмя элементами. Это паника, и поток раскручивает стек: его значения уничтожаются, включая guard-объект блокировки, поэтому мьютекс разблокируется.
2. `worker.join()` возвращает `Err` вместо результата замыкания. Главный поток не упал; он узнал, что рабочий поток отказал.
3. Поскольку рабочий поток запаниковал, *удерживая блокировку*, мьютекс теперь **отравлен**. `orders.lock()` возвращает `Err(PoisonError)`. Rust не может знать, закончил ли поток своё изменение или остановился посередине, поэтому предупреждает вас.
4. `poisoned.into_inner()` всё равно отдаёт данные. Здесь изменение было завершено, поэтому их можно использовать. В настоящем коде решайте внимательно: если возможно наполовину сделанное изменение, сбросьте данные или остановитесь.

Процесс завершился с кодом 0: один баг в одном потоке не остановил программу. Это базовый инструмент **изоляции**: выполняйте независимую работу в независимых потоках или задачах.

## 3. Как удержать панику внутри сервиса

Сервер обрабатывает много запросов одновременно. Баг, который срабатывает на редком запросе, должен провалить *этот* запрос, а не весь сервер. Tokio помогает: паника внутри запущенной задачи останавливает только эту задачу, а её `JoinHandle` возвращает `JoinError`, который говорит «эта задача запаниковала».

Положите это в `src/lib.rs`:

```rust
//! Helpers for handling panics in a long-running program.

use std::any::Any;

/// Turns a panic payload into readable text.
/// `panic!("text")` carries a `&str`, `panic!("{x}")` and most std panics carry a `String`.
pub fn panic_message(payload: &(dyn Any + Send)) -> &str {
    if let Some(text) = payload.downcast_ref::<&str>() {
        text
    } else if let Some(text) = payload.downcast_ref::<String>() {
        text
    } else {
        "<non-text panic payload>"
    }
}

/// A request handler with a bug: it panics for an empty name.
pub fn greet(name: &str) -> String {
    let first = name.chars().next().unwrap(); // BUG: panics on ""
    format!("Hello, {name}! Your name starts with {first}.")
}
```

а это — в `src/bin/tasks.rs`:

```rust
use p20_panics::{greet, panic_message};

#[tokio::main]
async fn main() {
    let requests = ["Ada", "", "Linus"];
    for name in requests {
        // Every request runs in its own task, like in a real server.
        let handle = tokio::spawn(async move { greet(name) });
        match handle.await {
            Ok(reply) => println!("200 OK: {reply}"),
            Err(err) if err.is_panic() => {
                let payload = err.into_panic();
                println!(
                    "500 Internal Server Error (bug: {})",
                    panic_message(payload.as_ref())
                );
            }
            Err(err) => println!("task was cancelled: {err}"),
        }
    }
    println!("the server is still running");
}
```

```bash
cargo run -q --bin tasks
```

```text
200 OK: Hello, Ada! Your name starts with A.

thread 'tokio-rt-worker' (5915415) panicked at src/lib.rs:19:37:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
500 Internal Server Error (bug: called `Option::unwrap()` on a `None` value)
200 OK: Hello, Linus! Your name starts with L.
the server is still running
```

**Что происходит:**

1. **Полезная нагрузка паники** — трейт-объект (Урок 06.1) трейта `Any` из `std::any`: значение, настоящий тип которого неизвестен при компиляции, но о котором можно спросить во время работы. `downcast_ref::<T>()` спрашивает «это `T`?» и возвращает `Some(&T)`, если да. Сообщения паники почти всегда `&str` или `String`, поэтому `panic_message` пробует оба.
2. Пустое имя заставляет `greet` запаниковать внутри своей задачи. `handle.await` возвращает `Err(err)`, `err.is_panic()` истинно, а `err.into_panic()` отдаёт нам полезную нагрузку.
3. «Сервер» превращает баг в ответ 500 для этого одного запроса, а следующий запрос обслуживается нормально. Один пользователь увидел ошибку; двое других ничего не заметили.

Веб-фреймворки дают такую же изоляцию. В axum (Урок 16.3) каждое соединение работает в своей задаче, поэтому паникующий обработчик ломает только своё соединение. Middleware `CatchPanicLayer` из `tower-http` идёт на шаг дальше и превращает панику в нормальный ответ «500 Internal Server Error».

> ⚠️ **Частая ошибка:** изоляция — это не исправление. Баг остался и ударит по следующему пользователю с пустым именем. Изоляция даёт вам время; отчёт о панике (следующий раздел) говорит, что исправлять.

## 4. Делаем каждую панику заметной: хук паники

Сообщение паники по умолчанию идёт в стандартный поток ошибок несколькими строками свободного текста. В продакшене ваши логи собирает машина (Урок 16.12), и несколько неструктурированных строк легко пропустить. Ещё нужно *число*: «сколько паник случилось?» — один из лучших сигналов тревоги, который есть у сервиса.

`std::panic::set_hook` заменяет функцию, которая выполняется при каждой панике. Создайте `src/bin/hook.rs`:

```rust
use std::panic;
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread;

/// How many panics happened since start. A real service exports this as a metric.
static PANICS: AtomicU64 = AtomicU64::new(0);

fn install_panic_hook() {
    panic::set_hook(Box::new(|info| {
        let count = PANICS.fetch_add(1, Ordering::Relaxed) + 1;
        let thread = thread::current();
        let thread_name = thread.name().unwrap_or("<unnamed>");
        let location = info
            .location()
            .map(|l| format!("{}:{}", l.file(), l.line()))
            .unwrap_or_else(|| "unknown".to_string());
        let message = info.payload_as_str().unwrap_or("<non-text payload>");
        eprintln!(
            "level=ERROR event=panic thread={thread_name} location={location} \
             message={message:?} panics_total={count}"
        );
    }));
}

fn main() {
    install_panic_hook();

    let worker = thread::Builder::new()
        .name("report-worker".to_string())
        .spawn(|| {
            let sizes: Vec<u64> = Vec::new();
            let average = sizes.iter().sum::<u64>() / sizes.len() as u64; // BUG: divides by 0
            println!("average: {average}");
        })
        .unwrap();

    let _ = worker.join();
    println!(
        "main continues, panics so far: {}",
        PANICS.load(Ordering::Relaxed)
    );
}
```

```bash
cargo run -q --bin hook
```

```text
level=ERROR event=panic thread=report-worker location=src/bin/hook.rs:32 message="attempt to divide by zero" panics_total=1
main continues, panics so far: 1
```

**Что происходит:**

1. `set_hook` принимает замыкание в `Box`. Rust вызывает его при каждой панике в каждом потоке, *до* начала раскрутки стека, со значением `PanicHookInfo` по имени `info`.
2. `info.location()` даёт файл и строку паники; `info.payload_as_str()` даёт сообщение, если это текст.
3. Хук увеличивает атомарный счётчик (Урок 14.5) и пишет одну строку из пар `key=value`. Система логов может искать `event=panic` и поднимать тревогу по `panics_total`.
4. `thread::Builder::new().name(...)` даёт потоку имя, и строка лога становится гораздо полезнее, чем с `<unnamed>`.
5. Хук по умолчанию заменён, поэтому привычного многострочного сообщения больше нет. Упражнение 3 показывает, как сохранить и его.

В настоящем сервисе вы бы вызывали в хуке `tracing::error!` (Урок 09.10) вместо `eprintln!`, чтобы паники попадали в те же логи, что и всё остальное, и экспортировали бы счётчик как метрику (Урок 16.12).

> 💡 **Совет:** задайте `RUST_BACKTRACE=1` в окружении вашего сервиса. Тогда хук по умолчанию печатает трассировку стека (backtrace): список функций, которые выполнялись в момент паники. В собственном хуке `std::backtrace::Backtrace::capture()` даёт тот же список и учитывает ту же переменную.

> ⚠️ **Частая ошибка:** держите хук простым. Он выполняется посреди отказа: никаких блокировок, которые может удерживать паникующий код, никаких сетевых вызовов и уж точно никакого `unwrap`. Паника внутри хука паники аварийно прерывает процесс.

## 5. Раскрутка или прерывание?

Что делает паника во всей программе, выбирается одной настройкой в `Cargo.toml`:

```toml
[profile.release]
panic = "abort"
```

С `"abort"` (по умолчанию стоит `"unwind"`) первая паника в *любом* потоке сразу завершает весь процесс. Запустите пример с задачами в релизном режиме с этой настройкой и напечатайте код возврата:

```bash
cargo run -q --release --bin tasks
echo "exit code: $?"
```

```text
200 OK: Hello, Ada! Your name starts with A.

thread 'tokio-rt-worker' (5923630) panicked at src/lib.rs:19:37:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
exit code: 134
```

Хук всё ещё выполняется (сообщение напечатано), но затем процесс останавливается. «Linus» так и не был обслужен. Код возврата 134 означает «убит сигналом SIGABRT» (signal abort, сигнал аварийного прерывания, Урок 13.7). Ни `JoinError`, ни `catch_unwind` не могут перехватить прерывание, и `Drop` не выполняется, поэтому буферизованные строки логов и наполовину записанные файлы не сбрасываются на диск.

В продакшене используются оба варианта:

| | `panic = "unwind"` (по умолчанию) | `panic = "abort"` |
|---|---|---|
| Одна паника останавливает | один поток или задачу | весь процесс |
| `Drop` и очистка | выполняются во время раскрутки | не выполняются |
| Подходит для | серверов с множеством независимых запросов | программ, где баг означает, что состоянию в памяти нельзя доверять |
| Восстановление | сервис продолжает работать; баг исправляют позже | **супервизор** перезапускает процесс; тот восстанавливает состояние из надёжного хранилища |
| Размер бинарника | немного больше | немного меньше |

Стиль `abort` называют *crash-only*-проектированием: никогда не пытаться чинить состояние в памяти, а просто упасть и начать заново. Он хорошо подходит движку хранения, который может оказаться посреди записи: перезапуск и воспроизведение журнала упреждающей записи (WAL, Урок 17.4) безопаснее продолжения. Это работает, только если процесс кто-то перезапускает: `Restart=on-failure` в юнит-файле systemd (Урок 11.6) или Kubernetes (Этап 21).

> 🧪 **Тестирование:** тестовый раннер всегда использует раскрутку стека, что бы вы ни написали в `[profile.release]`, поэтому тесты с `#[should_panic]` и `catch_unwind` продолжают работать в `cargo test`.

Перед тем как продолжить, снова удалите строки `[profile.release]`.

## 6. Не пускаем паники в продакшен-код

Самая дешёвая паника — та, что никогда не доходит до продакшена. В Clippy (Урок 05.6) есть линты, которые указывают на каждое место, где код *может* запаниковать. По умолчанию они выключены, потому что `unwrap` уместен в примерах и тестах. Включите их для этого пакета в `Cargo.toml`:

```toml
[lints.clippy]
unwrap_used = "warn"
expect_used = "warn"
panic = "warn"
indexing_slicing = "warn"
```

и разрешите их внутри тестов файлом `clippy.toml` рядом с `Cargo.toml`:

```toml
allow-unwrap-in-tests = true
allow-expect-in-tests = true
allow-panic-in-tests = true
allow-indexing-slicing-in-tests = true
```

```bash
cargo clippy -q --lib
```

```text
warning: used `unwrap()` on an `Option` value
  --> src/lib.rs:19:17
   |
19 |     let first = name.chars().next().unwrap(); // BUG: panics on ""
   |                 ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
   |
   = note: if this value is `None`, it will panic
   = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.100.0/index.html#unwrap_used
   = note: requested on the command line with `-W clippy::unwrap-used`
```

**Что происходит:**

1. `[lints.clippy]` включает для пакета четыре линта; Cargo передаёт их Clippy как флаги командной строки, поэтому в примечании написано «requested on the command line».
2. `unwrap_used` нашёл ровно тот баг, на который наткнулся наш пример с задачами. `indexing_slicing` тоже отметил бы `list[4]` в `thread_panic.rs` (запустите `cargo clippy --bins`, чтобы увидеть).
3. `clippy.toml` оставляет тестам свободу использовать `unwrap()`: в тесте паника — *правильный* способ упасть.

Исправление — вернуть ожидаемую ошибку вместо паники. Упражнение 2 просит написать его вместе с тестом.

> 📏 **Соглашение:** когда паника действительно невозможна, пишите `expect("причина")` с причиной и разрешайте линт в этом одном месте через `#[allow(clippy::expect_used)]` с комментарием. Тогда каждый оставшийся `expect` в кодовой базе — задокументированное решение, и ревьюер может проверить причину.

> 🔒 **Безопасность:** в коде, который обрабатывает недоверенный ввод (парсеры, обработчики запросов), считайте каждое предупреждение этих линтов возможным отказом в обслуживании. Урок 10.12 показал, как один специально подобранный вход может уронить сервер, который паникует.

## Частые ошибки

**1. Использовать `catch_unwind` как обработку ошибок.** `catch_unwind` — страховочная сетка на границе (одна задача, один запрос, один плагин), а не способ «попробовать» рискованный код. Ожидаемые проблемы должны быть `Result`. Кроме того, некоторые паники вообще нельзя перехватить: при `panic = "abort"` или когда паника случается внутри хука паники.

**2. Игнорировать `JoinError`.** `let _ = handle.await;` выбрасывает информацию о том, что задача запаниковала. Всегда смотрите на результат задачи, которая вам важна, хотя бы чтобы записать его в лог.

**3. Распространять отравление.** Если за каждым `lock()` следует `.unwrap()`, одна паника под блокировкой заставляет паниковать и каждый следующий `lock()` (Упражнение 1 это показывает). Решите в одном месте, что отравленная блокировка означает для ваших данных.

**4. Выбрать `abort` без супервизора.** С `panic = "abort"` и без того, кто перезапускает процесс, первый же баг в продакшене останавливает сервис, пока его не заметит человек.

## Попробуйте сами

1. Создайте `code/p20-panics`, добавьте `src/bin/thread_panic.rs` и запустите его. Замените `list[4]` на `list[3]` и запустите снова: блокировка больше не отравлена.
2. Добавьте `src/lib.rs` и `src/bin/tasks.rs`. Запустите их и проверьте, что «Linus» обслуживается после паники.
3. Добавьте `src/bin/hook.rs`. Запустите его один раз обычно и один раз через `RUST_BACKTRACE=1 cargo run -q --bin hook`. Печатает ли ваш хук трассировку стека? (Нет: только хук по умолчанию сам читает эту переменную.)
4. Добавьте `panic = "abort"` в `[profile.release]`, выполните `cargo run -q --release --bin tasks; echo $?`, затем снова удалите настройку.
5. Включите четыре линта Clippy и выполните `cargo clippy --lib --bins`. Посчитайте предупреждения.

## Упражнения

### Упражнение 1 — Отравление распространяется (легко)

В `thread_panic.rs` замените весь `match orders.lock() { … }` этими двумя строками, запустите программу и объясните вывод:

```rust
    let list = orders.lock().unwrap();
    println!("orders: {list:?}");
```

<details>
<summary>Подсказка</summary>

Что возвращает `lock()` для отравленного мьютекса и что `unwrap()` с этим делает?

</details>

<details>
<summary>Ответ</summary>

```text
thread '<unnamed>' (5934422) panicked at src/bin/ex1.rs:11:25:
index out of bounds: the len is 4 but the index is 4
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
worker finished with an error: true

thread 'main' (5934392) panicked at src/bin/ex1.rs:18:30:
called `Result::unwrap()` on an `Err` value: PoisonError { .. }
```

(В выводе указан файл `src/bin/ex1.rs`, потому что код был сохранён под этим именем; у вас будет `thread_panic.rs`.) Паника рабочего потока отравила мьютекс. `lock()` возвращает `Err(PoisonError)`, а `unwrap()` превращает его во вторую панику, теперь в главном потоке, которая завершает программу с кодом 101. Один баг стал двумя отказами.

</details>

### Упражнение 2 — Исправьте обработчик, с тестами (средне)

Напишите `pub fn greet_checked(name: &str) -> Result<String, String>` в `src/lib.rs`. Для пустого имени она возвращает `Err("name must not be empty")`, иначе — то же приветствие, что и `greet`. Напишите два теста, по одному на каждый случай, и проверьте, что `cargo clippy --lib` больше не предупреждает об этой функции.

<details>
<summary>Подсказка</summary>

`let Some(first) = name.chars().next() else { return Err(...); };` обрабатывает пустой случай без `unwrap`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// The fixed handler: an empty name is an expected error, not a bug.
pub fn greet_checked(name: &str) -> Result<String, String> {
    let Some(first) = name.chars().next() else {
        return Err("name must not be empty".to_string());
    };
    Ok(format!("Hello, {name}! Your name starts with {first}."))
}

#[cfg(test)]
mod greet_tests {
    use super::*;

    #[test]
    fn empty_name_is_an_error_not_a_panic() {
        assert_eq!(greet_checked(""), Err("name must not be empty".to_string()));
    }

    #[test]
    fn normal_name_is_greeted() {
        assert_eq!(
            greet_checked("Ada"),
            Ok("Hello, Ada! Your name starts with A.".to_string())
        );
    }
}
```

Почему это работает: пустое имя теперь — ожидаемая ошибка, которую вызывающий может превратить в «400 Bad Request». В этой функции не осталось пути, который может запаниковать, поэтому Clippy нечего о ней сообщить.

</details>

### Упражнение 3 — Сохраните и сообщение по умолчанию (средне)

Измените хук так, чтобы он сначала писал однострочную запись `level=ERROR event=panic message=…`, а затем запускал и хук по умолчанию, чтобы привычное сообщение (и, с `RUST_BACKTRACE=1`, трассировка стека) всё равно появлялось. Проверьте его на потоке, который читает индекс 0 пустого `Vec<u32>`.

<details>
<summary>Подсказка</summary>

`panic::take_hook()` возвращает текущий хук (хук по умолчанию) как замыкание в `Box`. Переместите его в своё новое замыкание и вызовите в конце.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::panic;
use std::thread;

fn main() {
    let default_hook = panic::take_hook();
    panic::set_hook(Box::new(move |info| {
        let message = info.payload_as_str().unwrap_or("<non-text payload>");
        eprintln!("level=ERROR event=panic message={message:?}");
        default_hook(info);
    }));

    let worker = thread::spawn(|| {
        let empty: Vec<u32> = Vec::new();
        empty[0]
    });
    println!("worker failed: {}", worker.join().is_err());
}
```

```text
level=ERROR event=panic message="index out of bounds: the len is 0 but the index is 0"

thread '<unnamed>' (5934631) panicked at src/bin/ex3.rs:14:14:
index out of bounds: the len is 0 but the index is 0
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
worker failed: true
```

Почему это работает: `take_hook` убирает хук по умолчанию и отдаёт его вам; `move` кладёт его внутрь нового замыкания, которое вызывает его после записи своей строки. С `RUST_BACKTRACE=1` хук по умолчанию добавляет список `stack backtrace:`.

</details>

### Упражнение 4 — Исполнитель задач, который переживает паники (сложно)

Напишите `pub fn run_isolated<T>(job: impl FnOnce() -> T + std::panic::UnwindSafe) -> Result<T, String>`: она выполняет `job`, обычно возвращает `Ok(value)`, а если задача паникует — `Err(message)`. Используйте `catch_unwind` и `panic_message`. Напишите три теста: хорошая задача возвращает своё значение; задача, которая паникует с `"disk is full"`, даёт `Err("disk is full")`; в списке из трёх задач, где паникует вторая, первая и третья всё равно дают свои значения.

<details>
<summary>Подсказка</summary>

`std::panic::catch_unwind(job)` возвращает `Result<T, Box<dyn Any + Send>>`. Используйте `map_err`, чтобы превратить `Box` в `String`. `UnwindSafe` — маркерный трейт, который говорит «после паники можно смотреть на данные этого замыкания»; у простых замыканий он есть автоматически.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Runs one job and turns a panic into an `Err` with the panic message.
/// This works only with `panic = "unwind"` (the default).
pub fn run_isolated<T>(job: impl FnOnce() -> T + std::panic::UnwindSafe) -> Result<T, String> {
    std::panic::catch_unwind(job).map_err(|payload| panic_message(payload.as_ref()).to_string())
}

#[cfg(test)]
mod runner_tests {
    use super::*;

    #[test]
    fn a_good_job_returns_its_value() {
        assert_eq!(run_isolated(|| 2 + 2), Ok(4));
    }

    #[test]
    fn a_panicking_job_becomes_an_error() {
        let result: Result<u32, String> = run_isolated(|| panic!("disk is full"));
        assert_eq!(result, Err("disk is full".to_string()));
    }

    #[test]
    fn later_jobs_still_run_after_a_panic() {
        let results: Vec<Result<u32, String>> = vec![
            run_isolated(|| 1),
            run_isolated(|| panic!("job 2 failed")),
            run_isolated(|| 3),
        ];
        assert_eq!(results, vec![Ok(1), Err("job 2 failed".to_string()), Ok(3)]);
    }
}
```

```bash
cargo test -q --lib runner_tests
```

```text
running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 3 filtered out; finished in 0.00s
```

(Число отфильтрованных тестов зависит от того, сколько у вас других тестов.) Почему это работает: `catch_unwind` останавливает раскрутку стека на границе замыкания, поэтому паника не может выйти из `run_isolated`. Это дополнительная цель с пулом потоков из Проекта 06.11: оберните так каждую задачу, и ошибочная задача больше не убивает рабочий поток. (Сообщения паник по-прежнему печатает хук; тестовый раннер скрывает их, потому что тесты проходят.)

</details>

## Вопросы для самопроверки

1. Что из этого должно быть паникой: отсутствующий файл конфигурации, слишком длинное имя пользователя или чтение за концом буфера, размер которого задал ваш собственный код?

<details>
<summary>Ответ</summary>

Только последнее: это баг в вашем коде. Отсутствующий файл и слишком длинное имя — ожидаемые ошибки, их нужно возвращать как `Err`.

</details>

2. Поток паникует, удерживая `Mutex`. Что происходит с мьютексом и почему?

<details>
<summary>Ответ</summary>

Он становится отравленным: последующие вызовы `lock()` возвращают `Err(PoisonError)`. Rust не может знать, было ли изменение данных закончено, поэтому предупреждает каждого, кто следующим использует данные. `into_inner()` всё равно даёт к ним доступ.

</details>

3. В сервере на Tokio паникует обработчик. Останавливается ли сервер?

<details>
<summary>Ответ</summary>

Нет (при `panic = "unwind"` по умолчанию). Останавливается только эта задача; её `JoinHandle` возвращает `JoinError` с `is_panic() == true`. Остальные задачи продолжают работать.

</details>

4. Назовите две вещи, которые делает хороший хук паники в сервисе.

<details>
<summary>Ответ</summary>

Он пишет структурированную строку лога с местом и сообщением (чтобы система логов её нашла) и увеличивает счётчик паник, который экспортируется как метрика и по которому срабатывает тревога. Он также может вызвать предыдущий хук.

</details>

5. Почему `panic = "abort"` может быть *более* надёжным выбором для движка хранения?

<details>
<summary>Ответ</summary>

После бага посреди записи состояние в памяти может быть несогласованным. Прервать процесс и перезапуститься из надёжных данных (например, воспроизведя журнал упреждающей записи) безопаснее, чем продолжать с сомнительным состоянием. Для этого нужен супервизор, который перезапускает процесс.

</details>

6. Что делает `allow-unwrap-in-tests = true` в `clippy.toml` и почему это разумно?

<details>
<summary>Ответ</summary>

Это отключает предупреждения линта `unwrap_used` внутри тестов. В тесте паника — нормальный способ упасть, поэтому `unwrap()` там уместен; линт предназначен для продакшен-кода.

</details>

## Итоги

- Ожидаемые проблемы — это `Err`; паника означает баг. Баги будут попадать в продакшен, поэтому сервис должен их переживать, сообщать о них и изолировать их.
- Паника раскручивает только свой поток или задачу. `join()` возвращает `Err`, `JoinError` из Tokio сообщает `is_panic()`, а мьютекс, удерживаемый во время паники, становится **отравленным**.
- Выполняйте независимую работу в независимых задачах: тогда один плохой запрос падает в одиночку. `catch_unwind` — такая же страховочная сетка для синхронных задач.
- **Хук паники** (`std::panic::set_hook`) превращает каждую панику в одну структурированную строку лога и счётчик; `take_hook` позволяет сохранить сообщение по умолчанию.
- `panic = "unwind"` сохраняет процесс живым; `panic = "abort"` сразу его останавливает (код возврата 134) и требует супервизора. Выбирайте для каждой программы отдельно.
- Линты Clippy `unwrap_used`, `expect_used`, `panic` и `indexing_slicing`, разрешённые в тестах через `clippy.toml`, находят возможные паники раньше продакшена.

## Источники и что почитать

- [The Rust Programming Language — 9.3 To panic! or Not to panic!](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html) — официальные рекомендации о том, когда паниковать, а когда возвращать `Result`.
- [std::panic::set_hook](https://doc.rust-lang.org/std/panic/fn.set_hook.html) и [std::panic::catch_unwind](https://doc.rust-lang.org/std/panic/fn.catch_unwind.html) — API (Application Programming Interface, программный интерфейс) хука и пределы перехвата паник.
- [std::sync::Mutex — poisoning](https://doc.rust-lang.org/std/sync/struct.Mutex.html) — что означает отравление и как вернуть данные.
- [tokio::task::JoinError](https://docs.rs/tokio/1.54.1/tokio/task/struct.JoinError.html) — `is_panic`, `into_panic` и отменённые задачи.
- [The Cargo Book — Profiles: panic](https://doc.rust-lang.org/cargo/reference/profiles.html) — настройка `panic` и почему тесты всегда раскручивают стек.
- [Clippy lints](https://rust-lang.github.io/rust-clippy/master/index.html) — найдите `unwrap_used`, `indexing_slicing` и параметры `allow-…-in-tests`.

---

[🏠 Начало](../START-HERE.md) · [← 20.1 Что такое надёжность](01-what-reliability-means.md) · [📋 Этап 20](README.md) · [20.3 Фаззинг подробно →](03-fuzzing-in-depth.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
