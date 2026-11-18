<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 14.2 Мьютексы и блокировки чтения-записи подробно](02-mutexes-and-rwlocks-in-depth.md) · [📋 Этап 14](README.md) · [14.4 Пулы потоков, scoped-потоки и rayon →](04-thread-pools-scoped-threads-and-rayon.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/14-concurrency-and-async/03-condition-variables.md) <!-- nav -->

# Урок 14.3 — Условные переменные

> **Этап 14 — Конкурентность и асинхронность подробно** · Урок 3 из 16 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Почему поток не должен ждать чего-то, проверяя это снова и снова.
- Как условная переменная (`Condvar`) позволяет потоку спать, пока другой поток не изменит состояние.
- Три правила условных переменных: мьютекс, условие и цикл.
- Что такое ложные пробуждения и потерянные пробуждения и почему правила защищают от тех и других.
- Как построить ограниченную блокирующую очередь на одном мьютексе и двух условных переменных и как ждать с тайм-аутом.

## Перед началом

- Вы прошли [Урок 14.2 — Мьютексы и блокировки чтения-записи подробно](02-mutexes-and-rwlocks-in-depth.md). Вы знаете guard-объекты, критические секции и `drop(guard)`.
- Вы знаете `VecDeque` из [Урока 07.5 — Стеки и очереди](../07-algorithms-and-data-structures/05-stacks-and-queues.md) и ограниченную очередь сервера с пулом потоков из [Урока 06.11](../06-rust-advanced/11-project-multithreaded-web-server.md).
- Продолжайте работать в `~/projects/rust/code/p14-concurrency`; каждый пример — новый файл в `src/bin/`.

## Новые слова

| Слово | Значение |
|---|---|
| **активное ожидание** (busy waiting) | Ожидание, при котором условие проверяется в цикле снова и снова, без сна. Тратит целое ядро CPU (Central Processing Unit, центральный процессор). |
| **условная переменная** (condition variable, `Condvar`) | Место, где потоки могут спать, пока другой поток не скажет «состояние изменилось, посмотрите снова». Всегда работает вместе с мьютексом. |
| **`wait`** | Метод `Condvar`, который одним шагом отпускает мьютекс и усыпляет поток. Когда поток просыпается, он снова блокирует мьютекс, прежде чем `wait` вернёт управление. |
| **уведомление** (notify: `notify_one`, `notify_all`) | Разбудить один ожидающий поток или все. |
| **предикат** (predicate) | Вопрос «да или нет», который ожидающий поток задаёт о состоянии, например «очередь не пуста?». |
| **ложное пробуждение** (spurious wake-up) | Поток возвращается из `wait`, хотя его никто не уведомлял, или условие уже снова ложно. Такое разрешено. |
| **потерянное пробуждение** (lost wake-up) | Уведомление отправлено, когда никто ещё не ждёт, поэтому оно пропадает, а поток, начавший ждать позже, спит вечно. |
| **производитель / потребитель** (producer / consumer) | Поток, который создаёт элементы работы и кладёт их в очередь / поток, который достаёт их и обрабатывает. |
| **ограниченная очередь** (bounded queue) | Очередь с максимальным размером. Когда она полна, производителю приходится ждать. |
| **блокирующая операция** (blocking) | Операция *блокирует*, когда вызвавший поток останавливается и ждёт внутри неё (например `lock()`, `recv()`, `pop()` на пустой блокирующей очереди). |
| **защёлка** (latch) | Одноразовые ворота: потоки ждут, пока счётчик не дойдёт до нуля, а потом все могут идти дальше. |

## 1. Проблема: ждать чего-то

Рабочий поток загружает файл конфигурации; главный поток должен дождаться, пока он будет готов. Имея только мьютекс, главный поток может лишь спрашивать «уже готово?» снова и снова. Создайте `src/bin/busy_wait.rs`:

```rust
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    let ready = Arc::new(Mutex::new(false));

    let worker_ready = Arc::clone(&ready);
    let worker = thread::spawn(move || {
        thread::sleep(Duration::from_millis(100)); // pretend to load a config
        *worker_ready.lock().unwrap() = true;
    });

    // Busy waiting: ask again and again "are we there yet?"
    let mut checks: u64 = 0;
    while !*ready.lock().unwrap() {
        checks += 1;
    }
    println!("ready after {checks} checks");
    worker.join().unwrap();
}
```

`cargo run --release --bin busy_wait`:

```text
ready after 20770610 checks
```

**Что происходит:**

1. Рабочий поток спит 100 мс, затем под блокировкой ставит флаг в `true`.
2. Главный поток блокирует, читает флаг, разблокирует и повторяет. За 100 мс он проверил флаг около 20 миллионов раз.
3. Всё это время одно ядро CPU было загружено на 100 % бесполезной работой, а главный поток ещё и боролся с рабочим за блокировку.

Можно добавить `thread::sleep(Duration::from_millis(10))` внутрь цикла. Это сбережёт CPU, но теперь главный поток замечает изменение с опозданием до 10 мс, и хорошее время сна приходится угадывать. На самом деле нам нужно вот что: **спать, пока кто-нибудь не скажет, что состояние изменилось**. Именно это и делает условная переменная.

## 2. `Condvar`: спать, пока не уведомят

Условная переменная похожа на звонок «позвоните, пожалуйста» у прилавка. Вы не стоите у прилавка, крича «здесь кто-нибудь есть?». Вы звоните один раз и садитесь; продавец приходит, когда заказ готов, и зовёт вас.

У `std::sync::Condvar` три важных метода:

| Метод | Что делает |
|---|---|
| `cvar.wait(guard)` | Принимает `MutexGuard`, **отпускает мьютекс и засыпает** одним шагом. Когда поток будят, **снова блокирует мьютекс** и возвращает guard. |
| `cvar.notify_one()` | Будит один поток, спящий в `wait` на этой условной переменной (если такой есть). |
| `cvar.notify_all()` | Будит их все. |

Вот та же программа с условной переменной. Мьютекс и условная переменная всегда идут вместе, поэтому храним их в одном кортеже. Создайте `src/bin/ready_flag.rs`:

```rust
use std::sync::{Arc, Condvar, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));

    let worker_pair = Arc::clone(&pair);
    let worker = thread::spawn(move || {
        thread::sleep(Duration::from_millis(100)); // pretend to load a config
        let (lock, cvar) = &*worker_pair;
        *lock.lock().unwrap() = true; // 1. change the state under the lock
        cvar.notify_one(); // 2. wake up a waiting thread
        println!("worker: config loaded");
    });

    let (lock, cvar) = &*pair;
    let mut ready = lock.lock().unwrap();
    let mut wakeups = 0;
    while !*ready {
        ready = cvar.wait(ready).unwrap(); // unlock, sleep, lock again
        wakeups += 1;
    }
    println!("main: ready after {wakeups} wake-up(s)");
    drop(ready);
    worker.join().unwrap();
}
```

```text
worker: config loaded
main: ready after 1 wake-up(s)
```

**Что происходит:**

1. `&*worker_pair` превращает `Arc<(Mutex, Condvar)>` в ссылку на кортеж, а `let (lock, cvar) = …` даёт имена обеим частям.
2. Главный поток блокирует мьютекс и видит `false`. Он вызывает `cvar.wait(ready)`: guard передаётся, мьютекс отпускается, поток засыпает. Пока он спит, он не тратит CPU.
3. Теперь рабочий поток может взять блокировку (она свободна), ставит флаг в `true` и вызывает `notify_one`.
4. Главный поток просыпается. Прежде чем `wait` вернёт управление, он снова блокирует мьютекс, так что `ready` опять обычный guard. Цикл проверяет флаг: `true`, и цикл заканчивается. Понадобилось одно пробуждение вместо 20 миллионов проверок.

```text
 main                                   worker
 lock ─ flag is false
 wait(guard): unlock + sleep  ─────┐
                                   │    lock ─ flag = true ─ unlock
                                   │    notify_one() ──────────┐
 (woken) lock again ◀──────────────┴───────────────────────────┘
 flag is true → leave the loop
```

Главное здесь — шаг 2: отпустить мьютекс и начать спать — это **один** шаг. Если бы это были два шага, рабочий поток мог бы поставить флаг и отправить уведомление в промежутке между ними, и главный поток спал бы вечно. Этот промежуток — **потерянное пробуждение** из раздела 3.

## 3. Три правила

Любое корректное использование условной переменной следует трём правилам.

**Правило 1: условие хранится в переменной под защитой мьютекса.** Сама условная переменная ничего не помнит. Это только зал ожидания. Истина («готово ли?», «пуста ли очередь?») должна лежать в обычных данных за мьютексом.

