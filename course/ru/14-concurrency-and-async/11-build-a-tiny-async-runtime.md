<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 14.10 Как работают future: Future, poll и Waker](10-how-futures-work.md) · [📋 Этап 14](README.md) · [14.12 Pin и самоссылающиеся future →](12-pin-and-self-referential-futures.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/14-concurrency-and-async/11-build-a-tiny-async-runtime.md) <!-- nav -->

# Урок 14.11 — Пишем крошечный асинхронный рантайм

> **Этап 14 — Конкурентность и асинхронность подробно** · Урок 11 из 16 · ⏱ около 135 минут

## Что вы узнаете

- Из каких частей состоит асинхронный рантайм: задачи, очередь готовых задач, waker, таймеры и цикл, который их связывает.
- Как `Waker`, построенный через `std::task::Wake`, возвращает свою задачу в очередь.
- Как написать future `sleep`, которому не нужен лишний поток, потому что исполнитель хранит все таймеры в одном списке.
- Как исполнитель спит до следующего таймера или следующего пробуждения и никогда не крутится впустую.
- Что Tokio добавляет к этой схеме и зачем.

## Перед началом

- Вы прошли [Урок 14.10 — Как работают future: Future, poll и Waker](10-how-futures-work.md). Этот урок использует его идеи: контракт waker, `Wake`, `park` и `unpark`.
- Вы знаете `Box<dyn Trait>` из [Урока 06.1](../06-rust-advanced/01-trait-objects-and-dynamic-dispatch.md), `RefCell` из [Урока 06.4](../06-rust-advanced/04-interior-mutability.md) и очередь (`VecDeque`) с Этапа 07.
- Вы знаете, что делает epoll, из [Урока 13.13 — Неблокирующий ввод-вывод и epoll](../13-operating-systems/13-non-blocking-io-and-epoll.md). Здесь мы его только упоминаем.
- Код этого урока находится в `~/projects/rust/code/p14-tiny-runtime`: библиотека с маленькими программами в `src/bin/`, как в Уроке 14.10.

## Новые слова

| Слово | Значение |
|---|---|
| **задача** (task) | Future, которым владеет рантайм и выполняет его самостоятельно; запускается через `spawn`. У каждой задачи свой waker. |
| **очередь готовых задач** (run queue) | Список задач, которые разбудили и которые ждут опроса. Исполнитель берёт их из начала. |
| **исполнитель** (executor) | Часть рантайма, которая берёт задачи из очереди и опрашивает их. |
| **таймер** (timer) | Запись «разбуди этот waker в это время». Её создаёт future `sleep`. |
| **срок** (deadline) | Момент (`Instant`), когда таймер должен сработать. |
| **реактор** (reactor) | Часть рантайма, которая ждёт событий I/O (input/output, ввод-вывод) от операционной системы (epoll в Linux, kqueue в macOS) и вызывает waker задач, которые их ждут. В нашем крошечном рантайме есть таймеры, но нет реактора. |
| **локальная для потока переменная** (thread-local) | Переменная, у которой в каждом потоке своя копия; создаётся макросом `thread_local!`. |
| **кооперативное планирование** (cooperative scheduling) | Задачи сами отдают управление — в точках `.await`. Рантайм не может остановить задачу, которая никогда не делает `.await`. |
| **голодание** (starvation) | Задача никак не получает возможности выполниться, потому что другие задачи всё время занимают поток. |

## 1. План

Рантайм — это цикл вокруг очереди. Вот всё, что будет в нашем рантайме, и как задача проходит через него:

```text
            spawn(future)
                  |
                  v
   +-------------------------------+   pop_front   +-------------------+
   | run queue: VecDeque<Arc<Task>>| ------------> | poll the task     |
   +-------------------------------+               +-------------------+
          ^             ^                             |            |
          |             |                        Ready |            | Pending
   waker.wake()    waker.wake()                        v            v
          |             |                        drop the task   the future keeps
   +-------------+  +-----------------+                           its waker somewhere:
   | timers list |  | another thread  |                           a timer, a channel,
   | (deadline,  |  | (like Delay in  |                           another thread
   |  waker)     |  |  Lesson 14.10)  |
   +-------------+  +-----------------+

   When the queue is empty: park the thread until the nearest deadline or until a wake.
```

