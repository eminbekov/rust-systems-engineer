<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 14.12 Pin и самоссылающиеся future](12-pin-and-self-referential-futures.md) · [📋 Этап 14](README.md) · [14.14 Отмена, тайм-ауты и корректное завершение →](14-cancellation-timeouts-and-graceful-shutdown.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/14-concurrency-and-async/13-tokio-tasks-and-channels.md) <!-- nav -->

# Урок 14.13 — Tokio подробно: задачи и каналы

> **Этап 14 — Конкурентность и асинхронность подробно** · Урок 13 из 16 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Два рантайма Tokio (многопоточный и однопоточный) и как построить любой из них через `Builder`.
- Всё о результате задачи: `JoinHandle`, `JoinError` для паник и отмен и `JoinSet` для множества задач.
- Как выполнять блокирующую или тяжёлую работу через `spawn_blocking`, не замораживая другие задачи.
- Четыре канала Tokio — `mpsc`, `oneshot`, `broadcast` и `watch`, — какой из них выбрать и как `Semaphore` ограничивает число одновременно работающих задач.
- Паттерн *актор*: одна задача владеет состоянием, остальные посылают ей сообщения.

## Перед началом

- Вы прошли [Урок 14.12 — Pin и самоссылающиеся future](12-pin-and-self-referential-futures.md).
- Вы знаете `tokio::spawn`, `join!`, `select!`, `mpsc`, тайм-ауты и `#[tokio::test]` из [Урока 06.9 — Введение в async/await](../06-rust-advanced/09-async-await-introduction.md). Этот урок идёт глубже.
- Вы знаете, как рантайм устроен внутри, из [Урока 14.11](11-build-a-tiny-async-runtime.md): очередь готовых задач, waker, кооперативное планирование.
- Код этого урока находится в `~/projects/rust/code/p14-tokio`: библиотека с программами в `src/bin/`:

```bash
cd ~/projects/rust
cargo new --lib code/p14-tokio
cd code/p14-tokio
cargo add tokio@1 --features full
mkdir src/bin
```

Фича `full` включает все части Tokio (рантайм, таймеры, каналы, сигналы, I/O: input/output, ввод-вывод). Для учёбы это удобно; настоящий сервис перечисляет только нужные ему фичи.

## Новые слова

| Слово | Значение |
|---|---|
| **многопоточный рантайм** (multi-thread runtime) | Рантайм Tokio по умолчанию: пул рабочих потоков (по умолчанию по одному на ядро CPU, Central Processing Unit, центральный процессор), которые делят задачи и крадут работу друг у друга. |
| **однопоточный рантайм** (current-thread runtime) | Рантайм Tokio, который выполняет все задачи в одном потоке — том, что вызвал `block_on`, как ваш рантайм из Урока 14.11. |
| **`JoinHandle<T>`** | То, что возвращает `tokio::spawn`. Ожидание его через `.await` даёт `Result<T, JoinError>`. |
| **`JoinError`** | Ошибка, которую вы получаете, если задача запаниковала или была прервана (отменена). |
| **прервать** (abort) | Отменить задачу снаружи через `handle.abort()`. Задача выбрасывается на своём следующем `.await`. |
| **отсоединить** (detach) | Позволить задаче работать, не храня её дескриптор. Выбрасывание `JoinHandle` отсоединяет задачу, но **не** останавливает её. |
| **`JoinSet`** | Набор задач, результаты которых можно собирать в порядке их завершения. |
| **пул блокирующих потоков** (blocking pool) | Отдельный набор потоков, который Tokio держит для `spawn_blocking`, чтобы блокирующая работа не останавливала рабочие потоки. |
| **актор** (actor) | Задача, которая владеет каким-то состоянием и меняет его только в ответ на сообщения из канала. |
| **отставание** (lagging) | Получатель `broadcast`, который так медлителен, что старые сообщения перезаписались до того, как он их прочитал. |
| **семафор** (semaphore, `Semaphore`) | Счётчик разрешений, который ограничивает, сколько задач могут делать что-то одновременно. |
| **API** (Application Programming Interface, программный интерфейс) | Функции и типы, которые библиотека предлагает вашему коду. |

## 1. Два вида рантайма

`#[tokio::main]` из Урока 06.9 строит **многопоточный** рантайм. Оба вида можно построить вручную через `tokio::runtime::Builder`. Создайте `src/bin/flavors.rs`:

```rust
use std::time::Duration;

async fn show_threads(label: &str) {
    let mut handles = Vec::new();
    for id in 0..4 {
        handles.push(tokio::spawn(async move {
            tokio::time::sleep(Duration::from_millis(10)).await;
            let thread = std::thread::current();
            format!(
                "task {id} ran on {:?} {:?}",
                thread.name().unwrap_or("?"),
                thread.id()
            )
        }));
    }
    println!("{label}:");
    for handle in handles {
        println!("  {}", handle.await.unwrap());
    }
}

fn main() {
    let multi = tokio::runtime::Builder::new_multi_thread()
        .worker_threads(2)
        .enable_all()
        .build()
        .unwrap();
    multi.block_on(show_threads("multi-thread runtime, 2 workers"));

    let single = tokio::runtime::Builder::new_current_thread()
        .enable_all()
        .build()
        .unwrap();
    single.block_on(show_threads("current-thread runtime"));
}
```