**Правило 2: меняйте состояние под блокировкой, затем уведомляйте.** Если поменять флаг без блокировки, ожидающий поток может проверить его, увидеть `false` и уснуть сразу после вашего уведомления.

**Правило 3: всегда ждите в цикле, который проверяет условие.** Причин две:

- **Ложное пробуждение** разрешено: `wait` может вернуться, хотя никто не вызывал notify. Операционные системы допускают это, потому что так условные переменные проще и быстрее реализовать.
- Даже после настоящего уведомления другой поток мог успеть первым и снова изменить состояние (например, забрать единственный элемент из очереди).

Цикл `while !condition { guard = cvar.wait(guard).unwrap(); }` справляется с обоими случаями: после каждого пробуждения проверить снова.

Что ломается без правила 1? Уведомление, которого никто не ждёт, просто теряется. В этой программе рабочий поток уведомляет раньше, чем главный начинает ждать. Создайте `src/bin/lost_wakeup.rs`:

```rust
use std::sync::{Arc, Condvar, Mutex};
use std::thread;
use std::time::Duration;

fn main() {
    // WRONG: a condition variable without a condition to check.
    let pair = Arc::new((Mutex::new(()), Condvar::new()));

    let worker_pair = Arc::clone(&pair);
    thread::spawn(move || {
        let (_lock, cvar) = &*worker_pair;
        cvar.notify_one(); // "I am done!" ... but nobody is waiting yet
    });

    thread::sleep(Duration::from_millis(100)); // main is a bit late
    let (lock, cvar) = &*pair;
    let guard = lock.lock().unwrap();
    let (_guard, result) = cvar.wait_timeout(guard, Duration::from_secs(1)).unwrap();
    println!("timed out: {}", result.timed_out());
}
```

```text
timed out: true
```

**Что происходит:** уведомление пришло первым и потерялось, потому что в тот момент ни один поток не спал в `wait`. С обычным `wait` главный поток спал бы вечно; `wait_timeout` (раздел 5) сдаётся через секунду, поэтому проблему видно. В `ready_flag.rs` такой же порядок событий безвреден: главный поток проверяет флаг *до* того, как ждать, видит `true` и вообще не засыпает.

`Condvar::wait_while` пишет этот цикл за вас. Он принимает guard и замыкание, которое отвечает на вопрос «ждать ли дальше?»:

```rust
    let (lock, cvar) = &*pair;
    let ready = cvar.wait_while(lock.lock().unwrap(), |ready| !*ready).unwrap();
```

Эти две строки делают ту же работу, что строка `let mut ready = …` и цикл `while` в `ready_flag.rs` (только без подсчёта пробуждений). `wait_while` использует и очередь из следующего раздела.

> ⚠️ **Частая ошибка:** `if !ready { guard = cvar.wait(guard).unwrap(); }` — `if` вместо `while`. В тестах работает, а в продакшене ломается после ложного пробуждения или когда два потребителя соревнуются за один элемент. Всегда используйте `while` или `wait_while`.

## 4. Ограниченная блокирующая очередь

Теперь классическое применение: очередь между **производителями** (потоками, которые создают работу) и **потребителями** (потоками, которые её выполняют). У неё есть максимальный размер, поэтому быстрый производитель не может заполнить всю память — ему приходится ждать. В Уроке 06.11 для этого использовался `mpsc::sync_channel`. Здесь мы построим очередь сами, потому что так видно, как такой канал устроен внутри.

Причин ждать две, поэтому и условных переменных две:

| Кто ждёт | Чего | Condvar | Кто уведомляет |
|---|---|---|---|
| потребитель в `pop` | «очередь не пуста» | `not_empty` | `push`, после добавления элемента |
| производитель в `push` | «очередь не полна» | `not_full` | `pop`, после удаления элемента |

Ещё есть флаг `closed`, чтобы потребители знали, что элементов больше не будет. Создайте `src/bin/blocking_queue.rs`:

```rust
use std::collections::VecDeque;
use std::sync::{Condvar, Mutex};
use std::thread;
use std::time::Duration;

struct State<T> {
    items: VecDeque<T>,
    closed: bool,
}

/// A queue with a fixed capacity. `push` waits while it is full,
/// `pop` waits while it is empty.
pub struct BlockingQueue<T> {
    state: Mutex<State<T>>,
    not_empty: Condvar,
    not_full: Condvar,
    capacity: usize,
}

impl<T> BlockingQueue<T> {
    pub fn new(capacity: usize) -> Self {
        assert!(capacity > 0, "capacity must be at least 1");
        BlockingQueue {
            state: Mutex::new(State {
                items: VecDeque::new(),
                closed: false,
            }),
            not_empty: Condvar::new(),
            not_full: Condvar::new(),
            capacity,
        }
    }

    /// Waits for free space, then adds `item`. Returns `Err(item)` if the queue is closed.
    pub fn push(&self, item: T) -> Result<(), T> {
        let state = self.state.lock().unwrap();
        let mut state = self
            .not_full
            .wait_while(state, |s| s.items.len() == self.capacity && !s.closed)
            .unwrap();
        if state.closed {
            return Err(item);
        }
        state.items.push_back(item);
        self.not_empty.notify_one();
        Ok(())
    }

    /// Waits for an item. Returns `None` when the queue is closed and empty.
    pub fn pop(&self) -> Option<T> {
        let state = self.state.lock().unwrap();
        let mut state = self
            .not_empty
            .wait_while(state, |s| s.items.is_empty() && !s.closed)
            .unwrap();
        let item = state.items.pop_front();
        if item.is_some() {
            self.not_full.notify_one();
        }
        item
    }

    /// No more pushes. Waiting threads wake up; `pop` drains what is left.
    pub fn close(&self) {
        self.state.lock().unwrap().closed = true;
        self.not_empty.notify_all();
        self.not_full.notify_all();
    }
}

fn main() {
    let queue = BlockingQueue::new(2);
    thread::scope(|s| {
        s.spawn(|| {
            for i in 1..=5 {
                queue.push(i).unwrap();
                println!("produced {i}");
            }
            queue.close();
        });
        s.spawn(|| {
            while let Some(i) = queue.pop() {
                println!("    consumed {i}");
                thread::sleep(Duration::from_millis(50)); // a slow consumer
            }
            println!("    queue closed, consumer stops");
        });
    });
}
```

```text
produced 1
produced 2
produced 3
    consumed 1
    consumed 2
produced 4
    consumed 3
produced 5
    consumed 4
    consumed 5
    queue closed, consumer stops
```

**Что происходит:**

1. Все данные (`items` и `closed`) лежат внутри **одного** мьютекса (правило 1). Обе условные переменные используются с этим же мьютексом.
2. `push` ждёт на `not_full`, пока очередь полна и не закрыта. Когда можно продолжать, он добавляет элемент и вызывает `not_empty.notify_one()`, чтобы разбудить одного спящего потребителя (правило 2: состояние изменилось под блокировкой).
3. `pop` ждёт на `not_empty`, пока очередь пуста и не закрыта. Забрав элемент, он освободил место, поэтому будит одного производителя через `not_full.notify_one()`.
4. Ёмкость равна 2, а потребитель медленный, поэтому перед элементами 4 и 5 производитель вынужден ждать свободного места. Это ожидание и есть **обратное давление** (backpressure, Урок 06.11): быстрая сторона замедляется до скорости медленной, и расход памяти остаётся ограниченным.
5. `close` ставит флаг и будит **всех** через `notify_all`: каждый ожидающий потребитель должен увидеть, что очередь закрыта, а не только один. `pop` по-прежнему отдаёт оставшиеся элементы и возвращает `None` только когда очередь закрыта **и** пуста, поэтому ничего не теряется.
6. Строки печатаются *после* возврата из `push` и `pop`, поэтому порядок напечатанных строк может немного отставать от настоящего (здесь потребитель уже забрал элемент 1, когда было напечатано «produced 3»). Ваш вывод может немного отличаться.

> 💡 **Совет:** `notify_one` достаточно, когда продвинуться может любой один ожидающий (один новый элемент — один ждущий потребитель). Используйте `notify_all`, когда изменение важно всем (закрытие, завершение работы) или когда разные ожидающие ждут разных условий на одной условной переменной.

> ⚡ **Производительность:** `notify_one` можно вызывать, держа блокировку (как здесь), или сразу после уничтожения guard-объекта. Уведомление после разблокировки может сэкономить разбуженному потоку немного ожидания блокировки. Корректны оба варианта; держитесь более простого, пока измерение не покажет обратное.

## 5. Ожидание с ограничением по времени

