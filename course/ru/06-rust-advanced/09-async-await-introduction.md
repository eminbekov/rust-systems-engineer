<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 06.8 Процедурные макросы](08-procedural-macros.md) · [📋 Этап 06](README.md) · [06.10 FFI: вызов кода на C →](10-ffi-calling-c.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/06-rust-advanced/09-async-await-introduction.md) <!-- nav -->

# Урок 06.9 — Введение в async/await

> **Этап 06 — Rust: продвинутый уровень** · Урок 9 из 12 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Почему программам, которые ждут (сеть, таймер, диск), нужен другой инструмент, чем «один поток на задачу».
- Что такое `Future`, что на самом деле делают `async fn` и `.await` и почему future ничего не делает, пока его не опросят (poll).
- Как пользоваться рантаймом Tokio: `#[tokio::main]`, `tokio::spawn`, `tokio::join!`, каналы, `select!` и тайм-ауты.
- Главную ошибку async — блокировку рантайма — и другие классические ошибки с настоящими сообщениями компилятора.
- Как тестировать async-код и как выбирать между async и потоками.

## Перед началом

- Вы прошли [Урок 06.8 — Процедурные макросы](08-procedural-macros.md). `#[tokio::main]` — процедурный макрос.
- Вы знаете потоки, каналы и `Arc<Mutex<T>>`: [Урок 05.10 — Потоки, каналы и общее состояние](../05-rust-intermediate/10-threads-channels-and-shared-state.md).
- Вы знаете замыкания и `move`: [Урок 05.4 — Замыкания](../05-rust-intermediate/04-closures.md).
- Создайте проект: `cd ~/projects/rust && cargo new code/p06-async`, затем добавьте Tokio:

  ```sh
  cd code/p06-async
  cargo add tokio@1.54 --features rt-multi-thread,macros,time,sync
  ```

## Новые слова

| Слово | Значение |
|---|---|
| **I/O** (Input/Output, ввод-вывод) | Всё, где ваша программа общается с внешним миром: сеть, диск, клавиатура, таймеры. I/O медленный по сравнению с CPU. |
| **CPU** (Central Processing Unit, центральный процессор) | Микросхема, которая выполняет инструкции программы. |
| **конкурентность** (concurrency) | Несколько задач *находятся в работе* одновременно, даже если в каждый точный момент выполняется только одна. |
| **параллелизм** (parallelism) | Несколько задач *действительно выполняются* в один и тот же момент, на разных ядрах CPU. |
| **асинхронный** (async) | Способ писать код, который может приостанавливаться, пока ждёт, и позволять другому коду работать в это время. |
| **future** | Значение, которое представляет работу, завершающуюся позже. В Rust это всё, что реализует трейт `Future`. |
| **опрос** (poll) | Спросить future: «ты можешь продвинуться сейчас?» Он отвечает `Ready(value)` или `Pending`. |
| **`async fn`** | Функция, которая при вызове не выполняет тело, а возвращает future. |
| **`.await`** | Внутри async-функции: дождаться завершения future и позволить другой работе выполняться, пока вы ждёте. |
| **конечный автомат** (state machine) | Сущность, которая всегда находится в одном из нескольких именованных состояний и переходит между ними. Компилятор превращает каждую async-функцию в такой автомат. |
| **рантайм** (runtime) | Библиотека, которая запускает future: опрашивает их, будит и предоставляет таймеры и сетевой I/O. В стандартной библиотеке Rust рантайма нет. Tokio — самый используемый. |
| **исполнитель** (executor) | Часть рантайма, которая решает, какой future опрашивать следующим. |
| **waker** | Дескриптор, который future передаёт рантайму, чтобы сказать: «опроси меня снова, я могу продвинуться». |
| **задача** (task) | Future, который передали рантайму для самостоятельного выполнения (через `tokio::spawn`). Как очень лёгкий поток. |
| **блокировка** (blocking) | Действие, которое держит поток занятым или спящим, не давая рантайму запускать другие задачи. |
| **`Send`** | Маркерный трейт: значения этого типа можно переместить в другой поток. |
| **отмена** (cancellation) | Остановка future до его завершения. В Rust отменяют, уничтожая (drop) future. |
| **тайм-аут** (timeout) | Предел того, сколько вы готовы ждать. |

## 1. Проблема: программы, которые ждут

Представьте повара, который кипятит воду для чая. Воде нужно три минуты. *Блокирующий* повар стоит у кастрюли и три минуты ничего не делает. Если попросить его ещё и поджарить хлеб, он начнёт после того, как чай будет готов.

Ускориться можно двумя способами:

1. **Нанять больше поваров** — по одному на задачу. Это дают потоки (Урок 05.10). Это работает, но каждому повару нужно место на кухне (потоку нужна память, обычно несколько мегабайт адресного пространства под стек, а операционной системе нужно переключаться между ними).
2. **Один умный повар, который никогда не ждёт.** Он ставит воду на плиту и, пока она греется, жарит хлеб; когда чайник свистит, возвращается. Это **async** (асинхронное) программирование.

Сервер, который общается с 10 000 клиентов, — та же история. Большую часть времени каждое соединение *ждёт* сетевое сообщение. Десять тысяч потоков были бы тяжёлыми; десять тысяч async-задач — лёгкие.