`cargo run --bin flavors` (какая задача попадёт на какой рабочий поток, меняется от запуска к запуску):

```text
multi-thread runtime, 2 workers:
  task 0 ran on "tokio-rt-worker" ThreadId(3)
  task 1 ran on "tokio-rt-worker" ThreadId(2)
  task 2 ran on "tokio-rt-worker" ThreadId(3)
  task 3 ran on "tokio-rt-worker" ThreadId(3)
current-thread runtime:
  task 0 ran on "main" ThreadId(1)
  task 1 ran on "main" ThreadId(1)
  task 2 ran on "main" ThreadId(1)
  task 3 ran on "main" ThreadId(1)
```

**Что происходит:**

1. `new_multi_thread().worker_threads(2)` запускает два рабочих потока с именем `tokio-rt-worker`. Четыре задачи распределяются по обоим (`ThreadId(2)` и `ThreadId(3)`).
2. `enable_all()` включает таймер и реактор I/O. Без этого `sleep` запаниковал бы, потому что таймера нет.
3. `new_current_thread()` вообще не создаёт потоков: каждая задача выполняется в потоке, который вызвал `block_on`, здесь — в `main`. Это ваш рантайм из Урока 14.11, только взрослый.
4. `block_on` доводит future до конца — так же, как это делает за вас `#[tokio::main]`.

У атрибута те же варианты: `#[tokio::main(flavor = "current_thread")]` или `#[tokio::main(worker_threads = 2)]`. Какой использовать?

| | многопоточный | однопоточный |
|---|---|---|
| использует | все ядра CPU | один поток |
| задачи должны быть `Send` | да (они переходят между потоками) | `tokio::spawn` всё равно требует `Send`; `LocalSet` разрешает задачи без `Send` |
| подходит для | серверов, большинства программ | тестов, маленьких утилит, встраивания в программу со своими потоками |

`#[tokio::test]` по умолчанию использует **однопоточный** рантайм. Помните об этом: тест с блокирующим кодом может зависнуть там, где настоящая программа лишь замедлится.

## 2. Задачи и их результаты

`JoinHandle<T>` — это future. Ожидание его даёт `Result<T, JoinError>`: `Ok` со значением задачи или `Err`, если задача запаниковала или была прервана. Создайте `src/bin/join_errors.rs`:

```rust
use std::time::Duration;

#[tokio::main]
async fn main() {
    let ok = tokio::spawn(async { 6 * 7 });
    let panics = tokio::spawn(async {
        let numbers: Vec<u32> = Vec::new();
        numbers[0] // index out of bounds: the task panics
    });
    let slow = tokio::spawn(async {
        tokio::time::sleep(Duration::from_secs(10)).await;
        "never printed"
    });
    slow.abort();

    println!("ok:     {:?}", ok.await);
    match panics.await {
        Ok(value) => println!("value {value}"),
        Err(error) => println!("panics: is_panic = {}", error.is_panic()),
    }
    match slow.await {
        Ok(text) => println!("{text}"),
        Err(error) => println!("slow:   is_cancelled = {}", error.is_cancelled()),
    }
}
```

```text

thread 'tokio-rt-worker' (5675703) panicked at src/bin/join_errors.rs:8:16:
index out of bounds: the len is 0 but the index is 0
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
ok:     Ok(42)
panics: is_panic = true
slow:   is_cancelled = true
```

**Что происходит:**

1. Сообщение о панике печатает рабочий поток, но программа **не** падает. Tokio перехватывает панику и сохраняет её в результате задачи, поэтому одна плохая задача не может уронить весь сервер.
2. `panics.await` даёт `Err(JoinError)` с `is_panic() == true`. (`error.into_panic()` вернул бы значение паники, если вы хотите поднять её снова.)
3. `slow.abort()` просит Tokio отменить задачу. На следующем `.await` (здесь — `sleep`) future задачи выбрасывается. Затем её дескриптор даёт `Err` с `is_cancelled() == true`. Выбрасывание future — так в Rust устроена любая отмена; Урок 14.14 строится на этом.

Когда вы запускаете много задач, собирать их дескрипторы в `Vec` и ждать по порядку медленно, если первая задача самая долгая. **`JoinSet`** отдаёт результаты в порядке **завершения** задач. Создайте `src/bin/join_set.rs`:

```rust
use std::time::Duration;
use tokio::task::JoinSet;

async fn fetch(id: u64) -> (u64, u64) {
    // Pretend to load something; later ids are faster.
    tokio::time::sleep(Duration::from_millis(100 - id * 20)).await;
    (id, id * 10)
}

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();
    for id in 1..=4 {
        set.spawn(fetch(id));
    }
    let mut total = 0;
    while let Some(result) = set.join_next().await {
        let (id, value) = result.unwrap();
        println!("finished: id {id} -> {value}");
        total += value;
    }
    println!("total = {total}");
}
```

```text
finished: id 4 -> 40
finished: id 3 -> 30
finished: id 2 -> 20
finished: id 1 -> 10
total = 100
```

`join_next()` возвращает `Some(Result<T, JoinError>)` для каждой завершившейся задачи и `None`, когда набор пуст. Есть ещё одно свойство, которое делает `JoinSet` безопаснее отдельных дескрипторов: **когда `JoinSet` выбрасывается, все его задачи прерываются**. Выброшенный отдельный `JoinHandle` лишь отсоединяет свою задачу, и она продолжает работать в фоне.

## 3. Блокирующая работа: `spawn_blocking`

Из Урока 14.11 вы знаете, что планирование кооперативное: задача, которая не доходит до `.await`, держит свой поток. Вот как это выглядит в Tokio — в однопоточном рантайме, где это легче всего увидеть. Создайте `src/bin/blocking.rs`:

```rust
use std::time::{Duration, Instant};

/// Stands for heavy, blocking work: a long computation or a blocking library call.
fn heavy_work() -> u64 {
    std::thread::sleep(Duration::from_millis(300));
    (1..=1_000_u64).sum()
}

async fn ticker(start: Instant) {
    for i in 1..=3 {
        tokio::time::sleep(Duration::from_millis(100)).await;
        println!("  tick {i} at {} ms", start.elapsed().as_millis());
    }
}

#[tokio::main(flavor = "current_thread")]
async fn main() {
    println!("blocking inside a task:");
    let start = Instant::now();
    let ticks = tokio::spawn(ticker(start));
    tokio::task::yield_now().await; // let the ticker start
    let sum = heavy_work(); // BAD: blocks the only runtime thread
    println!("  sum = {sum} at {} ms", start.elapsed().as_millis());
    ticks.await.unwrap();

    println!("with spawn_blocking:");
    let start = Instant::now();
    let ticks = tokio::spawn(ticker(start));
    let sum = tokio::task::spawn_blocking(heavy_work).await.unwrap();
    println!("  sum = {sum} at {} ms", start.elapsed().as_millis());
    ticks.await.unwrap();
}
```

```text
blocking inside a task:
  sum = 500500 at 303 ms
  tick 1 at 303 ms
  tick 2 at 405 ms
  tick 3 at 508 ms
with spawn_blocking:
  tick 1 at 100 ms
  tick 2 at 202 ms
  sum = 500500 at 303 ms
  tick 3 at 305 ms
```

**Что происходит:**

1. В первой части `heavy_work()` выполняется прямо внутри задачи `main`. `std::thread::sleep` блокирует единственный поток рантайма на 300 мс. Задача `ticker` должна печатать на 100-й миллисекунде, но не может выполниться: его первый тик появляется на 303 мс, и все следующие тоже опаздывают.
2. Во второй части `spawn_blocking(heavy_work)` выполняет функцию в потоке из **пула блокирующих потоков** и возвращает `JoinHandle`. Поток рантайма свободен, поэтому `ticker` печатает вовремя, пока идёт работа.
3. `spawn_blocking` принимает обычное замыкание или функцию, а не future: код внутри может блокироваться сколько угодно.

Используйте `spawn_blocking` для блокирующих библиотек (некоторые API баз данных или файлов), вызовов `std::fs` в горячем цикле и вычислений дольше примерно 100 микросекунд. Для тяжёлых параллельных вычислений часто лучше подходит пул потоков `rayon` ([Урок 14.4](04-thread-pools-scoped-threads-and-rayon.md)); он создан для работы CPU.

## 4. Четыре вида каналов

`mpsc` вы знаете из Урока 06.9: много отправителей, один получатель, ограниченная ёмкость. В Tokio есть ещё три, каждый для своей формы общения:

| Канал | Отправители → получатели | Каждое сообщение достаётся | Типичное применение |
|---|---|---|---|
| `mpsc` | много → один | единственному получателю | очередь работы или команд; ограниченная ёмкость даёт обратное давление |
| `oneshot` | один → один | получателю, ровно один раз | ответ на один запрос |
| `broadcast` | много → много | **каждому** получателю | события, которые должны увидеть все подписчики (сообщения чата, уведомления) |
| `watch` | много → много | получатели видят только **последнее** значение | конфигурация, состояние, флаг «завершаемся» |

Создайте `src/bin/channels.rs`:

```rust
use std::time::Duration;
use tokio::sync::{broadcast, oneshot, watch};

#[tokio::main]
async fn main() {
    // oneshot: exactly one value, from one sender to one receiver.
    let (reply_tx, reply_rx) = oneshot::channel::<u32>();
    tokio::spawn(async move {
        reply_tx.send(42).unwrap();
    });
    println!("oneshot got {}", reply_rx.await.unwrap());

    // broadcast: every receiver gets every message.
    let (news_tx, mut first) = broadcast::channel::<String>(16);
    let mut second = news_tx.subscribe();
    news_tx.send("hello".to_string()).unwrap();
    news_tx.send("world".to_string()).unwrap();
    for _ in 0..2 {
        let a = first.recv().await.unwrap();
        let b = second.recv().await.unwrap();
        println!("broadcast: first got {a}, second got {b}");
    }

    // watch: only the latest value; receivers see changes.
    let (config_tx, mut config_rx) = watch::channel(1_u32);
    let watcher = tokio::spawn(async move {
        while config_rx.changed().await.is_ok() {
            println!("watch: config is now {}", *config_rx.borrow_and_update());
        }
        println!("watch: sender is gone");
    });
    for level in 2..=3 {
        tokio::time::sleep(Duration::from_millis(10)).await;
        config_tx.send(level).unwrap();
    }
    tokio::time::sleep(Duration::from_millis(10)).await;
    drop(config_tx);
    watcher.await.unwrap();
}
```

```text
oneshot got 42
broadcast: first got hello, second got hello
broadcast: first got world, second got world
watch: config is now 2
watch: config is now 3
watch: sender is gone
```

**Что происходит:**

1. `oneshot::channel` даёт отправителя, чей `send` принимает `self`: его можно использовать один раз. Получатель сам является future; `.await` даёт `Ok(value)` или `Err`, если отправителя выбросили, ничего не отправив.
2. `broadcast::channel(16)` хранит до 16 сообщений. Каждый получатель (первый и каждый созданный через `subscribe()`) получает свою копию каждого сообщения, поэтому тип сообщения должен быть `Clone`.
3. `watch::channel(1)` хранит одно значение. `changed().await` ждёт, пока не будет отправлено новое значение, а `borrow_and_update()` читает его и отмечает как увиденное. Если отправитель очень быстро отправит 2 и 3, медленный получатель может увидеть только 3: `watch` — это «какое значение сейчас», а не «каждое значение».
4. Когда отправителя `watch` выбрасывают, `changed()` возвращает `Err`, и цикл заканчивается. Так работает каждый канал Tokio: если другой стороны нет, вы получаете ошибку или `None`, а не зависание.

Что происходит, если получатель `broadcast` слишком медленный? Канал его не ждёт (иначе один медленный подписчик останавливал бы всех) — он перезаписывает самые старые сообщения. При ёмкости 2 и пяти сообщениях, отправленных до того, как получатель что-то прочитал, его следующий `recv()` возвращает `Err(RecvError::Lagged(3))`: «вы пропустили 3 сообщения». Следующие вызовы возвращают 4 и 5 — самые старые из оставшихся сообщений, а затем `Err(RecvError::Closed)`, когда все отправители исчезли.

> ⚡ **Производительность:** `mpsc::unbounded_channel()` никогда не заставляет отправителя ждать, и это кажется удобным. Но если получатель медленнее отправителей, очередь растёт, пока у процесса не кончится память. Предпочитайте ограниченный `mpsc::channel(n)`: когда он полон, `send().await` ждёт, и медленный потребитель автоматически замедляет производителей. Это **обратное давление** (backpressure), и на нём построен Урок 14.15.

**Ограничение работы, а не сообщений: `Semaphore`.** Ограниченный канал ограничивает, сколько *сообщений* ждёт. Иногда нужно ограничить, сколько *задач* делают что-то одновременно: не больше 2 загрузок, не больше 100 открытых соединений. `tokio::sync::Semaphore::new(n)` (семафор) хранит `n` **разрешений** (permits). `acquire().await` ждёт, пока освободится разрешение, и возвращает его как guard-объект; уничтожение guard-объекта возвращает разрешение. У `Arc<Semaphore>` метод `acquire_owned()` возвращает разрешение, которое владеет своим `Arc`, поэтому его можно переместить в запущенную задачу. `try_acquire()` никогда не ждёт: он сразу возвращает ошибку, если разрешений не осталось. Создайте `src/bin/semaphore.rs`:

```rust
use std::sync::Arc;
use std::time::{Duration, Instant};
use tokio::sync::Semaphore;
use tokio::task::JoinSet;

#[tokio::main]
async fn main() {
    let limit = Arc::new(Semaphore::new(2)); // at most 2 downloads at the same time
    let start = Instant::now();
    let mut set = JoinSet::new();
    for id in 1..=5 {
        let limit = Arc::clone(&limit);
        set.spawn(async move {
            // Waits until one of the 2 permits is free; the permit moves into the task.
            let _permit = limit.acquire_owned().await.expect("semaphore is open");
            tokio::time::sleep(Duration::from_millis(100)).await; // the "download"
            println!("download {id} done at {} ms", start.elapsed().as_millis());
        }); // `_permit` is dropped here, and the next waiting task gets it
    }
    while set.join_next().await.is_some() {}

    // try_acquire never waits: it fails at once when no permit is left.
    let _first = limit.try_acquire().expect("2 permits are free again");
    let _second = limit.try_acquire().expect("one permit is still free");
    println!("third try_acquire: {:?}", limit.try_acquire().err());
}
```