`wait_timeout(guard, duration)` возвращает `(guard, WaitTimeoutResult)`. `result.timed_out()` говорит, истекло ли время. Как и с `wait`, поток может проснуться раньше (ложное пробуждение), поэтому нужен цикл — и этот цикл должен ещё помнить, сколько времени осталось. `wait_timeout_while(guard, duration, condition)` делает всё это за вас:

```rust
    // A fragment: inside a method of the queue from section 4.
    let state = self.state.lock().unwrap();
    let (state, result) = self
        .not_empty
        .wait_timeout_while(state, Duration::from_millis(50), |s| s.items.is_empty())
        .unwrap();
    if result.timed_out() { /* nothing arrived in 50 ms */ }
```

Тайм-ауты — способ не ждать вечно поток, который упал, или производителя, который так и не пришёл. В Упражнении 2 вы добавите в очередь `pop_timeout`.

> 🧪 **Тестирование:** не тестируйте блокирующую структуру одним потоком и вызовами `sleep`. Запустите несколько производителей и потребителей, закройте очередь, когда производители закончат, соберите всё, что получили потребители, отсортируйте и сравните с отправленным: каждый элемент должен прийти **ровно один раз**. Упражнение 1 — именно такой тест.

> 📏 **Соглашение:** прежде чем писать свой код на условных переменных, проверьте, нет ли нужного инструмента в стандартной библиотеке: `mpsc::sync_channel` — ограниченная блокирующая очередь, `std::sync::Barrier` позволяет N потокам ждать друг друга, `OnceLock` ждёт, пока значение будет задано один раз. Пишите код с условными переменными, когда ничего из этого не подходит.

## Частые ошибки

1. **`if` вместо `while` вокруг `wait`.** Ложное пробуждение или другой потребитель могут снова сделать условие ложным. Используйте `while` или `wait_while`.
2. **Условная переменная без условия.** Уведомления не сохраняются. Если уведомить до того, как другой поток начал ждать, уведомление потеряется (раздел 3). Состояние должно жить в мьютексе.
3. **Изменение состояния без блокировки.** Правило 2. Ожидающий может проверить, увидеть старое значение и уснуть сразу после вашего уведомления.
4. **Два разных мьютекса с одной условной переменной.** Все потоки, ждущие на одной условной переменной, должны использовать один и тот же мьютекс. Документация говорит, что `wait` может запаниковать, если условную переменную используют больше чем с одним мьютексом.
5. **`let _ = m.lock().unwrap();`, чтобы «держать» блокировку.** Образец `_` не связывает guard, поэтому он уничтожается сразу. Rust отказывается такое компилировать. В `src/bin/let_underscore.rs`:

   ```rust
   use std::sync::Mutex;

   fn main() {
       let m = Mutex::new(0);
       let _ = m.lock().unwrap(); // meant: "hold the lock for this block"
       println!("is the lock held here?");
   }
   ```

   ```text
   error: non-binding let on a synchronization lock
    --> src/bin/let_underscore.rs:5:9
     |
   5 |     let _ = m.lock().unwrap(); // meant: "hold the lock for this block"
     |         ^ this lock is not assigned to a binding and is immediately dropped
   ```

   Дайте ему имя, начинающееся с подчёркивания, — `let _guard = …`, — тогда он доживёт до конца блока.

## Попробуйте сами

1. В `busy_wait.rs` добавьте `thread::sleep(Duration::from_millis(10));` внутрь цикла `while`. Сколько проверок осталось? (На тестовой машине: `ready after 9 checks`.) Какова цена? Подумайте, насколько поздно главный поток может заметить изменение.
2. В `blocking_queue.rs` измените ёмкость с 2 на 1, а потом на 10. Как меняется порядок строк «produced» и «consumed»? С ёмкостью 10 производителю никогда не приходится ждать: все пять строк «produced» идут первыми.
3. В `ready_flag.rs` удалите строку `cvar.notify_one();` (компилятор теперь предупредит, что `cvar` в рабочем потоке не используется). Запустите программу: она печатает `worker: config loaded` и зависает. Почему? Остановите её через `Ctrl+C`, затем верните строку.

## Упражнения

### Упражнение 1 — Каждый элемент ровно один раз (средне)

Добавьте в `blocking_queue.rs` тест `every_item_arrives_exactly_once`. Используйте очередь ёмкостью 8, 4 производителя (производитель `p` кладёт числа `p * 1000 + i` для `i` из `0..1000`) и 4 потребителя, которые вызывают `pop` до `None` и возвращают то, что получили. Когда все производители закончат, закройте очередь. Отсортированный список полученных чисел должен быть ровно `0..4000`. Добавьте второй, маленький тест: после `close` вызов `push(7)` возвращает `Err(7)`, а `pop()` возвращает `None`.