Async нужен для **I/O-bound** работы: программа в основном ждёт. Для **CPU-bound** работы (тяжёлые вычисления) async не помогает, потому что ждать нечего. Сравним оба подхода в конце урока.

## 2. Что такое future

**Future** — это значение, которое означает «какая-то работа, которая завершится позже». Стандартная библиотека определяет его как трейт:

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

Единственная операция — `poll`. Кто-то (рантайм) вызывает `poll` снова и снова. Каждый раз future:

- возвращает `Poll::Ready(value)` — «я закончил, вот результат», или
- возвращает `Poll::Pending` — «ещё нет; я сообщил рантайму, когда меня вызвать снова» (через **waker** внутри `cx`).

`Pin<&mut Self>` означает: «изменяемая ссылка на future, который обещает не перемещаться в памяти». Future, созданные компилятором, могут хранить ссылки на собственные поля, поэтому после начала опроса они не должны перемещаться. Для этого урока достаточно знать *ритуал*: `pin!(future)` создаёт закреплённый future.

Чтобы убедиться, что здесь нет магии, напишем future вручную и крошечный исполнитель, который его опрашивает. (Настоящие рантаймы гораздо умнее. Свой вы построите на одном из следующих этапов.)

```rust
use std::future::Future;
use std::pin::{Pin, pin};
use std::task::{Context, Poll, Waker};

struct CountDown(u32);

impl Future for CountDown {
    type Output = &'static str;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.0 == 0 {
            Poll::Ready("liftoff")
        } else {
            println!("count: {}", self.0);
            self.0 -= 1;
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

fn block_on<F: Future>(future: F) -> F::Output {
    let mut future = pin!(future);
    let mut cx = Context::from_waker(Waker::noop());
    loop {
        if let Poll::Ready(value) = future.as_mut().poll(&mut cx) {
            return value;
        }
    }
}

fn main() {
    println!("{}", block_on(CountDown(3)));
}
```

**Что происходит:**

1. `CountDown(3)` — future, которому нужно четыре опроса, чтобы завершиться.
2. `block_on` — наш мини-исполнитель. Он закрепляет future, создаёт `Context` с «пустым» waker (`Waker::noop()`) и вызывает `poll` в цикле, пока ответ не станет `Ready`.
3. Первые три опроса печатают `count: 3`, `count: 2`, `count: 1`, уменьшают счётчик, вызывают `cx.waker().wake_by_ref()` («пожалуйста, опроси меня снова») и возвращают `Pending`.
4. Четвёртый опрос видит `0` и возвращает `Ready("liftoff")`. `block_on` возвращает это значение.

```text
count: 3
count: 2
count: 1
liftoff
```

## 3. `async fn` и `.await`

Писать `poll` вручную трудно. Компилятор делает это за вас. **`async fn`** при вызове не выполняется. Она возвращает future, который содержит тело:

```rust
use std::future::Future;
use std::pin::pin;
use std::task::{Context, Poll, Waker};

async fn add(a: i32, b: i32) -> i32 {
    println!("adding {a} and {b}");
    a + b
}

fn block_on<F: Future>(future: F) -> F::Output {
    let mut future = pin!(future);
    let mut cx = Context::from_waker(Waker::noop());
    loop {
        if let Poll::Ready(value) = future.as_mut().poll(&mut cx) {
            return value;
        }
    }
}

fn main() {
    let future = add(2, 3);
    println!("the future exists, nothing has run yet");
    let result = block_on(future);
    println!("result = {result}");
}
```

**Что происходит:**

1. `add(2, 3)` создаёт future. Тело **не** выполняется, поэтому пока ничего не напечатано.
2. Программа печатает `the future exists, nothing has run yet`.
3. `block_on` опрашивает future. Теперь тело выполняется: печатает `adding 2 and 3` и возвращает `5`.

```text
the future exists, nothing has run yet
adding 2 and 3
result = 5
```

Future **ленивые**: они ничего не делают, пока их не опросят. Это самый важный факт об async в Rust.

Внутри async-функции вы используете **`.await`** для другого future. Это значит: «опроси тот future; если он `Pending`, приостанови *эту* функцию и верни управление рантайму; когда внутренний future будет готов, продолжи с этого самого места».

Компилятор превращает async-функцию в **конечный автомат**. Каждый `.await` — место, где функция может остановиться, а потом продолжить:

```text
async fn example() {
    let a = step_one().await;     // state 0: waiting for step_one
    let b = step_two(a).await;    // state 1: waiting for step_two (a is kept)
    println!("{b}");              // state 2: done
}

       poll         poll                poll
 start ───► [wait one] ───► [wait two] ───► finished
            variables that live across an .await are stored inside the future
```

Локальные переменные, которые нужны после `.await`, хранятся внутри значения future. Запомните это для раздела 8.

## 4. Рантайм: Tokio

Кто вызывает `poll`? **Рантайм**. В стандартной библиотеке рантайма намеренно нет: для разных задач нужны разные устройства. Самый используемый — **Tokio**. Он даёт:

- **исполнитель**, который запускает много задач на нескольких потоках;
- таймеры (`tokio::time`), сетевой и файловый I/O, каналы, блокировки;
- макросы `#[tokio::main]` и `#[tokio::test]`.