Правила:

1. `spawn` оборачивает future в `Task` и кладёт его в очередь.
2. Исполнитель достаёт задачу и опрашивает её один раз.
3. Если задача `Ready`, она закончена и выбрасывается. Если `Pending` — она уже отдала кому-то свой waker.
4. Когда этот кто-то вызывает `wake`, задача снова попадает в очередь, а поток исполнителя получает `unpark`.
5. Когда опрашивать нечего, исполнитель запускает наступившие таймеры, а затем спит до ближайшего срока (или до пробуждения из другого потока).

Создайте библиотеку:

```bash
cd ~/projects/rust
cargo new --lib code/p14-tiny-runtime
cd code/p14-tiny-runtime
mkdir src/bin
```

Зависимостей нет: всё берётся из стандартной библиотеки. Весь код разделов 2–4 идёт в `src/lib.rs`, часть за частью.

## 2. Задачи и их waker

Первая часть `src/lib.rs`:

```rust
//! A tiny single-threaded async runtime: a task queue, wakers and timers.

use std::cell::RefCell;
use std::collections::VecDeque;
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Wake, Waker};
use std::thread::{self, Thread};
use std::time::{Duration, Instant};

/// A future of any type, on the heap, that can move between threads.
type BoxFuture = Pin<Box<dyn Future<Output = ()> + Send>>;

/// Everything that the executor, the tasks and the wakers share.
struct Shared {
    /// Tasks that were woken and wait to be polled.
    queue: Mutex<VecDeque<Arc<Task>>>,
    /// Sleeping tasks: when to wake them, and how.
    timers: Mutex<Vec<(Instant, Waker)>>,
    /// The thread that runs the executor, so a waker can unpark it.
    thread: Thread,
}

/// One spawned future and the way back into the queue.
struct Task {
    future: Mutex<Option<BoxFuture>>,
    shared: Arc<Shared>,
}

impl Wake for Task {
    fn wake(self: Arc<Self>) {
        let shared = Arc::clone(&self.shared);
        shared.queue.lock().unwrap().push_back(self);
        shared.thread.unpark();
    }
}

thread_local! {
    /// The runtime of this thread, while `block_on` runs.
    static CURRENT: RefCell<Option<Arc<Shared>>> = const { RefCell::new(None) };
}

fn current() -> Arc<Shared> {
    CURRENT
        .with(|current| current.borrow().clone())
        .expect("must be called inside block_on")
}

/// Starts a new task. It runs when the executor gets to it.
pub fn spawn<F>(future: F)
where
    F: Future<Output = ()> + Send + 'static,
{
    let shared = current();
    let task = Arc::new(Task {
        future: Mutex::new(Some(Box::pin(future))),
        shared: Arc::clone(&shared),
    });
    shared.queue.lock().unwrap().push_back(task);
}
```

**Что происходит:**

1. `BoxFuture` — один тип для всех задач. У каждого блока `async` свой безымянный тип, поэтому очередь хранит их как `Box<dyn Future>`. `Pin<Box<…>>` (из `Box::pin`) даёт закреплённый future, который нужен `poll`; закрепление объясняет Урок 14.12. `Send` нужен, потому что `Waker` могут отправить в другой поток, а waker держит задачу.
2. `Shared` — сердце рантайма. Все его поля за `Mutex`, потому что waker можно вызвать из других потоков.
3. `Task` хранит свой future в `Mutex<Option<…>>`. `Option` — потому что на время опроса future вынимают, а после завершения его нет.
4. `impl Wake for Task` — это и есть waker: «положи меня снова в очередь и сделай `unpark` потоку исполнителя». Это тот же приём, что и `ThreadWaker` в Уроке 14.10, плюс очередь.
5. `CURRENT` — локальная для потока переменная: `spawn` и `sleep` должны находить рантайм, не получая его аргументом. `block_on` её устанавливает, а `current()` читает. Tokio использует ту же идею, поэтому `tokio::spawn` паникует вне рантайма.
6. `spawn` создаёт задачу и кладёт её в очередь. Он её не опрашивает: задача выполнится, когда до неё дойдёт исполнитель.