<details>
<summary>Подсказка</summary>

Используйте `thread::scope`. Храните `ScopedJoinHandle` производителей и потребителей в двух векторах. Дождитесь производителей, вызовите `queue.close()`, затем дождитесь потребителей и через `extend` соберите их результаты в один большой вектор. В замыкании производителя нужен `move` (для `p`), поэтому сначала сделайте ссылку `let queue = &queue;` и перемещайте её.

</details>

<details>
<summary>Ответ</summary>

Добавьте в конец `src/bin/blocking_queue.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn every_item_arrives_exactly_once() {
        let queue = BlockingQueue::new(8);
        let mut received = Vec::new();
        thread::scope(|s| {
            let producers: Vec<_> = (0..4)
                .map(|p| {
                    let queue = &queue;
                    s.spawn(move || {
                        for i in 0..1000 {
                            queue.push(p * 1000 + i).unwrap();
                        }
                    })
                })
                .collect();
            let consumers: Vec<_> = (0..4)
                .map(|_| {
                    s.spawn(|| {
                        let mut got = Vec::new();
                        while let Some(x) = queue.pop() {
                            got.push(x);
                        }
                        got
                    })
                })
                .collect();
            for p in producers {
                p.join().unwrap();
            }
            queue.close(); // all producers are done
            for c in consumers {
                received.extend(c.join().unwrap());
            }
        });
        received.sort();
        let expected: Vec<i32> = (0..4000).collect();
        assert_eq!(received, expected);
    }

    #[test]
    fn push_after_close_gives_the_item_back() {
        let queue = BlockingQueue::new(1);
        queue.close();
        assert_eq!(queue.push(7), Err(7));
        assert_eq!(queue.pop(), None);
    }
}
```

```text
running 2 tests
test tests::push_after_close_gives_the_item_back ... ok
test tests::every_item_arrives_exactly_once ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Почему это работает: если бы элемент потерялся, какого-то числа не хватало бы; если бы он пришёл дважды, число встретилось бы дважды. В обоих случаях отсортированный список отличался бы от `0..4000`. Закрытие только после `join` всех производителей гарантирует, что потребители остановятся, лишь когда всё отправлено. Если бы `close` не будил всех потребителей, тест бы завис, а не прошёл.

</details>

### Упражнение 2 — `pop_timeout` (средне)

Добавьте в `BlockingQueue` метод `pub fn pop_timeout(&self, timeout: Duration) -> Option<T>`: как `pop`, но возвращает `None`, если за `timeout` не пришло ни одного элемента. Напишите два теста: на пустой очереди `pop_timeout(50 мс)` возвращает `None` и занимает не меньше 50 мс; а элемент, который другой поток кладёт через 10 мс, возвращается из `pop_timeout(1 с)`.

<details>
<summary>Подсказка</summary>

Скопируйте `pop` и замените `wait_while(state, …)` на `wait_timeout_while(state, timeout, …)`. Он возвращает кортеж `(guard, result)`. `result` вам даже не нужен: если после ожидания очередь всё ещё пуста, `pop_front()` вернёт `None`. Время в тесте измеряйте через `std::time::Instant`.

</details>

<details>
<summary>Ответ</summary>

Метод внутри `impl<T> BlockingQueue<T>`:

```rust
    /// Like `pop`, but gives up after `timeout`.
    pub fn pop_timeout(&self, timeout: Duration) -> Option<T> {
        let state = self.state.lock().unwrap();
        let (mut state, _result) = self
            .not_empty
            .wait_timeout_while(state, timeout, |s| s.items.is_empty() && !s.closed)
            .unwrap();
        let item = state.items.pop_front();
        if item.is_some() {
            self.not_full.notify_one();
        }
        item
    }
```

Тесты внутри `mod tests` (добавьте туда `use std::time::Instant;`):

```rust
    #[test]
    fn pop_timeout_gives_up_on_an_empty_queue() {
        let queue: BlockingQueue<i32> = BlockingQueue::new(1);
        let start = Instant::now();
        assert_eq!(queue.pop_timeout(Duration::from_millis(50)), None);
        assert!(start.elapsed() >= Duration::from_millis(50));
    }

    #[test]
    fn pop_timeout_returns_an_item_that_arrives_in_time() {
        let queue = BlockingQueue::new(1);
        thread::scope(|s| {
            s.spawn(|| {
                thread::sleep(Duration::from_millis(10));
                queue.push(42).unwrap();
            });
            assert_eq!(queue.pop_timeout(Duration::from_secs(1)), Some(42));
        });
    }
