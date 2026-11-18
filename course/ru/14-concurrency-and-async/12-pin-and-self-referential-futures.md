<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 14.11 Пишем крошечный асинхронный рантайм](11-build-a-tiny-async-runtime.md) · [📋 Этап 14](README.md) · [14.13 Tokio подробно: задачи и каналы →](13-tokio-tasks-and-channels.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/14-concurrency-and-async/12-pin-and-self-referential-futures.md) <!-- nav -->

# Урок 14.12 — Pin и самоссылающиеся future

> **Этап 14 — Конкурентность и асинхронность подробно** · Урок 12 из 16 · ⏱ около 120 минут

## Что вы узнаете

- Почему значение, которое указывает само на себя, ломается при перемещении и почему многие future — именно такие значения.
- Что обещает `Pin` и что означает автотрейт `Unpin`.
- Два способа закрепить future — `Box::pin` и `pin!` — и когда какой использовать.
- Как читать ошибку компилятора «… cannot be unpinned» и как её исправлять.
- Как написать future, который оборачивает другой future, с помощью *проекции закрепления* (pin projection) из крейта `pin-project-lite`.

## Перед началом

- Вы прошли [Урок 14.11 — Пишем крошечный асинхронный рантайм](11-build-a-tiny-async-runtime.md). Там вы использовали `Pin<Box<dyn Future>>` и `Pin::new` без полного объяснения; этот урок его даёт.
- Вы знаете сырые указатели и формат `{:p}` из [Урока 06.6 — Небезопасный Rust и Miri](../06-rust-advanced/06-unsafe-rust-and-miri.md), а *перемещения* — с [Этапа 03](../03-ownership-and-borrowing/README.md).
- Вы знаете `tokio::select!` и `tokio::time::sleep` из [Урока 06.9](../06-rust-advanced/09-async-await-introduction.md).
- Код этого урока находится в `~/projects/rust/code/p14-pin`. Добавьте зависимости:

```bash
cd ~/projects/rust
cargo new --lib code/p14-pin
cd code/p14-pin
cargo add futures@0.3 pin-project-lite@0.2
cargo add tokio@1 --features rt-multi-thread,macros,time
mkdir src/bin
```

## Новые слова

| Слово | Значение |
|---|---|
| **самоссылающийся** (self-referential) | Значение, которое содержит указатель или ссылку на другую свою часть. |
| **перемещение** (move, в памяти) | Копирование байтов значения по новому адресу, после чего старое место больше не используется. Любое перемещение в Rust может так делать. |
| **висячий указатель** (dangling pointer) | Указатель на место, где значения больше нет. Чтение через него — неопределённое поведение. |
| **`Pin<P>`** | Обёртка вокруг указателя `P` (например, `&mut T` или `Box<T>`), которая обещает: значение, на которое он указывает, больше не будет перемещено, пока его не уничтожат. |
| **закрепление** (pinning) | Помещение значения за `Pin`, чтобы оно оставалось по одному адресу. |
| **`Unpin`** | Автотрейт, означающий «перемещать это значение всегда безопасно, даже если оно закреплено». Почти все типы его реализуют; future, созданные компилятором, — нет. |
| **автотрейт** (auto trait) | Трейт, который компилятор автоматически реализует для типа, если он есть у всех полей, — как `Send` и `Sync`. |
| **`PhantomPinned`** | Маркерный тип без данных. Структура, которая его содержит, не является `Unpin`. |
| **проекция закрепления** (pin projection) | Получение закреплённой ссылки на одно поле (`Pin<&mut Field>`) из закреплённой ссылки на всю структуру (`Pin<&mut Struct>`). |

## 1. Проблема: значение, которое указывает само на себя

Перемещение в Rust копирует байты значения на новое место. Обычно это совершенно нормально. Это перестаёт быть нормальным, когда значение хранит адрес одного из своих полей. Создайте `src/bin/moving.rs`:

```rust
/// A struct that keeps a pointer to one of its own fields.
struct SelfRef {
    name: String,
    name_ptr: *const String,
}

impl SelfRef {
    fn new(name: &str) -> Self {
        SelfRef {
            name: name.to_string(),
            name_ptr: std::ptr::null(),
        }
    }

    fn remember_address(&mut self) {
        self.name_ptr = &self.name;
    }

    fn report(&self, label: &str) {
        let real: *const String = &self.name;
        println!(
            "{label}: field at {real:p}, pointer says {:p}, same: {}",
            self.name_ptr,
            real == self.name_ptr
        );
    }
}

fn main() {
    let mut first = SelfRef::new("ferris");
    first.remember_address();
    first.report("before move");

    let moved = first; // a move copies the bytes to a new place
    moved.report("after move ");
}
```

`cargo run --bin moving` (ваши адреса будут другими):

```text
before move: field at 0x16d751d90, pointer says 0x16d751d90, same: true
after move : field at 0x16d751db0, pointer says 0x16d751d90, same: false
```

**Что происходит:**

1. `remember_address` сохраняет адрес `self.name` внутри самой структуры. Создать сырой указатель безопасно; `unsafe` нужен только для чтения через него.
2. `let moved = first;` перемещает структуру. Её байты теперь лежат по новому адресу (здесь на 32 байта дальше в стеке).
3. Поле переместилось, а сохранённый указатель по-прежнему хранит **старый** адрес. Теперь это висячий указатель. Мы только сравниваем адреса; чтение через `name_ptr` здесь было бы неопределённым поведением.

Проверка заимствований не даёт написать такое на обычных ссылках (`&String`, указывающая в ту же структуру) — именно из-за этой опасности. Поэтому в примере нужны сырые указатели.

## 2. Почему future бывают самоссылающимися

`async fn` может держать ссылку на собственную локальную переменную через `.await`. Создайте `src/bin/future_size.rs`:

```rust
async fn tick() {}

async fn sum_after_tick() -> u64 {
    let numbers = [1_u64, 2, 3, 4];
    let first = &numbers[0]; // a reference into a local of the future
    tick().await; // `numbers` and `first` must survive this point
    *first + numbers[3]
}

fn main() {
    let future = sum_after_tick();
    println!("the future is {} bytes", std::mem::size_of_val(&future));
    println!("result = {}", futures::executor::block_on(future));
}
```

```text
the future is 48 bytes
result = 5
```

**Что происходит:**

1. Из Урока 14.10 вы знаете, что компилятор превращает `sum_after_tick` в конечный автомат. Всё, что живёт через `tick().await`, хранится **внутри future**: массив `numbers` (32 байта), ссылка `first` (8 байтов) и номер состояния.
2. `first` указывает на `numbers[0]`, и оба находятся внутри одного future. Future самоссылающийся, как и `SelfRef`.
3. До первого опроса ссылки ещё нет, поэтому перемещать future можно (здесь мы перемещаем его в `block_on`). После первого опроса `first` хранит адрес внутри future. Если бы future тогда переместился, `first` стала бы висячей.

Итак, правило для future: **после первого опроса future нельзя перемещать**. `Pin` — это способ, которым Rust делает это правило частью системы типов: `poll` принимает `self: Pin<&mut Self>`, поэтому его нельзя вызвать, не пообещав сначала не перемещать future.

## 3. `Pin` и `Unpin`

`Pin<P>` оборачивает указатель: `Pin<&mut T>`, `Pin<Box<T>>`. Он не добавляет ни данных, ни затрат во время выполнения. Он лишь отнимает одну возможность: из `Pin<&mut T>` нельзя получить обычный `&mut T` (в безопасном коде), а с обычным `&mut T` можно было бы вынуть значение через `std::mem::swap` или `std::mem::replace`. Нет `&mut T` — нет перемещения.

Но большинству типов перемещение вообще безразлично. `i32`, `String` или `Vec` не держат указателей внутрь себя. Для них `Pin` был бы бесполезной обузой, поэтому в Rust есть автотрейт **`Unpin`**: «перемещать это значение безопасно, даже когда оно закреплено». Для типа `Unpin` вызовы `Pin::new(&mut value)` и `pin.get_mut()` разрешены в безопасном коде, и `Pin` — просто формальность. Вы пользовались этим в Уроке 14.10 (`Pin::new(inner)` для `YieldOnce`).

Какие типы **не** являются `Unpin`?

