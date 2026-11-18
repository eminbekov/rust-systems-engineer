<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 14.9 Тестирование конкурентного кода с loom](09-testing-concurrent-code-with-loom.md) · [📋 Этап 14](README.md) · [14.11 Пишем крошечный асинхронный рантайм →](11-build-a-tiny-async-runtime.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/14-concurrency-and-async/10-how-futures-work.md) <!-- nav -->

# Урок 14.10 — Как работают future: Future, poll и Waker

> **Этап 14 — Конкурентность и асинхронность подробно** · Урок 10 из 16 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Почему future ничего не делает, пока его кто-нибудь не опросит, и во что компилятор превращает `async fn`.
- Контракт между future и исполнителем: «возвращай `Pending`, только если ты уже договорился о пробуждении».
- Как вручную написать *листовой future* (leaf future), который завершает вспомогательный поток, и почему он работает под любым рантаймом.
- Как построить `Waker` с помощью трейта `std::task::Wake` и написать `block_on`, который спит, а не крутится в цикле.
- Две классические ошибки: future, который никогда не просыпается, и потерянное пробуждение.

## Перед началом

- Вы прошли [Урок 14.9 — Тестирование конкурентного кода с loom](09-testing-concurrent-code-with-loom.md).
- Вы знаете трейт `Future`, `Poll::Ready` / `Poll::Pending`, `pin!` и маленький `block_on` с `Waker::noop()` из [Урока 06.9 — Введение в async/await](../06-rust-advanced/09-async-await-introduction.md). Этот урок начинается там, где закончился тот.
- Вы знаете `Arc<Mutex<T>>` и потоки из [Урока 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md), а `enum` с `match` — с Этапа 04.
- Код этого урока находится в `~/projects/rust/code/p14-futures`: это библиотека с несколькими маленькими программами в `src/bin/` (каждый файл там — одна программа; запускайте её командой `cargo run --bin <имя файла>`).

## Новые слова

| Слово | Значение |
|---|---|
| **ленивый** (lazy) | Ничего не делает, пока не попросят. Future в Rust не выполняет никакой работы, пока его не опросят. |
| **листовой future** (leaf future) | Future на самом нижнем уровне: он ждёт не другие future, а что-то внешнее (таймер, сокет, другой поток). Рантаймы и библиотеки пишут их вручную. |
| **`Context`** | Второй аргумент `poll`. Сегодня он несёт одно: `Waker` задачи, которая опрашивает future. |
| **`Waker`** | Дескриптор, который говорит «опроси эту задачу снова». Любой, у кого он есть (другой поток, таймер), может вызвать `wake()`. |
| **трейт `Wake`** (`Wake` trait) | Стандартный трейт (`std::task::Wake`): реализуйте `wake` для своего типа внутри `Arc`, и `Waker::from` превратит его в `Waker`. |
| **потерянное пробуждение** (lost wake-up) | Ошибка: событие произошло, но никто не вызвал waker (или waker вызвали до того, как его сохранили), поэтому future больше никогда не опрашивают. |
| **активный опрос** (busy polling) | Вызов `poll` снова и снова в цикле без сна. Это работает, но сжигает целое ядро CPU (Central Processing Unit, центральный процессор). |
| **park / unpark** | `std::thread::park()` усыпляет текущий поток; `thread.unpark()` из другого потока его будит. |
| **ложное пробуждение** (spurious wake-up) | Пробуждение без причины. `park` может вернуться раньше времени, поэтому код вокруг него должен проверить условие ещё раз. |

## 1. Подготовка, и future ленивы

Создайте пакет и добавьте две зависимости:

```bash
cd ~/projects/rust
cargo new --lib code/p14-futures
cd code/p14-futures
cargo add futures@0.3
cargo add tokio@1 --features rt-multi-thread,macros,time
mkdir src/bin
```

В крейте `futures` есть маленький простой исполнитель `futures::executor::block_on`. Мы пользуемся им, пока не напишем свой в разделе 5.

Создайте `src/bin/lazy.rs`:

```rust
async fn hello() -> u32 {
    println!("hello() is running");
    7
}

fn main() {
    let future = hello();
    println!("the future was created, nothing printed yet");
    let value = futures::executor::block_on(future);
    println!("value = {value}");
}
```

Запустите `cargo run --bin lazy`:

```text
the future was created, nothing printed yet
hello() is running
value = 7
```

**Что происходит:**

1. `hello()` не выполняет тело. Вызов только создаёт значение — future — и сохраняет в нём аргументы.
2. Первая напечатанная строка приходит из `main`, значит, тело `hello` ещё не начало работать.
3. `block_on` вызывает `poll` у future. Только теперь тело выполняется, печатает и возвращает `Ready(7)`.

Итак, future — это *описание работы*. Кто-то должен его опросить. Если никто этого не делает, ничего не происходит; компилятор даже предупреждает, когда вы создаёте future и выбрасываете его без `.await` («futures do nothing unless you `.await` or poll them»).

## 2. `async fn` — это конечный автомат

Что именно компилятор строит из `async fn`? Вот маленький листовой future `YieldOnce`, который в первый раз отвечает «ещё нет», а во второй — «готово», и `async fn`, которая ждёт его дважды. Рядом — написанный вручную future, который делает то же самое так, как это делает компилятор. Создайте `src/bin/state_machine.rs`:

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

/// A future that is not ready the first time it is polled.
struct YieldOnce {
    yielded: bool,
}

impl Future for YieldOnce {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if self.yielded {
            Poll::Ready(())
        } else {
            self.yielded = true;
            cx.waker().wake_by_ref(); // "poll me again soon"
            Poll::Pending
        }
    }
}

fn yield_once() -> YieldOnce {
    YieldOnce { yielded: false }
}

// What we write:
async fn two_steps() -> u32 {
    println!("step 1");
    yield_once().await;
    println!("step 2");
    yield_once().await;
    println!("done");
    42
}

// Roughly what the compiler makes of it:
enum TwoSteps {
    Start,
    WaitingFirst(YieldOnce),
    WaitingSecond(YieldOnce),
    Finished,
}

impl Future for TwoSteps {
    type Output = u32;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<u32> {
        loop {
            match &mut *self {
                TwoSteps::Start => {
                    println!("step 1");
                    *self = TwoSteps::WaitingFirst(yield_once());
                }
                TwoSteps::WaitingFirst(inner) => match Pin::new(inner).poll(cx) {
                    Poll::Pending => return Poll::Pending,
                    Poll::Ready(()) => {
                        println!("step 2");
                        *self = TwoSteps::WaitingSecond(yield_once());
                    }
                },
                TwoSteps::WaitingSecond(inner) => match Pin::new(inner).poll(cx) {
                    Poll::Pending => return Poll::Pending,
                    Poll::Ready(()) => {
                        println!("done");
                        *self = TwoSteps::Finished;
                        return Poll::Ready(42);
                    }
                },
                TwoSteps::Finished => panic!("polled after completion"),
            }
        }
    }
}

fn main() {
    println!("async fn:");
    println!("result = {}", futures::executor::block_on(two_steps()));
    println!("hand-written:");
    println!("result = {}", futures::executor::block_on(TwoSteps::Start));
}
```

```text
async fn:
step 1
step 2
done
result = 42
hand-written:
step 1
step 2
done
result = 42
```

**Что происходит:**

1. `YieldOnce` — листовой future. При первом опросе он запоминает, что уже уступил, вызывает `cx.waker().wake_by_ref()` («опроси меня снова») и возвращает `Pending`.
2. `TwoSteps` — это `enum`, у которого по одному варианту на *каждое место, где код может остановиться*: до начала, на первом `.await`, на втором `.await` и после конца. Ожидаемый внутренний future живёт внутри варианта.
3. `poll` выполняет код до следующего `.await`, затем опрашивает внутренний future. Если внутренний отвечает `Pending`, внешний тоже возвращает `Pending`: вся цепочка останавливается и отдаёт поток.
4. При следующем опросе `poll` продолжает с сохранённого состояния, а не с начала. `loop` позволяет перейти к следующему шагу без возврата.
5. `Pin::new(inner)` создаёт закреплённую ссылку на внутренний future. Здесь это разрешено, потому что `YieldOnce` — простой тип, который можно свободно перемещать; когда это не так, объясняет [Урок 14.12](12-pin-and-self-referential-futures.md).

Это главная идея асинхронного Rust: **`.await` не блокирует поток, а возвращает `Pending` вверх по цепочке**. Локальные переменные, которые должны пережить `.await`, хранятся в состоянии, поэтому future помнит, где он остановился.

Настоящий результат компилятора — похожий конечный автомат с пронумерованными состояниями. Как и наша ветка `Finished`, он паникует, если его опросить после завершения. `src/bin/twice.rs` опрашивает блок `async` дважды:

```rust
use std::future::Future;
use std::pin::pin;
use std::task::{Context, Waker};