```

```text
running 4 tests
test tests::push_after_close_gives_the_item_back ... ok
test tests::every_item_arrives_exactly_once ... ok
test tests::pop_timeout_returns_an_item_that_arrives_in_time ... ok
test tests::pop_timeout_gives_up_on_an_empty_queue ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.06s
```

Почему это работает: `wait_timeout_while` ждёт, пока очередь пуста, но в сумме никогда не дольше `timeout`, даже после ложных пробуждений. Когда он возвращается, `pop_front` говорит правду: элемент или `None`. Во втором тесте нарочно взят длинный тайм-аут (1 с), чтобы на медленной машине тест не стал нестабильным; он всё равно заканчивается примерно через 10 мс.

</details>

### Упражнение 3 — Защёлка с обратным отсчётом (сложно)

Напишите `src/bin/latch.rs` с типом `Latch`: `new(n)`, `count_down(&self)` (уменьшает счётчик, но не ниже нуля) и `wait(&self)` (блокирует, пока счётчик не станет нулём). В `main` три рабочих потока печатают «worker N is ready» и делают обратный отсчёт; главный поток ждёт, а потом печатает «all workers are ready, main goes on». Напишите тест: 4 потока увеличивают `AtomicUsize` и делают обратный отсчёт, и после `wait()` атомарное значение равно 4; и тест, что `Latch::new(0).wait()` возвращается сразу.

<details>
<summary>Подсказка</summary>

Состояние — `Mutex<usize>` плюс одна `Condvar`. Кого нужно будить, когда счётчик дошёл до нуля: один поток или все? В `wait` используйте `wait_while` и сохраните возвращённый guard в переменной `_guard` (см. частую ошибку 5).

</details>

<details>
<summary>Ответ</summary>

```rust
use std::sync::{Condvar, Mutex};
use std::thread;

/// Lets threads wait until `count_down` was called `n` times.
struct Latch {
    remaining: Mutex<usize>,
    zero: Condvar,
}

impl Latch {
    fn new(n: usize) -> Self {
        Latch {
            remaining: Mutex::new(n),
            zero: Condvar::new(),
        }
    }

    fn count_down(&self) {
        let mut remaining = self.remaining.lock().unwrap();
        if *remaining > 0 {
            *remaining -= 1;
            if *remaining == 0 {
                self.zero.notify_all();
            }
        }
    }

    fn wait(&self) {
        let remaining = self.remaining.lock().unwrap();
        let _guard = self.zero.wait_while(remaining, |r| *r > 0).unwrap();
    }
}