| Тип | `Unpin`? | Почему |
|---|---|---|
| `i32`, `String`, `Vec<T>`, `Box<T>` | да | нет указателей внутрь себя |
| `Pin<Box<T>>` для любого `T` | да | перемещение box перемещает только указатель; значение остаётся в куче |
| future из `async fn` или блока `async` | нет | после первого опроса может держать ссылки внутрь себя |
| `tokio::time::Sleep` | нет | содержит `PhantomPinned`, потому что таймер Tokio хранит указатель на него |
| ваша собственная структура | да, если все поля `Unpin` | автотрейт, как `Send` |

## 4. Как закрепить future

Есть два способа. Оба дают то, что можно опрашивать снова и снова.

**`Box::pin(future)`** кладёт future в кучу и возвращает `Pin<Box<F>>`. Последующее перемещение box перемещает только указатель, поэтому сам future не двигается. Используйте его, когда future нужно где-то хранить: в структуре, в `Vec`, в очереди задач (именно этим был `BoxFuture` в Уроке 14.11) или вернуть из функции. Это стоит одного выделения памяти в куче.

**`std::pin::pin!(future)`** закрепляет future в текущей функции (на стеке) и даёт `Pin<&mut F>`. Без выделения памяти. Future не может покинуть функцию, но для самого частого случая это и не нужно: future, который опрашивается несколько раз внутри цикла.

Этот частый случай — `select!` в цикле. Вам нужен один срок на весь цикл, а не новый срок на каждом круге, поэтому вы создаёте future `sleep` один раз и передаёте `select!` ссылку `&mut` на него. (`tokio::time::interval(d)` — таймер, который «тикает» каждые `d`; каждый `ticks.tick().await` ждёт следующего тика.)

```rust
use std::time::Duration;
use tokio::time::{interval, sleep};

#[tokio::main]
async fn main() {
    let deadline = sleep(Duration::from_millis(350));
    let mut ticks = interval(Duration::from_millis(100));
    loop {
        tokio::select! {
            _ = &mut deadline => break,
            _ = ticks.tick() => println!("tick"),
        }
    }
}
```

Это не компилируется:

```text
error[E0277]: `PhantomPinned` cannot be unpinned
  --> src/bin/select_loop.rs:9:9
   |
 9 | /         tokio::select! {
10 | |             _ = &mut deadline => break,
11 | |             _ = ticks.tick() => println!("tick"),
12 | |         }
   | |_________within `(PhantomData<&()>, PhantomPinned)`, the trait `Unpin` is not implemented for `PhantomPinned`
   |
   = note: consider using the `pin!` macro
           consider using `Box::pin` if you need to access the pinned value outside of the current scope
   = note: required because it appears within the type `(PhantomData<&()>, PhantomPinned)`
   = note: required for `Sleep` to implement `Unpin`
   = note: required for `&mut Sleep` to implement `Future`
```

Как это читать, снизу вверх: `select!` нужно, чтобы `&mut Sleep` был future; `&mut F` является future, только если `F` — `Unpin` (иначе опрос через обычный `&mut` мог бы его переместить); `Sleep` не `Unpin`, потому что содержит `PhantomPinned`. Компилятор даже подсказывает исправление. Вот рабочая версия, `src/bin/select_loop.rs`:

```rust
use std::pin::pin;
use std::time::Duration;
use tokio::time::{interval, sleep};

#[tokio::main]
async fn main() {
    let mut deadline = pin!(sleep(Duration::from_millis(350)));
    let mut ticks = interval(Duration::from_millis(100));
    loop {
        tokio::select! {
            _ = &mut deadline => break,
            _ = ticks.tick() => println!("tick"),
        }
    }
    println!("deadline reached");
}
```

```text
tick
tick
tick
tick
deadline reached
```

**Что происходит:**

1. `pin!(sleep(…))` закрепляет future `Sleep` в `main` и даёт `Pin<&mut Sleep>`. Этот закреплённый указатель сам является future и *является* `Unpin` (это всего лишь указатель; `Sleep` за ним остаётся на месте), поэтому `&mut deadline` — тоже future, и `select!` его принимает.
2. `interval` тикает на 0, 100, 200 и 300 мс (первый тик — сразу), поэтому напечатано четыре тика.
3. На 350 мс тот же самый future `deadline`, который опрашивался на каждом круге, завершается, и цикл прерывается. Если написать `sleep(Duration::from_millis(350))` прямо внутри `select!`, на каждом круге создавался бы новый sleep, и при тике каждые 100 мс он никогда бы не закончился.