Ваша первая программа на Tokio:

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn greet(name: &str, delay_ms: u64) -> String {
    sleep(Duration::from_millis(delay_ms)).await;
    format!("Hello, {name}!")
}

#[tokio::main]
async fn main() {
    let message = greet("Aziza", 100).await;
    println!("{message}");
}
```

**Что происходит:**

1. `greet` — async-функция. `sleep(...)` возвращает future, который становится готовым после задержки. Мы делаем для него `.await`.
2. Пока `greet` ждёт, поток свободен: рантайм мог бы запустить на нём другие задачи. Здесь их нет, поэтому он просто ждёт.
3. `main` сама по себе не может быть `async`, потому что кто-то должен создать рантайм. `#[tokio::main]` — это **атрибутный макрос** (Урок 06.8), который это делает.
4. `greet("Aziza", 100).await` приостанавливает `main`, пока приветствие не будет готово. Затем `main` его печатает.

```text
Hello, Aziza!
```

`cargo expand` (Урок 06.7) показывает, что написал `#[tokio::main]`. Это настоящее раскрытие в сокращении:

```rust
fn main() {
    let body = async {
        let message = greet("Aziza", 100).await;
        println!("{message}");
    };
    // ...
    {
        use tokio::runtime::Builder;
        return Builder::new_multi_thread()
            .enable_all()
            .build()
            .expect("Failed building the Runtime")
            .block_on(body);
    }
}
```

Тело вашей `async fn main` становится `async`-блоком. Обычная `main` строит рантайм и запускает этот блок через `block_on`, как наш мини-`block_on`, только намного мощнее. Никакой магии нет.

> 💡 **Совет:** Tokio разделён на *фичи*, чтобы компилировать только то, что вы используете. Для этого урока нужны `rt-multi-thread` (рантайм), `macros` (`#[tokio::main]`, `join!`, `select!`), `time` и `sync`. Если не уверены, `features = ["full"]` включает всё.

## 5. Делать несколько дел одновременно

### `join!`

Если вы делаете `.await` двух future один за другим, они выполняются **один за другим**. Чтобы запустить их одновременно, используйте `tokio::join!`:

```rust
use std::time::{Duration, Instant};
use tokio::time::sleep;

async fn work(name: &str, ms: u64) -> String {
    sleep(Duration::from_millis(ms)).await;
    format!("{name} done")
}

fn about(start: Instant) -> u128 {
    (start.elapsed().as_millis() + 50) / 100 * 100
}

#[tokio::main]
async fn main() {
    let start = Instant::now();
    let a = work("first", 200).await;
    let b = work("second", 200).await;
    println!("{a}, {b} after about {} ms", about(start));

    let start = Instant::now();
    let (a, b) = tokio::join!(work("first", 200), work("second", 200));
    println!("{a}, {b} after about {} ms", about(start));
}
```

**Что происходит:**

1. Первый блок полностью ждёт `work("first")` (200 мс), затем `work("second")` (200 мс): всего около 400 мс.
2. `tokio::join!` опрашивает оба future в одной задаче. Пока первый ждёт свой таймер, второй запускает свой. Оба ждут вместе: около 200 мс.
3. `about` округляет время до ближайших 100 мс, поэтому вывод стабилен, даже если реальное время 203 мс.

```text
first done, second done after about 400 ms
first done, second done after about 200 ms
```

### `spawn`

`tokio::spawn` передаёт future рантайму как независимую **задачу**. Он возвращает `JoinHandle`, для которого вы делаете `.await`, чтобы получить результат:

```rust
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    let mut handles = Vec::new();

    for id in 1..=3 {
        let handle = tokio::spawn(async move {
            sleep(Duration::from_millis(100 * (4 - id))).await;
            println!("task {id} finished");
            id * 10
        });
        handles.push(handle);
    }

    let mut total = 0;
    for handle in handles {
        total += handle.await.unwrap();
    }
    println!("total = {total}");
}
```

**Что происходит:**

1. Цикл создаёт три задачи. Задача 1 спит 300 мс, задача 2 — 200 мс, задача 3 — 100 мс. Все стартуют сразу.
2. Они завершаются в порядке своих таймеров, а не в порядке создания: 3, 2, 1.
3. `async move` перемещает `id` в задачу. Запущенная задача может пережить функцию, которая её создала, поэтому она не может заимствовать локальные переменные.
4. `handle.await` возвращает `Result<T, JoinError>`. Ветка `Err` возникает, если задача запаниковала. Здесь мы делаем `unwrap()`.
5. Результаты складываем в порядке дескрипторов: `10 + 20 + 30 = 60`.

```text
task 3 finished
task 2 finished
task 1 finished
total = 60
```

| | `join!` | `spawn` |
|---|---|---|
| Выполняется в | той же задаче (по одному, переключение на `.await`) | отдельных задачах (могут идти на разных потоках) |
| Может заимствовать локальные переменные | да | нет: нужны `'static` и `Send` |
| Результат | все результаты вместе | один `JoinHandle` на задачу |
| Подходит для | нескольких известных future в одной функции | множества или независимых работ |