fn main() {
    let mut future = pin!(async { 1 + 1 });
    let mut cx = Context::from_waker(Waker::noop());
    println!("{:?}", future.as_mut().poll(&mut cx));
    println!("{:?}", future.as_mut().poll(&mut cx));
}
```

```text
Ready(2)

thread 'main' (5437152) panicked at src/bin/twice.rs:6:27:
`async fn` resumed after completion
```

Исполнитель никогда не должен опрашивать завершённый future.

## 3. Контракт: кто опрашивает снова и когда?

В Уроке 06.9 мини-`block_on` опрашивал future в бесконечном цикле с `Waker::noop()`. Это работает, но дорого: пока future ждёт, цикл вызывает `poll` миллионы раз и держит одно ядро CPU загруженным на 100 % (вы измерите это в конце раздела 5). Настоящий рантайм вместо этого соблюдает контракт с каждым future:

> Когда `poll` возвращает `Pending`, future обязан позаботиться о том, чтобы `cx.waker()` (или его клон) был разбужен, когда станет возможен прогресс. Исполнитель не опрашивает задачу снова, пока её не разбудят.

Исполнитель обещает: «я опрошу тебя снова после пробуждения». Future обещает: «я договорюсь о пробуждении». Если future нарушает обещание, задача спит вечно. Вот такой ошибочный future в `src/bin/lost_wake.rs`:

```rust
use std::future::Future;
use std::pin::Pin;
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll};
use std::thread;
use std::time::{Duration, Instant};

/// Like `Delay`, but it forgets to keep the waker. BUG.
struct ForgetfulDelay {
    done: Arc<Mutex<bool>>,
}

impl ForgetfulDelay {
    fn new(duration: Duration) -> Self {
        let done = Arc::new(Mutex::new(false));
        let for_thread = Arc::clone(&done);
        thread::spawn(move || {
            thread::sleep(duration);
            *for_thread.lock().unwrap() = true; // nobody is told
        });
        ForgetfulDelay { done }
    }
}

impl Future for ForgetfulDelay {
    type Output = ();

    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<()> {
        if *self.done.lock().unwrap() {
            Poll::Ready(())
        } else {
            Poll::Pending // BUG: no waker was stored
        }
    }
}

#[tokio::main]
async fn main() {
    let start = Instant::now();
    let delay = ForgetfulDelay::new(Duration::from_millis(100));
    let result = tokio::time::timeout(Duration::from_secs(1), delay).await;
    println!("{result:?} after {} ms", start.elapsed().as_millis());
}
```

```text
Ok(()) after 1007 ms
```

**Что происходит:**

1. Tokio опрашивает `ForgetfulDelay` один раз. Тот ещё не готов, возвращает `Pending` и не сохраняет waker.
2. Через 100 мс вспомогательный поток ставит `done = true`, но сообщить об этом никому не может. Tokio не опрашивает задачу снова: он ждёт пробуждения, которое никогда не придёт.
3. На 1000-й миллисекунде срабатывает таймер `timeout`. Таймер Tokio *будит* задачу. `timeout` сначала опрашивает внутренний future, который теперь видит `done == true` и возвращает `Ready`.
4. Результат — `Ok(())`, но на 900 мс позже и только по везению. Без `timeout` строка `delay.await` зависает навсегда (попробуйте и остановите программу через Ctrl+C).

> ⚠️ **Частая ошибка:** future, который возвращает `Pending`, должен где-то сохранить waker или передать его тому, кто его вызовет. «Скоро буду готов» недостаточно: исполнитель не угадывает.

## 4. Правильный листовой future: `Delay`

Теперь правильный способ. Future и вспомогательный поток делят маленькое состояние: «мы закончили?» и «кого будить?». Поместите это в `src/lib.rs` (полный файл с `block_on` из раздела 5 в конце — это то, что у вас будет после урока):

```rust
//! Hand-written futures and a tiny `block_on`.