> 💡 **Совет:** та же ошибка появляется с обычным future из `async fn`, например `Pin::new(&mut hello())`: «`{async fn body of hello()}` cannot be unpinned», с той же заметкой «consider using the `pin!` macro». Ответ всегда один из двух: `pin!`, если future остаётся в этой функции, `Box::pin`, если ему нужно уйти куда-то ещё.

## 5. Пишем future-обёртку: проекция закрепления

Иногда вы пишете future, который содержит другой future, — как `timeout` содержит future, который он ограничивает. Внутренний future может быть самоссылающимся, поэтому он тоже должен оставаться закреплённым. Внутри `poll` у вас есть `self: Pin<&mut Self>`, а для поля нужен `Pin<&mut Inner>`. Получение этого и называется **проекцией закрепления**.

Вручную это требует `unsafe` (`Pin::get_unchecked_mut` и `Pin::new_unchecked`) и аккуратного обещания никогда не перемещать поле. Маленький крейт `pin-project-lite` делает это за вас, безопасно, макросом. Вот обёртка `Timed`, которая измеряет, сколько длился внутренний future и сколько опросов ему понадобилось. Поместите её в `src/lib.rs`:

```rust
//! A future wrapper that needs pin projection.

use pin_project_lite::pin_project;
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

pin_project! {
    /// Wraps a future and measures how long it ran and how often it was polled.
    pub struct Timed<F> {
        #[pin]
        inner: F,
        polls: u32,
        started: Option<Instant>,
    }
}

/// What `Timed` returns: the inner output plus the measurements.
#[derive(Debug)]
pub struct Report<T> {
    pub output: T,
    pub polls: u32,
    pub elapsed: Duration,
}

pub fn timed<F: Future>(inner: F) -> Timed<F> {
    Timed {
        inner,
        polls: 0,
        started: None,
    }
}

impl<F: Future> Future for Timed<F> {
    type Output = Report<F::Output>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();
        *this.polls += 1;
        let started = *this.started.get_or_insert_with(Instant::now);
        match this.inner.poll(cx) {
            Poll::Pending => Poll::Pending,
            Poll::Ready(output) => Poll::Ready(Report {
                output,
                polls: *this.polls,
                elapsed: started.elapsed(),
            }),
        }
    }
}
```

**Что происходит:**

1. `pin_project! { … }` оборачивает обычное определение структуры. `#[pin]` отмечает поля, которые должны оставаться закреплёнными: здесь только `inner`.
2. Макрос создаёт метод `project(self: Pin<&mut Self>)`. Он возвращает вспомогательную структуру с полем на каждое исходное поле: `inner` становится `Pin<&mut F>` (из-за `#[pin]`), а `polls` и `started` — обычными `&mut u32` и `&mut Option<Instant>`. Закреплять счётчик бессмысленно; мы его просто меняем.
3. `this.inner.poll(cx)` опрашивает внутренний future через закреплённую ссылку, поэтому это работает для любого `F: Future`, в том числе для future, которые не `Unpin`. Ограничение `Unpin` не нужно — в отличие от `Join2` из Урока 14.10.
4. `get_or_insert_with(Instant::now)` записывает время начала при первом опросе и сохраняет его дальше.
5. Макрос также следит, чтобы `Timed<F>` был `Unpin`, только когда `F` — `Unpin`; это правильное правило.

Используйте её из `src/bin/timed.rs`:

```rust
use p14_pin::timed;
use std::time::Duration;

async fn slow_answer() -> u32 {
    tokio::time::sleep(Duration::from_millis(100)).await;
    tokio::time::sleep(Duration::from_millis(50)).await;
    42
}

#[tokio::main]
async fn main() {
    let report = timed(slow_answer()).await;
    println!(
        "output = {}, polls = {}, elapsed = {} ms",
        report.output,
        report.polls,
        report.elapsed.as_millis()
    );
}
```

```text
output = 42, polls = 3, elapsed = 159 ms
```

Три опроса: один в начале и по одному после каждого сна. Время чуть больше 150 мс, потому что таймер всегда срабатывает в свой срок или позже, но никогда раньше.