## 3. Два future для ожидания: `sleep` и `yield_now`

Вторая часть `src/lib.rs`:

```rust
/// A future that is ready once `deadline` has passed.
pub struct Sleep {
    deadline: Instant,
}

/// Waits for `duration` without blocking the thread.
pub fn sleep(duration: Duration) -> Sleep {
    Sleep {
        deadline: Instant::now() + duration,
    }
}

impl Future for Sleep {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if Instant::now() >= self.deadline {
            return Poll::Ready(());
        }
        let shared = current();
        shared
            .timers
            .lock()
            .unwrap()
            .push((self.deadline, cx.waker().clone()));
        Poll::Pending
    }
}

/// Lets other tasks run once, then continues.
pub fn yield_now() -> YieldNow {
    YieldNow { yielded: false }
}

pub struct YieldNow {
    yielded: bool,
}

impl Future for YieldNow {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if self.yielded {
            Poll::Ready(())
        } else {
            self.yielded = true;
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}
```

**Что происходит:**

1. `sleep` только вычисляет срок. Помните: future ленивы, пока ничего не зарегистрировано.
2. Когда `Sleep` опрашивают до срока, он добавляет `(deadline, waker)` в список таймеров рантайма и возвращает `Pending`. Контракт из Урока 14.10 соблюдён: исполнитель вызовет этот waker, когда придёт время.
3. Сравните с `Delay` из Урока 14.10: никакого вспомогательного потока. Тысяча спящих задач стоит тысячу маленьких записей в `Vec`, а не тысячу потоков.
4. Если `Sleep` опросят дважды до срока, он добавится дважды. Второе пробуждение вызовет лишь один лишний, безвредный опрос. (Tokio хранит каждый таймер только один раз; для этого нужно больше кода.)
5. `yield_now` — это `YieldOnce` из Урока 14.10: он будит себя и один раз возвращает `Pending`. Его задача уходит в **конец** очереди, поэтому сначала выполняются все остальные готовые задачи.

## 4. Цикл исполнителя

Последняя часть `src/lib.rs` — цикл, который приводит всё в движение:

```rust
/// Polls one task once. A finished task is dropped.
fn run_task(task: Arc<Task>) {
    let mut slot = task.future.lock().unwrap();
    if let Some(mut future) = slot.take() {
        let waker = Waker::from(Arc::clone(&task));
        let mut cx = Context::from_waker(&waker);
        if future.as_mut().poll(&mut cx).is_pending() {
            *slot = Some(future);
        }
    }
}

/// Wakes every timer whose deadline has passed.
/// Returns the nearest deadline that is still in the future.
fn fire_timers(shared: &Shared) -> Option<Instant> {
    let now = Instant::now();
    let mut due = Vec::new();
    let next = {
        let mut timers = shared.timers.lock().unwrap();
        timers.retain(|(deadline, waker)| {
            if *deadline <= now {
                due.push(waker.clone());
                false
            } else {
                true
            }
        });
        timers.iter().map(|(deadline, _)| *deadline).min()
    };
    for waker in due {
        waker.wake();
    }
    next
}

/// Runs `future` and every task it spawns, until `future` is finished.
pub fn block_on<F>(future: F) -> F::Output
where
    F: Future + Send + 'static,
    F::Output: Send + 'static,
{
    let shared = Arc::new(Shared {
        queue: Mutex::new(VecDeque::new()),
        timers: Mutex::new(Vec::new()),
        thread: thread::current(),
    });
    CURRENT.with(|current| *current.borrow_mut() = Some(Arc::clone(&shared)));

    // The main future is a task too; it puts its result into `result`.
    let result = Arc::new(Mutex::new(None));
    let slot = Arc::clone(&result);
    spawn(async move {
        let value = future.await;
        *slot.lock().unwrap() = Some(value);
    });

    loop {
        // 1. Poll every task that is ready to run.
        loop {
            let next_task = shared.queue.lock().unwrap().pop_front();
            match next_task {
                Some(task) => run_task(task),
                None => break,
            }
        }
        // 2. Stop when the main future is finished.
        if let Some(value) = result.lock().unwrap().take() {
            shared.queue.lock().unwrap().clear();
            shared.timers.lock().unwrap().clear();
            CURRENT.with(|current| *current.borrow_mut() = None);
            return value;
        }
        // 3. Wake the timers that are due.
        let next_deadline = fire_timers(&shared);
        // 4. Nothing to do? Sleep until a waker or the next timer.
        if shared.queue.lock().unwrap().is_empty() {
            match next_deadline {
                Some(deadline) => {
                    thread::park_timeout(deadline.saturating_duration_since(Instant::now()))
                }
                None => thread::park(),
            }
        }
    }
}
```