use std::future::Future;
use std::pin::{Pin, pin};
use std::sync::atomic::{AtomicUsize, Ordering};
use std::sync::{Arc, Mutex};
use std::task::{Context, Poll, Wake, Waker};
use std::thread::{self, Thread};
use std::time::Duration;

/// The state that the future and the helper thread share.
struct Shared {
    done: bool,
    waker: Option<Waker>,
}

/// A future that becomes ready after `duration`.
/// A helper thread sleeps and then wakes the task.
pub struct Delay {
    shared: Arc<Mutex<Shared>>,
}

impl Delay {
    pub fn new(duration: Duration) -> Self {
        let shared = Arc::new(Mutex::new(Shared {
            done: false,
            waker: None,
        }));
        let for_thread = Arc::clone(&shared);
        thread::spawn(move || {
            thread::sleep(duration);
            let mut state = for_thread.lock().unwrap();
            state.done = true;
            if let Some(waker) = state.waker.take() {
                waker.wake();
            }
        });
        Delay { shared }
    }
}

impl Future for Delay {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        let mut state = self.shared.lock().unwrap();
        if state.done {
            Poll::Ready(())
        } else {
            state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

**Что происходит:**

1. `Delay::new` создаёт общее состояние и запускает вспомогательный поток. Поток спит, затем ставит `done = true` и, если waker сохранён, достаёт его и вызывает `wake()`.
2. `poll` блокирует состояние. Если `done`, возвращает `Ready`. Иначе сохраняет **клон** текущего waker (`Context` его только одалживает) и возвращает `Pending`. Контракт соблюдён: тот, кто поставит `done`, нас разбудит.
3. Каждый опрос перезаписывает сохранённый waker. Это важно: между опросами задачу могут перенести на другой поток или в другой исполнитель, и тогда правильным будет только самый новый waker. (`Waker::will_wake` может сказать, тот же ли это waker, чтобы не клонировать лишний раз.)
4. Проверка `done` и сохранение waker происходят **под одной блокировкой**. Иначе был бы возможен такой порядок: `poll` видит `done == false`; вспомогательный поток ставит `done = true` и не находит waker; `poll` сохраняет waker и возвращает `Pending`. Пробуждение потерялось бы навсегда. Блокировка делает «проверить и сохранить» одним шагом с точки зрения вспомогательного потока.

`Delay` ничего не знает о Tokio. Он работает с любым исполнителем, который даёт ему waker. `src/bin/delay.rs` запускает его под двумя разными:

```rust
use p14_futures::Delay;
use std::time::{Duration, Instant};

#[tokio::main]
async fn main() {
    let start = Instant::now();
    Delay::new(Duration::from_millis(200)).await;
    println!("tokio: ready after {} ms", start.elapsed().as_millis());

    let start = Instant::now();
    futures::executor::block_on(Delay::new(Duration::from_millis(200)));
    println!("futures: ready after {} ms", start.elapsed().as_millis());
}
```

```text
tokio: ready after 204 ms
futures: ready after 201 ms
```

(Вызов `futures::executor::block_on` внутри задачи Tokio блокирует поток рантайма, а Урок 06.9 говорил так не делать; это допустимо только в крошечном эксперименте вроде этого.)

> ⚡ **Производительность:** один поток на каждый `Delay` годится для учёбы, но дорог для настоящих программ: поток стоит памяти и работы операционной системы. Настоящие рантаймы хранят *все* таймеры в одной структуре данных и спят один раз до ближайшего срока. Это вы построите в [Уроке 14.11](11-build-a-tiny-async-runtime.md).

## 5. `Waker` из трейта `Wake` и `block_on`, который спит

Сторона исполнителя в контракте: создать waker, опросить и, если ответ `Pending`, спать, пока waker не вызовут. Для одного future в одном потоке «спать» — это `thread::park()`, а «разбудить» — `unpark()`.

Построить `Waker` вручную — низкоуровневая работа (`RawWaker` с таблицей из четырёх `unsafe`-функций). Стандартная библиотека даёт безопасный способ: реализуйте трейт `Wake` для типа, положите его в `Arc` и преобразуйте. Добавьте это в конец `src/lib.rs`:

```rust
/// Wakes a thread that sleeps in `block_on`.
struct ThreadWaker {
    thread: Thread,
}

impl Wake for ThreadWaker {
    fn wake(self: Arc<Self>) {
        self.thread.unpark();
    }
}

/// How many times `block_on` called `poll` (for the experiments in this lesson).
pub static POLLS: AtomicUsize = AtomicUsize::new(0);

/// Runs one future to completion on the current thread.
pub fn block_on<F: Future>(future: F) -> F::Output {
    let mut future = pin!(future);
    let waker = Waker::from(Arc::new(ThreadWaker {
        thread: thread::current(),
    }));
    let mut cx = Context::from_waker(&waker);
    loop {
        POLLS.fetch_add(1, Ordering::Relaxed);
        match future.as_mut().poll(&mut cx) {
            Poll::Ready(value) => return value,
            Poll::Pending => thread::park(),
        }
    }
}
```

**Что происходит:**

1. `ThreadWaker` запоминает поток, который выполняет `block_on` (`thread::current()`).
2. `impl Wake` говорит, что для нас значит «разбудить»: сделать `unpark` этому потоку. Метод принимает `self: Arc<Self>`, потому что waker общий и его можно клонировать много раз. `Waker::from(Arc::new(...))` безопасно строит за нас низкоуровневый `RawWaker`.
3. Цикл делает один опрос. При `Ready` возвращает значение. При `Pending` паркует поток: поток спит и не тратит CPU.
4. Когда вспомогательный поток `Delay` вызывает `wake()`, наш поток просыпается и опрашивает снова.
5. Если `unpark` случается *до* `park`, ничего не теряется: `unpark` оставляет «жетон», и следующий `park` сразу возвращается. А если `park` вернётся раньше без причины (ложное пробуждение), цикл просто опросит снова и снова получит `Pending`. Опрашивать слишком часто можно; ошибка — никогда не опросить снова.

Попробуйте с `async fn`, которая ждёт две задержки, в `src/bin/my_block_on.rs`:

```rust
use p14_futures::{Delay, POLLS, block_on};
use std::sync::atomic::Ordering;
use std::time::{Duration, Instant};

async fn add_slowly(a: u32, b: u32) -> u32 {
    Delay::new(Duration::from_millis(100)).await;
    Delay::new(Duration::from_millis(100)).await;
    a + b
}

fn main() {
    let start = Instant::now();
    let sum = block_on(add_slowly(2, 3));
    println!("sum = {sum} after {} ms", start.elapsed().as_millis());
    println!("poll was called {} times", POLLS.load(Ordering::Relaxed));
}
```

```text
sum = 5 after 205 ms
poll was called 3 times
```

Три опроса: один в начале и по одному после каждой задержки. Между ними поток спит. Ровно так же работает и Tokio, только с множеством задач, множеством потоков и одним общим таймером.

Для сравнения — активный цикл из Урока 06.9 с `Delay` на 200 мс, в `src/bin/busy.rs`:

```rust
use p14_futures::Delay;
use std::future::Future;
use std::pin::pin;
use std::task::{Context, Poll, Waker};
use std::time::Duration;

/// The `block_on` from Lesson 06.9: it never sleeps.
fn busy_block_on<F: Future>(future: F) -> (F::Output, u64) {
    let mut future = pin!(future);
    let mut cx = Context::from_waker(Waker::noop());
    let mut polls = 0;
    loop {
        polls += 1;
        if let Poll::Ready(value) = future.as_mut().poll(&mut cx) {
            return (value, polls);
        }
    }
}

fn main() {
    let ((), polls) = busy_block_on(Delay::new(Duration::from_millis(200)));
    println!("busy loop: poll was called {polls} times");
}
```

`cargo run --release --bin busy`:

```text
busy loop: poll was called 39946637 times
```

Почти 40 миллионов вызовов впустую (точное число меняется при каждом запуске) против 2 с работающим waker (один опрос, который возвращает `Pending`, и один после пробуждения).

> 🧪 **Тестирование:** написанный вручную future легко тестировать написанным вручную исполнителем: `block_on` превращает асинхронный код обратно в обычную функцию, поэтому подходит обычный `#[test]`. Это делает упражнение 2.

## Частые ошибки

**1. `Pending` без договорённости о пробуждении.** Задачу больше никогда не опрашивают (раздел 3). Каждому `Pending` нужен сохранённый waker или waker, переданный тому, кто его вызовет.

**2. Проверка условия и сохранение waker в два отдельных шага.** Событие может случиться между ними, и его пробуждение потеряется. Держите «проверить» и «сохранить waker» под одной блокировкой (или используйте атомарный протокол и проверяйте ещё раз после сохранения).

**3. Опрос future после того, как он вернул `Ready`.** Future, созданные компилятором, паникуют с «`async fn` resumed after completion». Написанные вручную могут вернуть бессмыслицу. Исполнители выбрасывают future, как только он стал `Ready`.

## Попробуйте сами

1. Создайте `code/p14-futures`, как в разделе 1, и запустите `lazy`, `state_machine` и `delay`.
2. Добавьте `ForgetfulDelay` (`src/bin/lost_wake.rs`) и запустите. Затем уберите `timeout` (напишите `delay.await;`) и убедитесь, что программа зависает. Остановите её через Ctrl+C.
3. Исправьте `ForgetfulDelay`, дав ему такое же состояние `Shared`, как у `Delay`. Программа должна завершаться примерно через 100 мс.
4. Добавьте `block_on` и запустите `my_block_on`. Измените программу так, чтобы она ждала три задержки, и предскажите число опросов до запуска.
5. Запустите `cargo clippy --all-targets` и `cargo fmt`, затем сделайте коммит: `git add code/p14-futures && git commit -m "p14-futures: Delay and block_on"`.

## Упражнения

### Упражнение 1 — Посчитайте опросы (легко)

Напишите future `YieldTimes { left: u32 }`, который `left` раз возвращает `Pending` (каждый раз будя себя), а затем `Ready(())`. Запустите `YieldTimes { left: 3 }` своим `block_on` и напечатайте `POLLS`. Сколько опросов вы ожидаете?

<details>
<summary>Подсказка</summary>

Скопируйте `YieldOnce` из раздела 2 и замените `bool` счётчиком, который уменьшается. `wake_by_ref` делает `unpark` нашему же потоку, поэтому следующий `park` сразу возвращается.

</details>

<details>
<summary>Ответ</summary>

```rust
use p14_futures::{POLLS, block_on};
use std::future::Future;
use std::pin::Pin;
use std::sync::atomic::Ordering;
use std::task::{Context, Poll};

/// Returns `Pending` `times` times, then `Ready`.
struct YieldTimes {
    left: u32,
}

impl Future for YieldTimes {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        if self.left == 0 {
            Poll::Ready(())
        } else {
            self.left -= 1;
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

fn main() {
    block_on(YieldTimes { left: 3 });
    println!("poll was called {} times", POLLS.load(Ordering::Relaxed));
}
```

```text
poll was called 4 times
```

Почему это работает: три ответа `Pending` и один `Ready` дают четыре опроса. Каждый `wake_by_ref` вызывает `unpark` до `park`, поэтому `park` не засыпает: жетон уже на месте.

</details>

### Упражнение 2 — Протестируйте исполнитель (средне)

Добавьте в `src/lib.rs` модуль `#[cfg(test)] mod tests` с тремя тестами: `block_on` возвращает значение блока `async`; `Delay` на 50 мс под вашим `block_on` длится не меньше 50 мс; `Delay` завершается и под `futures::executor::block_on`.

<details>
<summary>Подсказка</summary>

`block_on(async { 40 + 2 })` — обычный вызов функции, поэтому `assert_eq!` работает. Для времени используйте `std::time::Instant` и сравнивайте через `>=`, никогда через `==`: сон всегда может длиться чуть дольше.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::time::Instant;

    #[test]
    fn block_on_returns_the_value_of_an_async_block() {
        assert_eq!(block_on(async { 40 + 2 }), 42);
    }

    #[test]
    fn delay_waits_at_least_its_duration() {
        let start = Instant::now();
        block_on(Delay::new(Duration::from_millis(50)));
        assert!(start.elapsed() >= Duration::from_millis(50));
    }

    #[test]
    fn delay_works_under_another_executor() {
        futures::executor::block_on(Delay::new(Duration::from_millis(10)));
    }
}
```

`cargo test` печатает `test result: ok. 3 passed; 0 failed` для библиотеки (и `0 tests` для каждой программы в `src/bin/`). Почему это работает: `block_on` доводит future до конца в потоке теста. Третий тест доказывает, что `Delay` соблюдает контракт и не зависит от одного исполнителя.

</details>

### Упражнение 3 — Собственный `join` (сложно)

Напишите `Join2<A, B>` — future, который опрашивает два future и готов, когда готовы **оба**, с результатом `(A::Output, B::Output)`. Запустите им два `Delay` по 300 мс под своим `block_on`. Это должно занять около 300 мс, а не 600 мс. Можно требовать `A: Future + Unpin` и `B: Future + Unpin`.

<details>
<summary>Подсказка</summary>

Храните оба future и два `Option` для их результатов. В `poll` опрашивайте каждый future, у которого ещё нет результата, через `Pin::new(&mut this.a)`. `self.get_mut()` превращает `Pin<&mut Self>` в `&mut Self`, что разрешено, только когда тип `Unpin` (кратко: `Unpin` значит «можно перемещать даже после закрепления»; подробно — в Уроке 14.12). `Join2` будет `Unpin`, только если все его поля `Unpin`, поэтому результаты тоже должны быть `Unpin`. Оба future получают один waker: кто бы ни разбудил, опрашивается весь `Join2`.

</details>

<details>
<summary>Ответ</summary>

```rust
use p14_futures::{Delay, block_on};
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};
use std::time::{Duration, Instant};