> 🔒 **Безопасность:** `pin-project-lite` существует потому, что ручную проекцию легко сделать неправильно в `unsafe`-коде: один `mem::swap` поля, которое должно быть закреплено, или реализация `Drop`, которая его перемещает, — это неопределённое поведение, которое компилятор поймать не может. Предпочитайте макрос собственным блокам `unsafe`.

## Частые ошибки

**1. `&mut future` в `select!` без закрепления.** Вы получаете «`…` cannot be unpinned» (раздел 4). Исправление: `let mut future = pin!(future);` перед циклом.

**2. Создание future внутри цикла.** `tokio::select! { _ = sleep(d) => … }` внутри `loop` запускает новый `sleep` на каждом круге, поэтому «срок» так и не наступает, пока срабатывают другие ветки. Создайте future один раз, закрепите его и передавайте `&mut` на него.

**3. `Box::pin` везде, «чтобы ошибка ушла».** Это работает, но выделяет память при каждом вызове. Внутри одной функции достаточно `pin!`. `Box::pin` нужен, когда вы храните, возвращаете или держите future разных типов в одной коллекции (`Pin<Box<dyn Future>>`).

**4. Лишние ограничения `Unpin` в собственном обобщённом коде.** `fn run<F: Future + Unpin>(f: F)` отвергает любой future из `async fn`. Лучше закрепите future внутри (`let f = pin!(f);`) или сделайте проекцию через `pin-project-lite` в обёртке.

**5. Мысль, что `Pin` что-то двигает или исправляет во время выполнения.** `Pin` — лишь обещание, которое проверяет компилятор. Он ничего не копирует, не блокирует и не выделяет. `Box::pin` выделяет память из-за `Box`, а не из-за `Pin`.

## Попробуйте сами

1. Создайте `code/p14-pin`, как в разделе «Перед началом», запустите `moving` и `future_size`.
2. В `future_size.rs` замените массив на `[1_u64; 100]` и запустите снова. Предскажите новый размер future, прежде чем смотреть. (816 байтов: он вырастает на размер массива, потому что массив живёт внутри future.)
3. Напишите цикл с `select!` без `pin!`, прочитайте ошибку компилятора целиком, затем исправьте.
4. Добавьте `Timed` в `src/lib.rs` и запустите `timed`. Измените `slow_answer`, чтобы она спала три раза, и проверьте, что `polls` становится 4.
5. Запустите `cargo clippy --all-targets` и `cargo fmt`, затем сделайте коммит: `git add code/p14-pin && git commit -m "p14-pin: pinning and a Timed wrapper"`.

## Упражнения

### Упражнение 1 — Кто `Unpin`? (легко)

Напишите `fn assert_unpin<T: Unpin>(_value: &T) {}` и вызовите её с `&5_i32`, `&String::from("text")`, с `Pin<Box<dyn Future<Output = ()>>>`, с future из `async fn hello() {}` и с future `tokio::time::sleep(…)`. Предскажите, какие вызовы скомпилируются, затем проверьте через `cargo build`.

<details>
<summary>Подсказка</summary>

Посмотрите на таблицу в разделе 3. Комментируйте вызовы по одному, чтобы увидеть каждую ошибку отдельно.

</details>

<details>
<summary>Ответ</summary>

Первые три компилируются. Последние два — нет:

```text
error[E0277]: `{async fn body of hello()}` cannot be unpinned
error[E0277]: `PhantomPinned` cannot be unpinned
```

Почему это работает: `i32` и `String` не содержат указателей внутрь себя. `Pin<Box<…>>` — лишь указатель в кучу, поэтому его перемещение не двигает ничего важного. Future из `async fn` может стать самоссылающимся после первого опроса, а `Sleep` намеренно содержит `PhantomPinned`.

</details>

### Упражнение 2 — Протестируйте `Timed` (средне)

Добавьте тесты в `src/lib.rs`: `timed(async { 42 })` возвращает результат 42 ровно после **одного** опроса; блок `async`, который один раз вызывает `tokio::task::yield_now().await` и затем возвращает `"done"`, требует ровно **двух** опросов. Для первого теста используйте `futures::executor::block_on`, для второго — `#[tokio::test]`.

<details>
<summary>Подсказка</summary>