**Что происходит:**

1. `run_task` вынимает future из задачи, делает waker из самой задачи (`Waker::from(Arc<Task>)` работает, потому что `Task` реализует `Wake`) и опрашивает один раз. Future в состоянии `Pending` кладётся обратно; `Ready` просто не кладётся обратно и поэтому выбрасывается.
2. `fire_timers` убирает из списка все наступившие таймеры и будит их, что кладёт их задачи в очередь. Функция возвращает ближайший оставшийся срок. Waker вызываются *после* освобождения блокировки `timers`, поэтому разбуженная задача никогда не будет ждать блокировку, которую мы ещё держим.
3. `block_on` создаёт `Shared`, сохраняет его в `CURRENT` и запускает главный future как обычную задачу. Когда задача заканчивается, она записывает результат в `result`.
4. Шаг 1 цикла: внутренний `loop` достаёт по одной задаче. Строка `let next_task = …pop_front();` заканчивается на `;`, поэтому охранный объект (guard) блокировки очереди уничтожается **до** опроса задачи. Это важно: задача, которая будит себя или запускает новую задачу, снова блокирует очередь (в разделе «Частые ошибки» показано, что иначе произойдёт).
5. Шаг 2: когда главный future закончился, рантайм очищает очередь и таймеры и возвращает значение. Как и в Tokio, задачи, которые ещё работают, выбрасываются, то есть отменяются. Очистка заодно разрывает цикл ссылок «`Shared` держит задачи, задачи держат `Shared`», и память освобождается.
6. Шаги 3 и 4: запустить наступившие таймеры. Если очередь всё ещё пуста, припарковать поток: через `park_timeout` до ближайшего срока или через `park` до тех пор, пока какой-нибудь waker не вызовет `unpark` (другой поток, как в упражнении 2). Исполнитель никогда не крутится впустую.

## 5. Запуск

**Две задачи по очереди.** Создайте `src/bin/two_tasks.rs`:

```rust
use p14_tiny_runtime::{block_on, spawn, yield_now};

async fn worker(name: &'static str) {
    for step in 1..=3 {
        println!("{name}: step {step}");
        yield_now().await;
    }
}

fn main() {
    block_on(async {
        spawn(worker("A"));
        spawn(worker("B"));
        yield_now().await;
        println!("main: spawned two tasks");
        for _ in 0..3 {
            yield_now().await;
        }
        println!("main: done");
    });
}
```

`cargo run --bin two_tasks`:

```text
A: step 1
B: step 1
main: spawned two tasks
A: step 2
B: step 2
A: step 3
B: step 3
main: done
```

**Что происходит:**

1. Главная задача запускает A и B (очередь: A, B) и уступает, поэтому уходит в конец (очередь: A, B, main).
2. A печатает шаг 1 и уступает (в конец), B делает то же, затем main печатает свою строку.
3. Дальше они продолжают по очереди. Один поток, три задачи, и каждый `.await` на `yield_now` — место, где может поработать другая задача. Это и есть **кооперативное планирование**.

**Таймеры.** Создайте `src/bin/timers.rs`:

```rust
use p14_tiny_runtime::{block_on, sleep, spawn};
use std::time::{Duration, Instant};

fn main() {
    let start = Instant::now();
    block_on(async move {
        for ms in [300, 100, 200] {
            spawn(async move {
                sleep(Duration::from_millis(ms)).await;
                println!(
                    "{:>4} ms: task that slept {ms} ms",
                    start.elapsed().as_millis()
                );
            });
        }
        sleep(Duration::from_millis(400)).await;
        println!("{:>4} ms: main is done", start.elapsed().as_millis());
    });
}
```

`cargo run --bin timers`:

```text
 105 ms: task that slept 100 ms
 204 ms: task that slept 200 ms
 304 ms: task that slept 300 ms
 405 ms: main is done
```

Задачи запущены в порядке 300, 100, 200, но заканчиваются в порядке сроков, и вся программа длится 400 мс, а не 300 + 100 + 200 + 400 = 1000 мс. Все четыре ожидания идут одновременно в **одном** потоке, а между сроками поток спит в `park_timeout`.

> 🧪 **Тестирование:** `block_on` позволяет тестировать асинхронный код обычным `#[test]`. Тест вроде `assert_eq!(block_on(async { 6 * 7 }), 42);` проверяет весь путь: запуск главной задачи, опрос, результат. Упражнение 1 добавляет тест с тысячей задач.

## 6. Что добавляет настоящий рантайм

В нашем рантайме около 200 строк и тот же скелет, что у Tokio. Вот что добавляет Tokio и зачем:

| Наш рантайм | Tokio | Зачем |
|---|---|---|
| один поток | пул рабочих потоков, у каждого своя очередь; свободные потоки *крадут* задачи у занятых (work stealing) | использовать все ядра CPU (Central Processing Unit, центральный процессор) |
| `Vec` таймеров, который просматривается на каждом круге | *колесо таймеров* (timer wheel): таймеры разложены по корзинам по времени | миллионы таймеров остаются дешёвыми |
| нет I/O | реактор: epoll (Linux) или kqueue (macOS) ждёт события сокетов и вызывает waker | сетевые серверы |
| задача может оказаться в очереди дважды | флаг состояния у каждой задачи: разбуженная задача ставится в очередь только один раз | меньше бесполезных опросов |
| `spawn` ничего не возвращает | `JoinHandle` с результатом, `abort()` для отмены | результаты и управление (упражнение 3 строит маленький `JoinHandle`) |
| задача, которая никогда не делает `.await`, блокирует всё | бюджет: после множества операций собственные future Tokio намеренно возвращают `Pending` | справедливость между задачами |

Последняя строка важна для любой асинхронной программы, а не только для рантаймов. Кооперативное планирование означает, что задача, которая долго вычисляет **без `.await`**, держит поток. В нашем рантайме все остальные задачи ждут, пока она не закончит; у рантайма нет способа её остановить. Отсюда правило из Урока 06.9: никогда не блокируйте рантайм; переносите долгие вычисления в `spawn_blocking` или пул потоков.

> ⚡ **Производительность:** схема «одна очередь и сон до следующего события» — причина, по которой ожидание в async дёшево: ждущая задача стоит лишь своего future в памяти и записи в списке, а поток, которому нечего делать, вообще не тратит CPU.

## Частые ошибки

**1. Блокировка очереди удерживается во время опроса задачи.** Внутренний цикл так и хочется написать вот так:

```rust
        while let Some(task) = shared.queue.lock().unwrap().pop_front() {
            run_task(task);
        }
```

Программа ничего не печатает и зависает навсегда. Временный guard блокировки в условии `while let` живёт до конца тела цикла. Когда главная задача вызывает `spawn`, `spawn` снова блокирует очередь в том же потоке; `std::sync::Mutex` нельзя заблокировать дважды из одного потока, поэтому он ждёт сам себя. Исправление: достаньте задачу отдельной инструкцией, которая заканчивается на `;`, как в разделе 4.

**2. Забытый `unpark` в `Task::wake`.** Пробуждения из собственного потока исполнителя по-прежнему работают, потому что исполнитель проверяет очередь, прежде чем парковаться. Но пробуждение из другого потока (упражнение 2) кладёт задачу в очередь, пока исполнитель спит в `park()`, и никто его не будит: программа зависает.