/// Runs two futures at the same time and finishes when both are done.
struct Join2<A: Future, B: Future> {
    a: A,
    b: B,
    a_out: Option<A::Output>,
    b_out: Option<B::Output>,
}

impl<A, B> Future for Join2<A, B>
where
    A: Future + Unpin,
    B: Future + Unpin,
    A::Output: Unpin,
    B::Output: Unpin,
{
    type Output = (A::Output, B::Output);

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let this = self.get_mut();
        if this.a_out.is_none()
            && let Poll::Ready(value) = Pin::new(&mut this.a).poll(cx)
        {
            this.a_out = Some(value);
        }
        if this.b_out.is_none()
            && let Poll::Ready(value) = Pin::new(&mut this.b).poll(cx)
        {
            this.b_out = Some(value);
        }
        if this.a_out.is_some() && this.b_out.is_some() {
            Poll::Ready((this.a_out.take().unwrap(), this.b_out.take().unwrap()))
        } else {
            Poll::Pending
        }
    }
}

fn join2<A: Future, B: Future>(a: A, b: B) -> Join2<A, B> {
    Join2 {
        a,
        b,
        a_out: None,
        b_out: None,
    }
}

fn main() {
    let start = Instant::now();
    let both = join2(
        Delay::new(Duration::from_millis(300)),
        Delay::new(Duration::from_millis(300)),
    );
    block_on(both);
    println!("both done after {} ms", start.elapsed().as_millis());
}
```

```text
both done after 300 ms
```

`if cond && let Pattern = value { … }` — это *цепочка let* (let chain, редакция 2024): тело выполняется, только если удались обе части. Почему это работает: обе задержки запускают свои вспомогательные потоки при создании, поэтому ждут одновременно. Каждый опрос `Join2` передаёт один и тот же waker обоим внутренним future, а готовый результат хранится в своём `Option`, поэтому завершённый future больше не опрашивается.

</details>

## Вопросы для самопроверки

1. Вы вызвали `async fn`, но ни разу не сделали `.await`. Что выполнится?

<details>
<summary>Ответ</summary>

Ничего из её тела. Вызов только создаёт future (конечный автомат с аргументами). Тело выполняется, когда кто-то опрашивает future.

</details>

2. Чему соответствует один вариант конечного автомата, который строит компилятор?

<details>
<summary>Ответ</summary>

Одному месту, где функция может остановиться: начало, каждый `.await` и конец. Вариант хранит ожидаемый future и локальные переменные, которые должны пережить этот `.await`.

</details>

3. Что именно future должен сделать, прежде чем вернуть `Pending`?

<details>
<summary>Ответ</summary>

Позаботиться, чтобы текущий waker (`cx.waker()`) был разбужен, когда прогресс станет возможен: сохранить клон там, где его найдёт источник события, или передать его этому источнику. Иначе исполнитель никогда не опросит его снова.

</details>

4. Почему `Delay::poll` проверяет `done` и сохраняет waker под одной блокировкой?

<details>
<summary>Ответ</summary>

Чтобы не потерять пробуждение. Если бы вспомогательный поток поставил `done` и искал waker между нашей проверкой и нашим сохранением, он бы ничего не нашёл, а мы потом вернули бы `Pending`, и будить нас было бы уже некому.

</details>

5. Что даёт `impl Wake for ThreadWaker` и почему `wake` принимает `self: Arc<Self>`?

<details>
<summary>Ответ</summary>

Он позволяет `Waker::from(Arc::new(ThreadWaker { … }))` построить настоящий `Waker` без `unsafe`-кода. `wake` принимает `Arc`, потому что waker общий: его можно клонировать и раздавать во многие места, и данные должны жить, пока жив хоть один клон.

</details>

## Итоги

- Future ленивы: вызов `async fn` создаёт конечный автомат, и ничего не выполняется, пока его не опросят.
- Компилятор превращает каждую `async fn` в похожий на `enum` конечный автомат с одним состоянием на каждый `.await`; `.await` возвращает `Pending` вверх по цепочке, а не блокирует поток.
- Контракт: возвращайте `Pending`, только договорившись, что `cx.waker()` будет разбужен; исполнитель опрашивает снова только после пробуждения.
- Листовой future вроде `Delay` хранит самый новый waker в общем состоянии, проверяет и сохраняет под одной блокировкой и работает с любым исполнителем.
- `std::task::Wake` безопасно строит `Waker`; вместе с `thread::park`/`unpark` он даёт `block_on`, который спит, а не опрашивает впустую.
- Потерянные пробуждения (waker не сохранён или гонка между проверкой и сохранением) приводят к зависанию задач; лишние опросы безвредны.

## Источники и что почитать

- [std::future::Future](https://doc.rust-lang.org/std/future/trait.Future.html) — трейт и точные правила для `poll` и `Pending`.
- [std::task::Wake](https://doc.rust-lang.org/std/task/trait.Wake.html) — трейт с собственным примером `block_on`.
- [std::task::Waker](https://doc.rust-lang.org/std/task/struct.Waker.html) — `wake`, `wake_by_ref`, `will_wake` и `noop`.
- [The Rust Programming Language — 17.5 A Closer Look at the Traits for Async](https://doc.rust-lang.org/book/ch17-05-traits-for-async.html) — `Future`, `Pin` и конечные автоматы в официальной книге.
- [Tokio tutorial — Async in depth](https://tokio.rs/tokio/tutorial/async) — как Tokio использует waker, с написанным вручную `Delay`.
- [Asynchronous Programming in Rust (the async book)](https://rust-lang.github.io/async-book/) — официальная книга об асинхронности.

---

[🏠 Начало](../START-HERE.md) · [← 14.9 Тестирование конкурентного кода с loom](09-testing-concurrent-code-with-loom.md) · [📋 Этап 14](README.md) · [14.11 Пишем крошечный асинхронный рантайм →](11-build-a-tiny-async-runtime.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