```text
download 2 done at 102 ms
download 1 done at 102 ms
download 5 done at 205 ms
download 4 done at 206 ms
download 3 done at 308 ms
third try_acquire: Some(NoPermits)
```

Пять задач стартуют сразу, но разрешение в каждый момент держат только две, поэтому они заканчивают группами: две, две, одна, с интервалом около 100 мс. Какие номера приходят первыми, меняется от запуска к запуску; сама картина — нет. Разрешение возвращается, как бы задача ни закончилась, даже при панике или `abort`, потому что оно возвращается в `Drop`. Уроки 15.6 и 15.15 используют семафор, чтобы ограничить число соединений.

## 5. Паттерн «актор»

Многим задачам нужно менять одну общую вещь, например счётчики. Можно использовать `Arc<Mutex<HashMap<…>>>`. Паттерн **актор** — другой способ: одна задача владеет таблицей, а все остальные посылают ей *команды* через канал `mpsc`. Каждая команда несёт отправителя `oneshot` для ответа. Блокировка не нужна, потому что таблицу трогает только одна задача.

Поместите это в `src/lib.rs`:

```rust
//! An actor: one task owns the state, others send it commands.

use std::collections::HashMap;
use tokio::sync::{mpsc, oneshot};

/// The messages the actor understands.
enum Command {
    Increment {
        key: String,
        reply: oneshot::Sender<u64>,
    },
    Get {
        key: String,
        reply: oneshot::Sender<Option<u64>>,
    },
}

/// The actor itself: it owns the map, nobody else can touch it.
async fn run_counter(mut commands: mpsc::Receiver<Command>) {
    let mut counts: HashMap<String, u64> = HashMap::new();
    while let Some(command) = commands.recv().await {
        match command {
            Command::Increment { key, reply } => {
                let count = counts.entry(key).or_insert(0);
                *count += 1;
                let _ = reply.send(*count); // the caller may be gone; that is fine
            }
            Command::Get { key, reply } => {
                let _ = reply.send(counts.get(&key).copied());
            }
        }
    }
    // All handles were dropped: the channel is closed, the actor ends.
}

/// A cheap, cloneable handle that talks to the actor.
#[derive(Clone)]
pub struct CounterHandle {
    commands: mpsc::Sender<Command>,
}

impl CounterHandle {
    /// Starts the actor task. Must be called inside a Tokio runtime.
    pub fn spawn() -> Self {
        let (commands, receiver) = mpsc::channel(32);
        tokio::spawn(run_counter(receiver));
        CounterHandle { commands }
    }

    pub async fn increment(&self, key: &str) -> u64 {
        let (reply, response) = oneshot::channel();
        let command = Command::Increment {
            key: key.to_string(),
            reply,
        };
        self.commands.send(command).await.expect("actor is gone");
        response.await.expect("actor dropped the reply")
    }

    pub async fn get(&self, key: &str) -> Option<u64> {
        let (reply, response) = oneshot::channel();
        let command = Command::Get {
            key: key.to_string(),
            reply,
        };
        self.commands.send(command).await.expect("actor is gone");
        response.await.expect("actor dropped the reply")
    }
}
```

**Что происходит:**

1. `Command` — это `enum` с вариантом на каждую операцию. Каждый вариант, которому нужен ответ, несёт для него `oneshot::Sender`.
2. `run_counter` — это актор: цикл по входящим командам. Только этот код видит `counts`.
3. `CounterHandle` прячет каналы. `increment` создаёт `oneshot`, отправляет команду, затем ждёт ответ. Для вызывающего это выглядит как обычный асинхронный метод.
4. `#[derive(Clone)]` клонирует только `mpsc::Sender`, поэтому у каждой задачи может быть свой дескриптор. Когда выброшен последний дескриптор, `recv()` возвращает `None`, и актор сам заканчивается.
5. `let _ = reply.send(…)` игнорирует ошибку, которая возникает, когда вызывающий перестал ждать (например, после тайм-аута). Актор не должен падать из-за того, что один вызывающий ушёл.

Используйте его из `src/bin/actor.rs`:

```rust
use p14_tokio::CounterHandle;

#[tokio::main]
async fn main() {
    let counter = CounterHandle::spawn();
    let mut handles = Vec::new();
    for worker in 0..3 {
        let counter = counter.clone();
        handles.push(tokio::spawn(async move {
            for _ in 0..100 {
                counter.increment("visits").await;
            }
            println!("worker {worker} finished");
        }));
    }
    for handle in handles {
        handle.await.unwrap();
    }
    println!("visits = {:?}", counter.get("visits").await);
}
```