`yield_now` из Tokio работает как ваш `yield_now` из Урока 14.11: один раз возвращает `Pending` и будит себя. Ему нужен рантайм Tokio, который создаёт `#[tokio::test]`. Проверяйте `report.polls` через `assert_eq!`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_ready_future_is_polled_once() {
        let report = futures::executor::block_on(timed(async { 42 }));
        assert_eq!(report.output, 42);
        assert_eq!(report.polls, 1);
    }

    #[tokio::test]
    async fn a_future_that_yields_once_is_polled_twice() {
        let report = timed(async {
            tokio::task::yield_now().await;
            "done"
        })
        .await;
        assert_eq!(report.output, "done");
        assert_eq!(report.polls, 2);
    }
}
```

Почему это работает: блок `async` без `.await` внутри готов (`Ready`) при первом опросе. `yield_now` добавляет ровно один `Pending`, поэтому `Timed` насчитывает два опроса. Эти тесты не зависят от времени, поэтому никогда не бывают нестабильными.

</details>

### Упражнение 3 — Собственный `timeout` (сложно)

Напишите `WithTimeout<F>` с двумя закреплёнными полями: внутренним future и `tokio::time::Sleep`. Его результат — `Result<F::Output, TimedOut>`, где `TimedOut` — ваша собственная единичная структура. Добавьте `with_timeout(limit, inner)` и два `#[tokio::test]`: `async { 7 }` с пределом 200 мс даёт `Ok(7)`; `sleep` на 500 мс с пределом 20 мс даёт `Err(TimedOut)`.

<details>
<summary>Подсказка</summary>

Оба поля получают `#[pin]`: `Sleep` не `Unpin`, и `F` может не быть. В `poll` сделайте проекцию один раз, сначала опросите `inner` (если готов — верните `Ok`), затем `sleep` (если готов — верните `Err(TimedOut)`), иначе `Pending`. Оба внутренних future получили один и тот же waker, поэтому задачу разбудит тот, кто готов первым. Выведите (`derive`) `Debug` и `PartialEq` для `TimedOut`, чтобы работал `assert_eq!`.

</details>

<details>
<summary>Ответ</summary>

```rust
pin_project! {
    /// Like `tokio::time::timeout`: gives up when `sleep` finishes first.
    pub struct WithTimeout<F> {
        #[pin]
        inner: F,
        #[pin]
        sleep: tokio::time::Sleep,
    }
}

/// The error of `WithTimeout`.
#[derive(Debug, PartialEq)]
pub struct TimedOut;

pub fn with_timeout<F: Future>(limit: Duration, inner: F) -> WithTimeout<F> {
    WithTimeout {
        inner,
        sleep: tokio::time::sleep(limit),
    }
}

impl<F: Future> Future for WithTimeout<F> {
    type Output = Result<F::Output, TimedOut>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.project();
        if let Poll::Ready(output) = this.inner.poll(cx) {
            return Poll::Ready(Ok(output));
        }
        match this.sleep.poll(cx) {
            Poll::Ready(()) => Poll::Ready(Err(TimedOut)),
            Poll::Pending => Poll::Pending,
        }
    }
}
```

Два теста идут в модуль `tests`:

```rust
    #[tokio::test]
    async fn fast_future_wins() {
        let result = with_timeout(Duration::from_millis(200), async { 7 }).await;
        assert_eq!(result, Ok(7));
    }

    #[tokio::test]
    async fn slow_future_times_out() {
        let slow = tokio::time::sleep(Duration::from_millis(500));
        let result = with_timeout(Duration::from_millis(20), slow).await;
        assert_eq!(result, Err(TimedOut));
    }
```

`cargo test` показывает `test result: ok. 4 passed; 0 failed` для библиотеки (вместе с двумя тестами упражнения 2). Почему это работает: `project()` даёт `Pin<&mut F>` и `Pin<&mut Sleep>`, поэтому оба можно опрашивать, хотя ни один не `Unpin`. Если сначала опрашивать `inner`, то future, готовый в тот же момент, что и срок, всё равно побеждает; `tokio::time::timeout` делает тот же выбор.

</details>

## Вопросы для самопроверки

1. Почему перемещение ломает самоссылающееся значение?

<details>
<summary>Ответ</summary>