**Зачем `Send` и `'static`?** Многопоточный рантайм может переместить задачу с одного потока на другой между двумя опросами, поэтому всё внутри должно быть `Send`. Задача может также выполняться после того, как порождающая функция вернулась, поэтому она не может хранить обычные ссылки на свои локальные переменные: `'static`.

## 6. Не блокируйте рантайм

У рантайма мало потоков и много задач. Задача возвращает поток только на `.await`. Если задача держит поток занятым (долгое вычисление, `std::thread::sleep`, блокирующее чтение файла), **все остальные задачи на этом потоке застревают**.

Чтобы увидеть это ясно, используем рантайм всего с одним потоком (`flavor = "current_thread"`). Сначала неправильный способ:

```rust
use std::time::{Duration, Instant};

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let start = Instant::now();

    let a = tokio::spawn(async {
        std::thread::sleep(Duration::from_millis(300)); // blocks the whole thread!
    });
    let b = tokio::spawn(async {
        std::thread::sleep(Duration::from_millis(300));
    });

    a.await.unwrap();
    b.await.unwrap();
    println!("two 300 ms tasks took about {} ms", (start.elapsed().as_millis() + 50) / 100 * 100);
}
```

```text
two 300 ms tasks took about 600 ms
```

**Что происходит:** `std::thread::sleep` усыпляет *поток*. Единственный поток не может запустить задачу `b`, пока спит в задаче `a`. Две задачи выполнились одна за другой: 600 мс. Правильный способ — async-сон:

```rust
use std::time::{Duration, Instant};

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let start = Instant::now();

    let a = tokio::spawn(async {
        tokio::time::sleep(Duration::from_millis(300)).await;
    });
    let b = tokio::spawn(async {
        tokio::time::sleep(Duration::from_millis(300)).await;
    });

    a.await.unwrap();
    b.await.unwrap();
    println!("two 300 ms tasks took about {} ms", (start.elapsed().as_millis() + 50) / 100 * 100);
}
```

```text
two 300 ms tasks took about 300 ms
```

`tokio::time::sleep(...).await` приостанавливает только *задачу*. Поток тем временем выполняет другую задачу.

Что делать, если вам нужна блокирующая или тяжёлая работа на CPU? Отправьте её в отдельный пул потоков через `spawn_blocking`:

```rust
fn slow_sum(n: u64) -> u64 {
    (1..=n).sum()
}

#[tokio::main]
async fn main() {
    let handle = tokio::task::spawn_blocking(|| slow_sum(10_000_000));
    println!("waiting for the heavy work...");
    let result = handle.await.unwrap();
    println!("sum = {result}");
}
```

```text
waiting for the heavy work...
sum = 50000005000000
```

> ⚡ **Производительность:** практическое правило: код между двумя `.await` должен выполняться микросекунды, а не миллисекунды. Если кусок кода работает дольше (хеширование большого файла, сжатие, разбор огромного документа), используйте `spawn_blocking` или обычный поток.

## 7. Каналы, тайм-ауты и `select!`

Задачи общаются через каналы, как и потоки. Канал `mpsc` в Tokio имеет асинхронные `send` и `recv`:

```rust
use std::time::Duration;
use tokio::sync::mpsc;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel::<String>(8);

    let producer = tokio::spawn(async move {
        for i in 1..=3 {
            sleep(Duration::from_millis(50)).await;
            tx.send(format!("message {i}")).await.unwrap();
        }
        // tx is dropped here, so the channel closes
    });

    while let Some(message) = rx.recv().await {
        println!("got: {message}");
    }
    producer.await.unwrap();
    println!("channel closed");
}
```

**Что происходит:**

1. `mpsc::channel(8)` создаёт канал, который вмещает не больше 8 сообщений. Когда он заполнен, `send(...).await` ждёт: это **обратное давление** (backpressure) — получатель притормаживает отправителя.
2. Задача-производитель отправляет три сообщения с интервалом 50 мс и завершается. Её `tx` уничтожается.
3. `rx.recv().await` возвращает `Some(message)`, пока приходят сообщения, и `None`, когда все отправители исчезли. Это завершает цикл `while let`.

```text
got: message 1
got: message 2
got: message 3
channel closed
```

**Тайм-аут** ограничивает время ожидания. `tokio::select!` ждёт **первого** из нескольких future и уничтожает (отменяет) остальные:

```rust
use std::time::Duration;
use tokio::time::{sleep, timeout};

async fn slow() -> &'static str {
    sleep(Duration::from_millis(500)).await;
    "finished"
}

#[tokio::main]
async fn main() {
    match timeout(Duration::from_millis(100), slow()).await {
        Ok(text) => println!("ok: {text}"),
        Err(error) => println!("gave up: {error}"),
    }

    tokio::select! {
        text = slow() => println!("slow won: {text}"),
        _ = sleep(Duration::from_millis(100)) => println!("the timer won"),
    }
}
```

```text
gave up: deadline has elapsed
the timer won
```

**Что происходит:** `timeout` возвращает `Err(Elapsed)`, когда время вышло; future `slow()` уничтожается, поэтому его работа прекращается. `select!` делает то же с любым набором future: первый завершившийся выполняет свою ветку, остальные уничтожаются. **Уничтожить future — значит отменить его** в Rust.

### Ошибки и `?`