```text
worker 2 finished
worker 0 finished
worker 1 finished
visits = Some(300)
```

Порядок строк «finished» меняется от запуска к запуску; итог всегда 300. Три задачи на нескольких потоках, никакой блокировки, никаких потерянных обновлений: канал выстраивает команды в очередь, а актор обрабатывает их по одной.

> 🧪 **Тестирование:** актор легко тестировать: создайте дескриптор в `#[tokio::test]`, вызовите его методы и проверьте ответы. Хороший тест конкурентности запускает много задач с клонами дескриптора и проверяет, что итоговое число точное. Упражнение 2 пишет тесты обоих видов.

Когда что выбирать: `Mutex` проще для маленьких быстрых изменений общих данных. Актор лучше, когда состояние живёт долго, операций много или они медленные, или состояние включает что-то вроде соединения, у которого должен быть один владелец.

## Частые ошибки

**1. Мысль, что выбрасывание `JoinHandle` останавливает задачу.** Нет: задача отсоединяется и продолжает работать. Чтобы её остановить, вызовите `abort()` или держите задачи в `JoinSet`, который прерывает их при выбрасывании.

**2. Блокировка внутри задачи.** `std::thread::sleep`, долгий цикл или вызов блокирующей библиотеки занимают рабочий поток (раздел 3). В однопоточном рантайме и в `#[tokio::test]` это останавливает всё. Используйте `spawn_blocking`.

**3. `watch` там, где важно каждое значение.** Получатель `watch` видит последнее значение и может пропустить промежуточные. Для «каждого события» используйте `mpsc` (один потребитель) или `broadcast` (много потребителей).

**4. Игнорирование `Lagged` из `broadcast`.** `recv()` может вернуть `Err(Lagged(n))`. Если ваш цикл считает каждую `Err` «закрытием» и выходит, короткий всплеск сообщений завершит подписчика. Сопоставляйте `broadcast::error::RecvError::Lagged(missed)` и `RecvError::Closed` по отдельности.

**5. `unwrap()` для ответа, которого может уже не быть.** В акторе `reply.send(x).unwrap()` уронит актор, когда один вызывающий перестанет ждать. Игнорируйте эту ошибку (`let _ =`): это не проблема актора.

## Попробуйте сами

1. Создайте `code/p14-tokio` и запустите `flavors`, `join_errors` и `join_set`.
2. В `flavors.rs` замените `worker_threads(2)` на `worker_threads(1)` и предскажите идентификаторы потоков.
3. В `blocking.rs` замените атрибут main на `#[tokio::main(worker_threads = 2)]` и запустите снова. Задача `ticker` в первой части теперь печатает вовремя, потому что его выполняет второй рабочий поток. Но блокировка всё равно стоит вам целого рабочего потока.
4. Добавьте актор в `src/lib.rs`, запустите `actor` и добавьте тесты из упражнения 2.
5. Запустите `cargo clippy --all-targets` и `cargo fmt`, затем сделайте коммит: `git add code/p14-tokio && git commit -m "p14-tokio: tasks, channels and an actor"`.

## Упражнения

### Упражнение 1 — Квадраты через `JoinSet` (легко)

Запустите в `JoinSet` по одной задаче на каждое число от 1 до 5; каждая задача возвращает квадрат своего числа. Соберите результаты через `join_next` и напечатайте их сумму.

<details>
<summary>Подсказка</summary>

`set.spawn(async move { n * n })` требует `move`, чтобы каждая задача владела своим `n`. Результаты приходят в порядке завершения, но сложению порядок неважен.

</details>

<details>
<summary>Ответ</summary>

```rust
use tokio::task::JoinSet;

#[tokio::main]
async fn main() {
    let mut set = JoinSet::new();
    for n in 1..=5_u64 {
        set.spawn(async move { n * n });
    }
    let mut sum = 0;
    while let Some(result) = set.join_next().await {
        sum += result.unwrap();
    }
    println!("sum of squares = {sum}");
}
```

```text
sum of squares = 55
```

Почему это работает: `join_next` возвращает `Result` каждой задачи по мере её завершения и `None`, когда задач не осталось, поэтому цикл заканчивается сам. `unwrap` превращает здесь `JoinError` (панику или отмену) в панику, что для маленькой программы нормально.

</details>

### Упражнение 2 — Научите актор команде `Reset` (средне)

Добавьте команду `Reset { key }`, которая удаляет ключ из таблицы, и метод `reset(&self, key)`. Ответ ей не нужен. Напишите два `#[tokio::test]`: десять задач, каждая из которых 100 раз вызывает `increment("hits")`, дают `Some(1000)`; после `reset("a")` вызов `get("a")` возвращает `None`, а следующий `increment("a")` возвращает 1.

<details>
<summary>Подсказка</summary>