Перемещение копирует байты по новому адресу. Указатель, сохранённый внутри значения, по-прежнему хранит старый адрес, поэтому после перемещения он указывает туда, где поля уже нет (висячий указатель).

</details>

2. Почему future из `async fn` может быть самоссылающимся?

<details>
<summary>Ответ</summary>

Локальные переменные, которые живут через `.await`, хранятся внутри future. Если одна из них — ссылка на другую локальную переменную (`let first = &numbers[0];`), то после первого опроса future содержит указатель внутрь себя.

</details>

3. Что запрещает `Pin<&mut T>` и как?

<details>
<summary>Ответ</summary>

Вынимать `T` или менять его местами с другим значением, потому что безопасный код не может получить из него обычный `&mut T` (если только `T` не `Unpin`). Без `&mut T` нет ни `mem::swap`, ни `mem::replace`, поэтому значение остаётся на месте, пока его не уничтожат.

</details>

4. Что значит `Unpin` и какие распространённые типы не являются `Unpin`?

<details>
<summary>Ответ</summary>

«Перемещать это значение безопасно, даже когда оно закреплено»; для таких типов `Pin` — формальность. Future, созданные компилятором (`async fn`, блоки `async`), и типы с `PhantomPinned`, например `tokio::time::Sleep`, не являются `Unpin`.

</details>

5. Когда использовать `pin!`, а когда `Box::pin`?

<details>
<summary>Ответ</summary>

`pin!` — когда future остаётся в текущей функции, например future, который снова и снова опрашивается в цикле с `select!`; это ничего не стоит. `Box::pin` — когда future нужно хранить, вернуть или держать вместе с future других типов (`Pin<Box<dyn Future>>`); это стоит выделения памяти в куче.

</details>

6. Что делает `#[pin]` в `pin_project!`?

<details>
<summary>Ответ</summary>

Отмечает поле как структурно закреплённое: `project()` возвращает для него `Pin<&mut Field>`, поэтому его можно опрашивать, даже если оно не `Unpin`. Поля без `#[pin]` возвращаются как обычные `&mut Field`.

</details>

## Итоги

- Перемещение копирует байты по новому адресу; значение с указателем внутрь себя от этого ломается.
- Future из `async fn` могут держать ссылки на собственные локальные переменные через `.await`, поэтому после первого опроса их нельзя перемещать.
- `Pin<P>` обещает, что значение больше не переместится; `poll` принимает `Pin<&mut Self>`, и это правило соблюдается. Во время выполнения это ничего не стоит.
- Типы `Unpin` (почти все обычные типы) не замечают закрепления; future, созданные компилятором, и типы с `PhantomPinned` не являются `Unpin`.
- Закрепляйте через `pin!` внутри одной функции (бесплатно) или через `Box::pin`, чтобы хранить или вернуть future (одно выделение памяти).
- «… cannot be unpinned» означает «сначала закрепите»; обёртки вокруг future используют `pin-project-lite` для безопасной проекции закрепления.

## Источники и что почитать

- [std::pin module](https://doc.rust-lang.org/std/pin/index.html) — официальное объяснение закрепления, `Unpin` и проекции со всеми правилами.
- [std::pin::pin! macro](https://doc.rust-lang.org/std/pin/macro.pin.html) — закрепление на стеке.
- [The Rust Programming Language — 17.5 A Closer Look at the Traits for Async](https://doc.rust-lang.org/book/ch17-05-traits-for-async.html) — `Pin` и `Unpin` в официальной книге.
- [pin-project-lite на docs.rs](https://docs.rs/pin-project-lite/0.2.17/pin_project_lite/) — макрос, который используется в этом уроке.
- [Tokio tutorial — Select](https://tokio.rs/tokio/tutorial/select) — `select!` в циклах, с `&mut` и закреплением.
- [Asynchronous Programming in Rust — Pinning](https://rust-lang.github.io/async-book/part-reference/pinning.html) — глава о закреплении в официальной книге об асинхронности.

---

[🏠 Начало](../START-HERE.md) · [← 14.11 Пишем крошечный асинхронный рантайм](11-build-a-tiny-async-runtime.md) · [📋 Этап 14](README.md) · [14.13 Tokio подробно: задачи и каналы →](13-tokio-tasks-and-channels.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