`?` работает в async-функциях как обычно:

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn load_user(id: u32) -> Result<String, String> {
    sleep(Duration::from_millis(10)).await;
    if id == 0 {
        Err("user id 0 does not exist".to_string())
    } else {
        Ok(format!("user-{id}"))
    }
}

async fn greet(id: u32) -> Result<String, String> {
    let name = load_user(id).await?;
    Ok(format!("Hello, {name}"))
}

#[tokio::main]
async fn main() {
    println!("{:?}", greet(7).await);
    println!("{:?}", greet(0).await);
}
```

```text
Ok("Hello, user-7")
Err("user id 0 does not exist")
```

Пишите `.await?`: сначала дождитесь результата, потом пусть `?` вернётся раньше при `Err`.

## 8. Блокировки и `.await`

Охранник `std::sync::Mutex` не должен жить через `.await` в запущенной задаче. Посмотрите на ошибку:

```rust
use std::sync::Mutex;
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    let counter = std::sync::Arc::new(Mutex::new(0));
    let c = counter.clone();
    let handle = tokio::spawn(async move {
        let mut guard = c.lock().unwrap();
        sleep(Duration::from_millis(10)).await;
        *guard += 1;
    });
    handle.await.unwrap();
    println!("{}", counter.lock().unwrap());
}
```

```text
error: future cannot be sent between threads safely
   --> src/main.rs:9:18
    |
  9 |       let handle = tokio::spawn(async move {
    |  __________________^
    ...
    |
    = help: within `{async block@src/main.rs:9:31: 9:41}`, the trait `Send` is not implemented for `std::sync::MutexGuard<'_, i32>`
note: future is not `Send` as this value is used across an await
   --> src/main.rs:11:42
    |
 10 |         let mut guard = c.lock().unwrap();
    |             --------- has type `std::sync::MutexGuard<'_, i32>` which is not `Send`
 11 |         sleep(Duration::from_millis(10)).await;
    |                                          ^^^^^ await occurs here, with `mut guard` maybe used later
```

**Что это значит:** вспомните раздел 3: переменные, живущие через `.await`, хранятся в future. Охранник оказался бы в future, а задачу могут переместить на другой поток, но обычный `MutexGuard` должен освобождаться тем потоком, который его взял. Поэтому он не `Send`.

Есть два исправления. Либо освободить охранника до `.await` (лучше всего, если возможно):

```rust
use std::sync::{Arc, Mutex};
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0));
    let c = Arc::clone(&counter);
    let handle = tokio::spawn(async move {
        {
            let mut guard = c.lock().unwrap();
            *guard += 1;
        } // the guard is dropped here, before the await
        sleep(Duration::from_millis(10)).await;
    });
    handle.await.unwrap();
    println!("{}", counter.lock().unwrap());
}
```

```text
1
```

Либо использовать `tokio::sync::Mutex`, у которого `lock()` асинхронный, а охранника можно держать через `.await`:

```rust
use std::sync::Arc;
use std::time::Duration;
use tokio::sync::Mutex;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = Vec::new();
    for _ in 0..5 {
        let c = Arc::clone(&counter);
        handles.push(tokio::spawn(async move {
            let mut guard = c.lock().await;
            sleep(Duration::from_millis(10)).await;
            *guard += 1;
        }));
    }
    for handle in handles {
        handle.await.unwrap();
    }
    println!("{}", *counter.lock().await);
}
```

```text
5
```

> 📏 **Соглашение:** используйте обычный `std::sync::Mutex`, когда блокировка держится недолго и в нём нет async-кода. `tokio::sync::Mutex` берите только тогда, когда блокировку действительно нужно держать через `.await`.

## 9. Тестирование async-кода

Обычная функция `#[test]` не может быть `async`. Tokio даёт `#[tokio::test]`, который строит рантайм для каждого теста:

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn greet(name: &str, delay_ms: u64) -> String {
    sleep(Duration::from_millis(delay_ms)).await;
    format!("Hello, {name}!")
}

#[tokio::main]
async fn main() {
    println!("{}", greet("Aziza", 10).await);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn greet_builds_the_message() {
        assert_eq!(greet("Aziza", 1).await, "Hello, Aziza!");
    }

    #[tokio::test]
    async fn greet_takes_at_least_the_delay() {
        let start = std::time::Instant::now();
        greet("Bobur", 50).await;
        assert!(start.elapsed() >= Duration::from_millis(50));
    }
}
```

`cargo test` покажет `test result: ok. 2 passed; 0 failed`.

> 🧪 **Тестирование:** тесты с настоящим сном медленные и могут быть нестабильными. Tokio умеет останавливать свои часы в тестах (фича `test-util`, `#[tokio::test(start_paused = true)]`), так что 10-секундный сон не занимает реального времени. Делайте проверки времени с запасом, как `>=` в тесте выше, и никогда не требуйте точного значения.

## 10. Async или потоки?

| | Потоки (`std::thread`) | Async (Tokio) |
|---|---|---|
| Лучше всего для | тяжёлой работы на CPU, нескольких долгих задач | множества задач, которые в основном ждут I/O |
| Цена ещё одной задачи | стек (мегабайты адресного пространства) + работа операционной системы (OS) | небольшой future в памяти |
| Стиль кода | обычный код, который может блокироваться | `async`/`.await`; никогда не блокировать |
| Сложные места | блокировки, общие данные | `Send`/`'static`, случайная блокировка, отмена |
| Библиотеки | работает большая часть `std` | нужны async-версии I/O (Tokio, async-крейты) |