У команды без ответа нет `oneshot`. Почему тест может вызвать `get` сразу после `reset` и быть уверенным, что сброс уже произошёл? Подумайте о порядке сообщений в одном канале `mpsc`. Для десяти задач у `JoinSet` есть `join_all().await`, который ждёт их все.

</details>

<details>
<summary>Ответ</summary>

Новый вариант, новая ветка `match` и новый метод:

```rust
    Reset {
        key: String,
    },
```

```rust
            Command::Reset { key } => {
                counts.remove(&key);
            }
```

```rust
    pub async fn reset(&self, key: &str) {
        let command = Command::Reset {
            key: key.to_string(),
        };
        self.commands.send(command).await.expect("actor is gone");
    }
```

Тесты — в конце `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn many_tasks_never_lose_an_update() {
        let counter = CounterHandle::spawn();
        let mut tasks = tokio::task::JoinSet::new();
        for _ in 0..10 {
            let counter = counter.clone();
            tasks.spawn(async move {
                for _ in 0..100 {
                    counter.increment("hits").await;
                }
            });
        }
        tasks.join_all().await;
        assert_eq!(counter.get("hits").await, Some(1000));
    }

    #[tokio::test]
    async fn reset_forgets_a_key() {
        let counter = CounterHandle::spawn();
        counter.increment("a").await;
        counter.reset("a").await;
        assert_eq!(counter.get("a").await, None);
        assert_eq!(counter.increment("a").await, 1);
    }
}
```

`cargo test` показывает `test result: ok. 2 passed; 0 failed` для библиотеки. Почему это работает: один канал `mpsc` доставляет сообщения от одного отправителя в том порядке, в котором они были отправлены. Тест отправляет `Reset`, а затем `Get` через один и тот же дескриптор, поэтому актор обрабатывает их в этом порядке, хотя `reset` не ждёт ответа.

</details>

### Упражнение 3 — Конвейер из трёх стадий (сложно)

Напишите `async fn pipeline(lines: Vec<String>) -> (u64, usize)`: стадия 1 (задача) отправляет строки в ограниченный канал `mpsc` ёмкостью 4; стадия 2 (задача) разбирает каждую строку после `trim` как `u64`, отправляет хорошие числа во второй ограниченный канал и считает плохие строки; стадия 3 (сама функция) суммирует числа. Функция возвращает сумму и число плохих строк. Запустите её со строками от `"1"` до `"100"` плюс `"oops"` и `""`. Напишите два теста: `["1", "x", " 2 "]` даёт `(3, 1)`; и тест, который показывает, что полный канал ёмкостью 1 отвергает `try_send` с `TrySendError::Full`.

<details>
<summary>Подсказка</summary>

Стадия 2 узнаёт число плохих строк только в конце: отправьте его через `oneshot`. Каждая стадия заканчивается, когда её входной канал возвращает `None`, а это происходит, когда предыдущая стадия выбрасывает своего отправителя. Так «конец данных» проходит по конвейеру без какого-либо специального сообщения.

</details>

<details>
<summary>Ответ</summary>

```rust
use tokio::sync::{mpsc, oneshot};

/// Three stages connected by bounded channels:
/// lines -> parse -> sum. Returns (sum, number of bad lines).
async fn pipeline(lines: Vec<String>) -> (u64, usize) {
    let (line_tx, mut line_rx) = mpsc::channel::<String>(4);
    let (number_tx, mut number_rx) = mpsc::channel::<u64>(4);
    let (bad_tx, bad_rx) = oneshot::channel::<usize>();

    // Stage 1: produce lines.
    tokio::spawn(async move {
        for line in lines {
            if line_tx.send(line).await.is_err() {
                break; // the next stage is gone
            }
        }
    });

    // Stage 2: parse; count what cannot be parsed.
    tokio::spawn(async move {
        let mut bad = 0;
        while let Some(line) = line_rx.recv().await {
            match line.trim().parse::<u64>() {
                Ok(number) => {
                    if number_tx.send(number).await.is_err() {
                        break;
                    }
                }
                Err(_) => bad += 1,
            }
        }
        let _ = bad_tx.send(bad);
    });

    // Stage 3: sum, right here.
    let mut sum = 0;
    while let Some(number) = number_rx.recv().await {
        sum += number;
    }
    (sum, bad_rx.await.unwrap())
}

#[tokio::main]
async fn main() {
    let mut lines: Vec<String> = (1..=100).map(|n| n.to_string()).collect();
    lines.push("oops".to_string());
    lines.push("".to_string());
    let (sum, bad) = pipeline(lines).await;
    println!("sum = {sum}, bad lines = {bad}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[tokio::test]
    async fn sums_good_lines_and_counts_bad_ones() {
        let lines = vec!["1".to_string(), "x".to_string(), " 2 ".to_string()];
        assert_eq!(pipeline(lines).await, (3, 1));
    }

    #[tokio::test]
    async fn a_full_channel_rejects_try_send() {
        let (tx, _rx) = mpsc::channel::<u32>(1);
        assert!(tx.try_send(1).is_ok());
        assert!(matches!(
            tx.try_send(2),
            Err(mpsc::error::TrySendError::Full(2))
        ));
    }
}
```