**3. Вызов `spawn` или `sleep` вне `block_on`.** В `CURRENT` нет рантайма:

```text
thread 'main' (5574875) panicked at src/lib.rs:47:10:
must be called inside block_on
```

У Tokio то же правило и похожее сообщение паники.

**4. Вызов waker при удерживаемой блокировке таймеров.** `waker.wake()` может выполнить код, который блокирует что-то ещё, а в многопоточном рантайме — и саму блокировку таймеров. Сначала соберите наступившие waker, освободите блокировку, потом будите их — как делает `fire_timers`.

## Попробуйте сами

1. Напишите три части `src/lib.rs` и две программы и запустите их.
2. В `two_tasks.rs` уберите строки `yield_now().await;` внутри `worker`. Предскажите порядок строк, затем запустите. (Теперь каждая задача `worker` печатает все три шага за один опрос: A 1–3, затем B 1–3, затем обе строки `main`.)
3. Добавьте `println!("tick")` в начало внешнего `loop` в `block_on`, запустите `timers` и посчитайте «тики». Каждый тик — одно пробуждение исполнителя. Затем удалите эту строку.
4. Добавьте тесты из заметки 🧪 и из упражнения 1, запустите `cargo test`, `cargo clippy --all-targets` и `cargo fmt`.
5. Сделайте коммит: `git add code/p14-tiny-runtime && git commit -m "p14-tiny-runtime: executor, wakers and timers"`.

## Упражнения

### Упражнение 1 — Тысяча задач (легко)

Напишите тест: главный future запускает 1000 задач; каждая задача один раз уступает, а затем увеличивает общий `AtomicUsize`. После возврата из `block_on` счётчик должен быть равен 1000. Затем напишите второй тест: `sleep` на 50 мс длится не меньше 50 мс.

<details>
<summary>Подсказка</summary>

Поделите счётчик через `Arc`. Главный future не должен закончиться раньше задач: пусть он дважды сделает `yield_now().await`, чтобы каждая задача получила два хода (один для своего `yield_now`, второй — чтобы закончить).

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::atomic::{AtomicUsize, Ordering};

    #[test]
    fn block_on_returns_the_value() {
        assert_eq!(block_on(async { 6 * 7 }), 42);
    }

    #[test]
    fn a_thousand_tasks_all_run() {
        let count = Arc::new(AtomicUsize::new(0));
        let for_tasks = Arc::clone(&count);
        block_on(async move {
            for _ in 0..1000 {
                let count = Arc::clone(&for_tasks);
                spawn(async move {
                    yield_now().await;
                    count.fetch_add(1, Ordering::SeqCst);
                });
            }
            // Give every task the chance to run twice.
            yield_now().await;
            yield_now().await;
        });
        assert_eq!(count.load(Ordering::SeqCst), 1000);
    }

    #[test]
    fn sleep_waits_at_least_its_duration() {
        let start = Instant::now();
        block_on(sleep(Duration::from_millis(50)));
        assert!(start.elapsed() >= Duration::from_millis(50));
    }
}
```

`cargo test` показывает `test result: ok. 3 passed; 0 failed` для библиотеки. Почему это работает: задачи выполняются в порядке очереди. После первого уступания главная задача стоит за всеми 1000 задачами, после второго — снова за всеми, поэтому к её окончанию каждая задача уже закончила. Каждый тест выполняется в своём потоке, а `CURRENT` локальна для потока, поэтому тесты не делят один рантайм.

</details>

### Упражнение 2 — Пробуждение из другого потока (средне)

Скопируйте `Shared`, `Delay` и его `impl Future` из Урока 14.10 в `src/bin/delay.rs` и запустите `block_on(Delay::new(Duration::from_millis(150)))` *своим* рантаймом. Напечатайте время. Затем закомментируйте `shared.thread.unpark();` в `Task::wake` и запустите снова. Объясните разницу.

<details>
<summary>Подсказка</summary>

Вспомогательный поток `Delay` вызывает `waker.wake()`. Какой код тогда выполняется и в каком потоке? Где в этот момент поток исполнителя?

</details>

<details>
<summary>Ответ</summary>

`main` в `src/bin/delay.rs` (скопированный код `Delay` идёт над ним):

```rust
fn main() {
    let start = Instant::now();
    block_on(Delay::new(Duration::from_millis(150)));
    println!(
        "woken from another thread after {} ms",
        start.elapsed().as_millis()
    );
}
```

```text
woken from another thread after 150 ms
```

Без `unpark()` программа ничего не печатает и не заканчивается (остановите её через Ctrl+C). Почему: вспомогательный поток выполняет `Task::wake`, который кладёт задачу в очередь. Но поток исполнителя находится в `thread::park()`, потому что очередь была пуста, а таймеров не было. Закончить этот сон может только `unpark`. С `unpark` поток просыпается, находит задачу, опрашивает её, и `block_on` возвращается.

</details>

### Упражнение 3 — `JoinHandle` (сложно)

Напишите `spawn_with_handle(future) -> JoinHandle<T>` поверх своего `spawn` (в файле программы, не меняя библиотеку). `JoinHandle<T>` — это future, который готов с результатом задачи. Проверьте его: запустите `slow_square(2)` и `slow_square(3)`, где `slow_square(n)` спит `100 * n` мс и возвращает `n * n`, и напечатайте сумму двух результатов и время. Должно получиться 13 примерно через 300 мс.

<details>
<summary>Подсказка</summary>

Задача и дескриптор делят `Arc<Mutex<JoinState<T>>>` с полями `result: Option<T>` и `waker: Option<Waker>`. Задача сохраняет результат и будит сохранённый waker. `poll` дескриптора забирает результат, если он есть; иначе сохраняет свой waker и возвращает `Pending` — тот же шаблон, что у `Delay`.

</details>

<details>
<summary>Ответ</summary>

```rust
use p14_tiny_runtime::{block_on, sleep, spawn};
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Waker};
use std::time::{Duration, Instant};