fn main() {
    let latch = Latch::new(3);
    thread::scope(|s| {
        for id in 0..3 {
            let latch = &latch;
            s.spawn(move || {
                println!("worker {id} is ready");
                latch.count_down();
            });
        }
        latch.wait();
        println!("all workers are ready, main goes on");
    });
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::atomic::{AtomicUsize, Ordering};

    #[test]
    fn wait_returns_only_after_all_count_downs() {
        let latch = Latch::new(4);
        let done = AtomicUsize::new(0);
        thread::scope(|s| {
            for _ in 0..4 {
                s.spawn(|| {
                    done.fetch_add(1, Ordering::Relaxed);
                    latch.count_down();
                });
            }
            latch.wait();
            assert_eq!(done.load(Ordering::Relaxed), 4);
        });
    }

    #[test]
    fn a_latch_of_zero_does_not_block() {
        Latch::new(0).wait();
    }
}
```

```text
worker 0 is ready
worker 2 is ready
worker 1 is ready
all workers are ready, main goes on
```

(Порядок строк рабочих потоков меняется от запуска к запуску; последняя строка всегда последняя.) `cargo test --bin latch` показывает оба теста `ok`.

Почему это работает: счётчик — это условие, и он живёт в мьютексе (правило 1). `count_down` меняет его под блокировкой и уведомляет (правило 2). Используется `notify_all`, потому что, когда счётчик стал нулём, идти дальше может *каждый* ожидающий поток. `wait_while` сначала проверяет условие, поэтому `Latch::new(0).wait()` никогда не засыпает, а поздний `wait()` после последнего `count_down` не пропускает уведомление (правило 3). Атомарный счётчик в тесте читается после `wait()`; мьютекс внутри защёлки гарантирует, что главный поток видит все четыре увеличения (Урок 14.6 объясняет почему).

</details>

## Вопросы для самопроверки

1. Что плохого в том, чтобы ждать флаг, блокируя и проверяя его в цикле?

<details>
<summary>Ответ</summary>

Это активное ожидание: одно ядро работает на 100 %, миллионы раз в секунду не делая ничего полезного, а цикл ещё и борется за блокировку с потоком, который должен поставить флаг. Пауза в цикле экономит CPU, но делает реакцию запоздалой.

</details>

2. Что делает `cvar.wait(guard)`, шаг за шагом?

<details>
<summary>Ответ</summary>

Он принимает guard, отпускает мьютекс и усыпляет поток — всё одним шагом. Когда поток будят (уведомлением или ложно), он снова блокирует мьютекс и возвращает guard.

</details>

3. Почему ожидающий поток должен проверять условие в цикле?

<details>
<summary>Ответ</summary>

Из-за ложных пробуждений (`wait` может вернуться без уведомления) и потому, что другой поток мог снова изменить состояние между уведомлением и моментом, когда этот поток получил блокировку. После каждого пробуждения условие нужно проверить снова.

</details>

4. Поток вызывает `notify_one()` раньше, чем какой-либо поток вызвал `wait()`. Что происходит с уведомлением?

<details>
<summary>Ответ</summary>

Оно теряется: условная переменная не помнит уведомлений. Это безвредно, только если состояние хранится в мьютексе и ожидающий проверяет его перед ожиданием — тогда он видит, что условие уже истинно, и не засыпает.

</details>

5. Почему блокирующая очередь использует две условные переменные?

<details>
<summary>Ответ</summary>

Причин ждать две: потребители ждут «не пусто», а производители — «не полно». С отдельными условными переменными `push` может разбудить именно потребителя, а `pop` — именно производителя. С одной условной переменной `notify_one` мог бы разбудить поток не того вида, который снова уснёт, а нужный поток так и продолжит спать.

</details>

## Итоги

- Не ждите, проверяя в цикле. Условная переменная позволяет потоку спать, пока другой поток не изменит состояние.
- `wait` одним шагом отпускает мьютекс и засыпает, а перед возвратом снова блокирует. `notify_one` будит одного ожидающего, `notify_all` — всех.
- Три правила: условие живёт в мьютексе; меняйте его под блокировкой, затем уведомляйте; всегда ждите в цикле `while` или через `wait_while`.
- Уведомления не сохраняются (потерянное пробуждение), а `wait` может вернуться без уведомления (ложное пробуждение). Правила защищают от обоих.
- Ограниченная блокирующая очередь использует один мьютекс и две условные переменные (`not_empty`, `not_full`). Её ёмкость создаёт обратное давление; флаг `closed` вместе с `notify_all` аккуратно останавливает всех.
- `wait_timeout_while` ждёт с ограничением по времени и сам учитывает ложные пробуждения и оставшееся время.

## Источники и что почитать

- [std::sync::Condvar](https://doc.rust-lang.org/std/sync/struct.Condvar.html) — `wait`, `wait_while`, `wait_timeout_while`, `notify_one`, `notify_all` с официальным примером.
- [std::sync::Barrier](https://doc.rust-lang.org/std/sync/struct.Barrier.html) — готовый инструмент, когда N потоков должны ждать друг друга.
- [Rust Atomics and Locks, chapter 1 — Basics of Rust Concurrency](https://marabos.nl/atomics/basics.html) — Mara Bos объясняет парковку потоков и условные переменные на примере очереди.
- [Operating Systems: Three Easy Pieces — Condition Variables](https://pages.cs.wisc.edu/~remzi/OSTEP/threads-cv.pdf) — глава бесплатного учебника о производителях, потребителях и о том, зачем нужен цикл `while`.

---

[🏠 Начало](../START-HERE.md) · [← 14.2 Мьютексы и блокировки чтения-записи подробно](02-mutexes-and-rwlocks-in-depth.md) · [📋 Этап 14](README.md) · [14.4 Пулы потоков, scoped-потоки и rayon →](04-thread-pools-scoped-threads-and-rayon.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