```text
sum = 5050, bad lines = 2
```

Тесты программы из `src/bin/` запускаются командой `cargo test --bin pipeline`. Почему это работает: каждая стадия владеет отправителем следующего канала, поэтому, когда стадия заканчивается, её отправитель выбрасывается, и `recv()` следующей стадии возвращает `None`. При ёмкости 4 стадия 1 никогда не может уйти дальше чем на несколько строк вперёд от стадии 2: каналы дают обратное давление. `try_send` никогда не ждёт, поэтому на полном канале возвращает `Full` со значением внутри — так обратное давление становится видимым в тесте.

</details>

## Вопросы для самопроверки

1. Чем отличаются многопоточный и однопоточный рантаймы?

<details>
<summary>Ответ</summary>

У многопоточного есть пул рабочих потоков, которые делят и крадут задачи, поэтому он использует все ядра. Однопоточный выполняет каждую задачу в потоке, который вызвал `block_on`. `#[tokio::main]` использует многопоточный, `#[tokio::test]` — однопоточный.

</details>

2. Запущенная задача запаниковала. Что происходит с программой и как вы об этом узнаете?

<details>
<summary>Ответ</summary>

Программа продолжает работать: Tokio перехватывает панику (сообщение всё равно печатается). Ожидание `JoinHandle` задачи даёт `Err(JoinError)` с `is_panic() == true`.

</details>

3. Вы выбросили `JoinHandle`. Задача останавливается? А если выбросить `JoinSet`?

<details>
<summary>Ответ</summary>

Выбрасывание `JoinHandle` лишь отсоединяет задачу; она продолжает работать. Выбрасывание `JoinSet` прерывает все задачи в нём.

</details>

4. Когда использовать `spawn_blocking`?

<details>
<summary>Ответ</summary>

Для кода, который блокируется или долго вычисляет без `.await`: блокирующие библиотеки, блокирующие вызовы файлов или баз данных, долгие вычисления. Он выполняется в пуле блокирующих потоков, поэтому рабочие потоки остаются свободными для других задач.

</details>

5. Какой канал вы возьмёте для: очереди заданий; ответа на один запрос; уведомления «конфигурация изменилась»; сообщений чата для всех подключённых пользователей?

<details>
<summary>Ответ</summary>

Ограниченный `mpsc` для очереди заданий, `oneshot` для одного ответа, `watch` для последней конфигурации и `broadcast` для сообщений, которые должен получить каждый пользователь.

</details>

## Итоги

- У Tokio есть многопоточный рантайм (по умолчанию для `#[tokio::main]`) и однопоточный (по умолчанию для `#[tokio::test]`); `Builder` создаёт любой из них.
- Ожидание `JoinHandle` даёт `Result<T, JoinError>`; паники перехватываются и сообщаются, `abort()` отменяет задачу на её следующем `.await`.
- Выбрасывание `JoinHandle` отсоединяет задачу; выбрасывание `JoinSet` прерывает его задачи; `join_next` возвращает результаты в порядке завершения.
- Блокирующий код внутри задачи останавливает другие задачи; выполняйте его через `spawn_blocking`.
- `mpsc` — очередь (ограниченная = обратное давление), `oneshot` — один ответ, `broadcast` — каждое сообщение каждому получателю (медленные получают `Lagged`), `watch` — последнее значение.
- Актор владеет своим состоянием и обрабатывает команды из канала с ответами через `oneshot`; ему не нужны блокировки, и его легко тестировать.

## Источники и что почитать

- [Tokio tutorial — Spawning](https://tokio.rs/tokio/tutorial/spawning) — задачи, ограничения `'static` и `Send`.
- [Tokio tutorial — Channels](https://tokio.rs/tokio/tutorial/channels) — четыре канала и пример «запрос — ответ» с `oneshot`.
- [tokio::task module](https://docs.rs/tokio/1.54.1/tokio/task/index.html) — `spawn`, `JoinHandle`, `JoinSet`, `spawn_blocking`, `yield_now`, кооперативное планирование.
- [tokio::sync module](https://docs.rs/tokio/1.54.1/tokio/sync/index.html) — все каналы и типы синхронизации, с подсказками по выбору.
- [tokio::runtime::Builder](https://docs.rs/tokio/1.54.1/tokio/runtime/struct.Builder.html) — все параметры рантайма.
- [Actors with Tokio](https://ryhl.io/blog/actors-with-tokio/) — статья Alice Ryhl, мейнтейнера Tokio; паттерн «актор» подробнее.

---

[🏠 Начало](../START-HERE.md) · [← 14.12 Pin и самоссылающиеся future](12-pin-and-self-referential-futures.md) · [📋 Этап 14](README.md) · [14.14 Отмена, тайм-ауты и корректное завершение →](14-cancellation-timeouts-and-graceful-shutdown.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