struct JoinState<T> {
    result: Option<T>,
    waker: Option<Waker>,
}

/// A future that gives the output of a spawned task.
struct JoinHandle<T> {
    state: Arc<Mutex<JoinState<T>>>,
}

fn spawn_with_handle<F>(future: F) -> JoinHandle<F::Output>
where
    F: Future + Send + 'static,
    F::Output: Send + 'static,
{
    let state = Arc::new(Mutex::new(JoinState {
        result: None,
        waker: None,
    }));
    let for_task = Arc::clone(&state);
    spawn(async move {
        let value = future.await;
        let mut state = for_task.lock().unwrap();
        state.result = Some(value);
        if let Some(waker) = state.waker.take() {
            waker.wake();
        }
    });
    JoinHandle { state }
}

impl<T> Future for JoinHandle<T> {
    type Output = T;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<T> {
        let mut state = self.state.lock().unwrap();
        match state.result.take() {
            Some(value) => Poll::Ready(value),
            None => {
                state.waker = Some(cx.waker().clone());
                Poll::Pending
            }
        }
    }
}

async fn slow_square(n: u64) -> u64 {
    sleep(Duration::from_millis(100 * n)).await;
    n * n
}

fn main() {
    let start = Instant::now();
    let total = block_on(async {
        let a = spawn_with_handle(slow_square(2));
        let b = spawn_with_handle(slow_square(3));
        a.await + b.await
    });
    println!("total = {total} after {} ms", start.elapsed().as_millis());
}
```

```text
total = 13 after 305 ms
```

Почему это работает: обе задачи запущены до того, как main начинает чего-либо ждать, поэтому их сны (200 мс и 300 мс) перекрываются. `a.await` ждёт около 200 мс, а `b.await` — ещё около 100 мс. Проверка «есть ли результат?» и «сохранить мой waker» происходят под одной блокировкой, поэтому пробуждение не может потеряться (Урок 14.10).

</details>

## Вопросы для самопроверки

1. Что лежит в очереди готовых задач и кто туда это кладёт?

<details>
<summary>Ответ</summary>

Задачи (`Arc<Task>`), готовые к опросу. `spawn` кладёт туда новые задачи, а каждый `wake()` waker задачи кладёт туда эту задачу снова — из любого потока.

</details>

2. Почему нашему `sleep` не нужен вспомогательный поток, в отличие от `Delay` из Урока 14.10?

<details>
<summary>Ответ</summary>

Он регистрирует `(deadline, waker)` в списке таймеров рантайма. Исполнитель сам проверяет список, спит через `park_timeout` до ближайшего срока, а затем будит наступившие таймеры.

</details>

3. Что делает исполнитель, когда очередь готовых задач пуста?

<details>
<summary>Ответ</summary>

Запускает наступившие таймеры. Если очередь всё ещё пуста, паркует поток: до ближайшего срока (`park_timeout`) или, если таймеров нет, до тех пор, пока какой-нибудь waker не вызовет `unpark`.

</details>

4. Почему guard блокировки очереди уничтожается до опроса задачи?

<details>
<summary>Ответ</summary>

Опрос может вызвать `spawn` или `wake`, которые снова блокируют очередь. `std::sync::Mutex`, заблокированный дважды в одном потоке, вечно ждёт сам себя, и программа зависла бы.

</details>

5. Задача выполняет десятисекундный цикл вычислений без единого `.await`. Что происходит с остальными задачами в нашем рантайме и в Tokio?

<details>
<summary>Ответ</summary>

В нашем рантайме они все ждут десять секунд: планирование кооперативное, и только точки `.await` отдают поток. В Tokio этот рабочий поток тоже потерян на десять секунд: другие рабочие потоки продолжают выполнять другие задачи, но при малом числе потоков или в однопоточном рантайме всё замедляется или останавливается. Переносите такую работу в `spawn_blocking` или пул потоков.

</details>

## Итоги

- Рантайм — это цикл вокруг очереди готовых задач: достать задачу, опросить один раз, оставить, если `Pending`, выбросить, если `Ready`.
- Waker задачи (`impl Wake for Task`) кладёт задачу обратно в очередь и делает `unpark` потоку исполнителя, поэтому пробуждения из любого потока работают.
- Локальная для потока `CURRENT` позволяет `spawn` и `sleep` находить рантайм; вне `block_on` они паникуют.
- Таймеры — это записи `(deadline, waker)` в одном списке; исполнитель запускает наступившие и паркуется до ближайшего срока, а не крутится впустую.
- Никогда не держите блокировку во время опроса задачи или вызова waker.
- Tokio добавляет рабочие потоки с кражей задач, колесо таймеров, реактор I/O, `JoinHandle` и бюджет для справедливости; планирование всё равно кооперативное, поэтому никогда не блокируйте поток внутри задачи.

## Источники и что почитать

- [Tokio tutorial — Async in depth](https://tokio.rs/tokio/tutorial/async) — строит «mini Tokio» из тех же частей и объясняет, как работают waker в Tokio.
- [Making the Tokio scheduler 10x faster](https://tokio.rs/blog/2019-10-scheduler) — статья в блоге Tokio об очередях, краже задач и флаге состояния задачи.
- [std::task::Wake](https://doc.rust-lang.org/std/task/trait.Wake.html) — трейт, который мы реализовали для `Task`.
- [std::thread::park_timeout](https://doc.rust-lang.org/std/thread/fn.park_timeout.html) — парковка с ограничением времени и правила о ложных пробуждениях.
- [std::thread_local](https://doc.rust-lang.org/std/macro.thread_local.html) — макрос, на котором построена `CURRENT`.
- [mio на docs.rs](https://docs.rs/mio/latest/mio/) — библиотека I/O под реактором Tokio, если вы захотите добавить I/O в свой рантайм.

---

[🏠 Начало](../START-HERE.md) · [← 14.10 Как работают future: Future, poll и Waker](10-how-futures-work.md) · [📋 Этап 14](README.md) · [14.12 Pin и самоссылающиеся future →](12-pin-and-self-referential-futures.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