Ни то, ни другое не «лучше». Многие настоящие программы используют оба: async для сети, `spawn_blocking` или потоки для тяжёлых вычислений. Проект этого этапа ([Урок 06.11](11-project-multithreaded-web-server.md)) нарочно использует **потоки**: пул потоков — самый ясный способ понять, как сервер делит работу, а потом вы сможете сравнить с async. На следующих этапах мы вернёмся к async: вы построите крошечный рантайм и изучите Tokio подробно.

## Частые ошибки

**1. Забыли `.await`.** Future создаётся и выбрасывается. Ничего не выполняется:

```rust
async fn say(text: &str) {
    println!("{text}");
}

#[tokio::main]
async fn main() {
    say("hello");
    println!("end of main");
}
```

```text
warning: unused implementer of `Future` that must be used
  --> src/main.rs:11:5
   |
11 |     say("hello");
   |     ^^^^^^^^^^^^
   |
   = note: futures do nothing unless you `.await` or poll them
```

Исправление: `say("hello").await;`. Никогда не игнорируйте это предупреждение.

**2. `.await` вне async-функции.**

```text
error[E0728]: `await` is only allowed inside `async` functions and blocks
 --> src/main.rs:6:22
  |
5 | fn main() {
  | --------- this is not `async`
6 |     let x = answer().await;
  |                      ^^^^^ only allowed inside `async` functions and blocks
```

Исправление: сделайте функцию `async` или используйте `block_on` рантайма из обычного кода.

**3. `async fn main()` без рантайма.**

```text
error[E0752]: `main` function is not allowed to be `async`
 --> src/main.rs:1:1
  |
1 | async fn main() {
  | ^^^^^^^^^^^^^^^ `main` function is not allowed to be `async`
```

Исправление: добавьте над ней `#[tokio::main]`.

**4. Заимствование локальной переменной в запущенной задаче.**

```rust
#[tokio::main]
async fn main() {
    let name = String::from("Aziza");
    let handle = tokio::spawn(async {
        println!("hello, {name}");
    });
    handle.await.unwrap();
}
```

```text
error[E0373]: async block may outlive the current function, but it borrows `name`, which is owned by the current function
 --> src/main.rs:4:31
  |
4 |     let handle = tokio::spawn(async {
  |                               ^^^^^ may outlive borrowed value `name`
  |
help: to force the async block to take ownership of `name` (and any other referenced variables), use the `move` keyword
  |
4 |     let handle = tokio::spawn(async move {
  |                                     ++++
```

Исправление: используйте `async move`, как советует компилятор. Если значение нужно и потом, сначала сделайте `clone()`.

**5. Удержание `std::sync::MutexGuard` через `.await`** — см. раздел 8.

**6. Блокировка в async-коде** (`std::thread::sleep`, тяжёлые циклы, блокирующее чтение файлов) — см. раздел 6. Ошибки компилятора здесь нет. Единственный симптом — всё становится медленным. Поищите в async-коде `std::thread::sleep` и долгие циклы.

## Попробуйте сами

1. Откройте `~/projects/rust/code/p06-async` и вставьте первую программу на Tokio (раздел 4) в `src/main.rs`. Выполните `cargo run`: после короткой паузы вы должны увидеть `Hello, Aziza!`.
2. Вставьте пример с `join!` из раздела 5 и запустите. Убедитесь, что первая строка говорит около 400 мс, а вторая — около 200 мс.
3. Измените `tokio::join!` на три future по 200 мс. Сколько теперь займёт? (Около 200 мс: ожидание накладывается.)
4. Замените `tokio::time::sleep` на `std::thread::sleep` в одном из future в `join!` (уберите `.await`). Что станет со временем? Почему? (Станет около 400 мс: блокирующий сон не даёт другому future работать.)
5. Сохраните работу в учебном репозитории: `git add code/p06-async && git commit -m "feat(p06-async): add tokio join and spawn examples"`.

## Упражнения

### Упражнение 1 — Два удвоителя (легко)

Напишите `async fn double_after(ms: u64, n: i32) -> i32`, которая спит `ms` миллисекунд и возвращает `n * 2`. В `main` запустите `double_after(100, 4)` и `double_after(100, 5)` **одновременно** через `join!` и выведите `8 + 10 = 18`.

<details>
<summary>Подсказка</summary>

`let (a, b) = tokio::join!(first_future, second_future);` возвращает оба результата в кортеже.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn double_after(ms: u64, n: i32) -> i32 {
    sleep(Duration::from_millis(ms)).await;
    n * 2
}

#[tokio::main]
async fn main() {
    let (a, b) = tokio::join!(double_after(100, 4), double_after(100, 5));
    println!("{a} + {b} = {}", a + b);
}
```

```text
8 + 10 = 18
```

Почему это работает: оба future одновременно ждут свои 100 мс, поэтому вся программа занимает около 100 мс, а не 200.

</details>

### Упражнение 2 — Исправьте блокировку (средне)

Программа из раздела 8 не компилируется (`future cannot be sent between threads safely`). Исправьте её третьим способом, не таким, как два показанных: оставьте `std::sync::Mutex`, но сначала сделайте `.await`, а блокировку берите только после него.

<details>
<summary>Подсказка</summary>

Проблема только в том, что охранник жив, пока задача ждёт. Переместите строку `sleep(...).await` **выше** строки `c.lock()`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::sync::{Arc, Mutex};
use std::time::Duration;
use tokio::time::sleep;

#[tokio::main]
async fn main() {
    let counter = Arc::new(Mutex::new(0));
    let c = Arc::clone(&counter);
    let handle = tokio::spawn(async move {
        sleep(Duration::from_millis(10)).await;
        let mut guard = c.lock().unwrap();
        *guard += 1;
    });
    handle.await.unwrap();
    println!("{}", counter.lock().unwrap());
}
```

```text
1
```

Почему это работает: теперь, когда задача приостанавливается на `.await`, охранника нет, поэтому future — `Send`. Блокировка держится только на короткое, не асинхронное `*guard += 1`.

</details>

### Упражнение 3 — `fetch_all` с тестами (средне)

Напишите `async fn fetch(id: u32) -> String`, которая немного спит и возвращает `format!("item-{id}")`, и `async fn fetch_all(ids: Vec<u32>) -> Vec<String>`, которая загружает все id **конкурентно** через `tokio::spawn`, но возвращает результаты **в том же порядке, что и id**. Напишите два теста `#[tokio::test]`: один на порядок (`vec![3, 1, 2]`), другой на пустой ввод.

<details>
<summary>Подсказка</summary>

Сначала запустите все задачи и соберите `JoinHandle` в `Vec`. Только потом пройдите по дескрипторам по порядку и сделайте `.await` для каждого. `fetch` принимает `id` по значению, поэтому он `'static` и его можно запускать напрямую: `tokio::spawn(fetch(id))`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::time::Duration;
use tokio::time::sleep;

async fn fetch(id: u32) -> String {
    // pretend that bigger ids are slower
    sleep(Duration::from_millis(100 - 10 * id as u64)).await;
    format!("item-{id}")
}

async fn fetch_all(ids: Vec<u32>) -> Vec<String> {
    let mut handles = Vec::new();
    for id in ids {
        handles.push(tokio::spawn(fetch(id)));
    }
    let mut results = Vec::new();
    for handle in handles {
        results.push(handle.await.unwrap());
    }
    results
}

#[tokio::main]
async fn main() {
    println!("{:?}", fetch_all(vec![1, 2, 3]).await);
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn results_keep_the_order_of_the_ids() {
        let results = fetch_all(vec![3, 1, 2]).await;
        assert_eq!(results, vec!["item-3", "item-1", "item-2"]);
    }

    #[tokio::test]
    async fn empty_input_gives_empty_output() {
        assert!(fetch_all(vec![]).await.is_empty());
    }
}
```

```text
["item-1", "item-2", "item-3"]
```

`cargo test` покажет `test result: ok. 2 passed`.

Почему это работает: все задачи стартуют до первого `await`, поэтому выполняются конкурентно, хотя задача 3 заканчивается первой. Ожидание дескрипторов в исходном порядке даёт результаты в этом порядке.

</details>

### Упражнение 4 — Сдаться через 100 мс (средне)

Напишите `async fn slow_square(n: u64) -> u64`, которая спит `50 * n` миллисекунд и возвращает `n * n`. Для `n` из `[1, 3]` вызовите её с `timeout` 100 мс. Выведите `1 squared is 1` и `3: too slow, gave up`.

<details>
<summary>Подсказка</summary>

`tokio::time::timeout(duration, future).await` возвращает `Result<T, Elapsed>`. Используйте `match`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::time::Duration;
use tokio::time::{sleep, timeout};

async fn slow_square(n: u64) -> u64 {
    sleep(Duration::from_millis(50 * n)).await;
    n * n
}

#[tokio::main]
async fn main() {
    for n in [1, 3] {
        match timeout(Duration::from_millis(100), slow_square(n)).await {
            Ok(value) => println!("{n} squared is {value}"),
            Err(_) => println!("{n}: too slow, gave up"),
        }
    }
}
```

```text
1 squared is 1
3: too slow, gave up
```

Почему это работает: при `n = 1` работе нужно 50 мс, меньше предела. При `n = 3` нужно 150 мс, поэтому `timeout` возвращает `Err` и уничтожает future `slow_square`.

</details>

### Упражнение 5 — Посмотрите, как future чередуются (сложно)

Напишите future `YieldN { name, left }` (как `CountDown` в разделе 2), который возвращает `Pending` `left` раз, а затем `Ready(name)`. Печатайте строку при каждом опросе. Запустите два таких future, `A` с 2 и `B` с 3, через `tokio::join!`. Что вы замечаете в напечатанных строках?

<details>
<summary>Подсказка</summary>

Скопируйте `CountDown`. Добавьте поле `name: &'static str` и печатайте `"{} polled, {} left"`. Не забудьте `cx.waker().wake_by_ref()` перед возвратом `Pending`; без него Tokio никогда не опросит future снова.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct YieldN {
    name: &'static str,
    left: u32,
}

impl Future for YieldN {
    type Output = &'static str;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.left == 0 {
            Poll::Ready(self.name)
        } else {
            println!("{} polled, {} left", self.name, self.left);
            self.left -= 1;
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

#[tokio::main]
async fn main() {
    let (a, b) = tokio::join!(
        YieldN { name: "A", left: 2 },
        YieldN { name: "B", left: 3 },
    );
    println!("{a} and {b} are done");
}
```

Настоящий запуск напечатал вот что (30 запусков подряд дали тот же порядок). Для справедливости `join!` опрашивает future по очереди и начинает каждый новый круг со следующего future, чтобы ни один из них не оказывался всегда последним. Это называется *round-robin* («по кругу»):

```text
A polled, 2 left
B polled, 3 left
B polled, 2 left
A polled, 1 left
B polled, 1 left
A and B are done
```

Почему это работает: строки `A` и `B` перемешаны. `join!` опрашивает каждый future по очереди, и каждый возвращает управление через `Pending`. Это **кооперативное** планирование: задачи ходят по очереди, потому что добровольно возвращаются в точках `.await`. Это же показывает, почему future, который никогда не возвращается (блокировка), морит голодом все остальные.

</details>

## Вопросы для самопроверки

1. Что возвращает `async fn` при вызове? Когда выполняется её тело?

<details>
<summary>Ответ</summary>

Она возвращает future (тело хранится в нём). Тело выполняется только тогда, когда future опрашивают, например когда вы делаете для него `.await` или передаёте рантайму. До этого ничего не происходит.

</details>

2. Какие два возможных результата у `poll`? Что означает каждый?

<details>
<summary>Ответ</summary>

`Poll::Ready(value)`: работа завершена, вот результат. `Poll::Pending`: ещё не готово; future договорился (через waker), чтобы его опросили снова, когда он сможет продвинуться.

</details>

3. Почему стандартная библиотека Rust сама не запускает future?

<details>
<summary>Ответ</summary>

В Rust нет встроенного рантайма. Рантаймы — это библиотеки (Tokio и другие), потому что разным программам нужны разные устройства. Стандартная библиотека определяет только трейт `Future` и базовые типы.

</details>

4. Чем отличаются `join!` и `spawn`?

<details>
<summary>Ответ</summary>

`join!` выполняет несколько future конкурентно внутри текущей задачи и возвращает все результаты вместе; он может заимствовать локальные данные. `spawn` создаёт независимую задачу, которая может идти на другом потоке, поэтому её future должен быть `Send + 'static`.

</details>

5. Почему `std::thread::sleep` — ошибка внутри async-функции?

<details>
<summary>Ответ</summary>

Он блокирует весь поток, поэтому рантайм не может запускать другие задачи на этом потоке, пока он спит. Используйте `tokio::time::sleep(...).await`, который приостанавливает только задачу.

</details>

6. Как отменить future в Rust?

<details>
<summary>Ответ</summary>

Уничтожить его (drop). `timeout` и `select!` делают это с future, которые не завершились.

</details>

7. Когда лучше выбрать потоки, а не async?

<details>
<summary>Ответ</summary>

Для тяжёлой работы на CPU, для небольшого числа долгих задач или когда вы используете блокирующие библиотеки. Async лучше для множества задач, которые в основном ждут I/O.

</details>

## Итоги

- **Future** — ленивая работа, которая завершится позже; рантайм **опрашивает** его, пока он не вернёт `Ready`.
- `async fn` возвращает future; `.await` приостанавливает текущую функцию и позволяет рантайму выполнять другую работу.
- Компилятор превращает async-функцию в конечный автомат. Переменные, которые используются после `.await`, живут внутри future.
- В Rust нет встроенного рантайма: используйте Tokio (`#[tokio::main]`, `#[tokio::test]`).
- `join!` выполняет future одновременно в одной задаче; `spawn` создаёт независимые задачи, которым нужны `Send + 'static`.
- Никогда не блокируйте внутри async-кода. Используйте `tokio::time::sleep`, `spawn_blocking` или поток.
- Не держите `std::sync::MutexGuard` через `.await`; сначала освободите его или используйте `tokio::sync::Mutex`.
- Уничтожение future отменяет его. Это используют `timeout` и `select!`.
- Используйте async для множества задач с упором на I/O, а потоки — для задач с упором на CPU.

## Источники и что почитать

- [The Rust Programming Language — Chapter 17: Fundamentals of Asynchronous Programming](https://doc.rust-lang.org/book/ch17-00-async-await.html) — главы официальной книги о future, задачах и потоках данных.
- [Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/) — официальная книга об async: подробнее о `Pin`, waker и рантаймах.
- [Tokio tutorial](https://tokio.rs/tokio/tutorial) — пошаговое руководство от проекта Tokio.
- [tokio on docs.rs](https://docs.rs/tokio/latest/tokio/) — документация API (Application Programming Interface, программного интерфейса): `spawn`, `join!`, `select!`, `sync`, `time`.
- [std::future::Future](https://doc.rust-lang.org/std/future/trait.Future.html) — определение трейта в стандартной библиотеке.

---

[🏠 Начало](../START-HERE.md) · [← 06.8 Процедурные макросы](08-procedural-macros.md) · [📋 Этап 06](README.md) · [06.10 FFI: вызов кода на C →](10-ffi-calling-c.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
