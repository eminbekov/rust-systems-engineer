<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 14.7 Пишем спин-блокировку](07-building-a-spin-lock.md) · [📋 Этап 14](README.md) · [14.9 Тестирование конкурентного кода с loom →](09-testing-concurrent-code-with-loom.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/14-concurrency-and-async/08-lock-free-data-structures.md) <!-- nav -->

# Урок 14.8 — Неблокирующие структуры данных

> **Этап 14 — Конкурентность и асинхронность подробно** · Урок 8 из 16 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что на самом деле обещают «неблокирующий» (lock-free) и «без ожидания» (wait-free) и чем это отличается от «не использует `Mutex`».
- Как построить кольцевой буфер для одного производителя и одного потребителя на двух атомарных индексах и release/acquire.
- Как система типов может гарантировать «только один производитель и один потребитель».
- Две трудные задачи неблокирующих структур с указателями: освобождение памяти и ABA (значение меняется с A на B и обратно на A).
- Почему неблокирующее — не значит автоматически быстрое и когда вместо своего кода стоит взять `crossbeam`.

## Перед началом

- Вы прошли [Урок 14.7 — Пишем спин-блокировку](07-building-a-spin-lock.md). Вы знаете `UnsafeCell`, `unsafe impl Sync`, комментарии `SAFETY:`, Miri и `src/lib.rs` проекта этапа.
- Вы знаете циклы сравнения с обменом и ложное разделение из [Урока 14.5](05-atomics.md), release/acquire из [Урока 14.6](06-memory-ordering.md), а сырые указатели и `Box::into_raw` — из [Урока 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md).
- Вы знаете связные списки из [Урока 07.4](../07-algorithms-and-data-structures/04-linked-lists.md) и очереди из [Урока 07.5](../07-algorithms-and-data-structures/05-stacks-and-queues.md).
- В `~/projects/rust/code/p14-concurrency` в `Cargo.toml` уже есть `crossbeam = "0.8"` из Урока 14.4.

## Новые слова

| Слово | Значение |
|---|---|
| **блокирующий** (blocking, об алгоритме) | Если один поток остановлен посередине (например, держа блокировку), другие потоки тоже могут остановиться навсегда. |
| **неблокирующий** (lock-free) | В любой момент **какой-то** поток продвигается, даже если другие потоки остановлены в любом месте. Цикл CAS (compare-and-swap, сравнение с обменом) неблокирующий: если ваш CAS не сработал, значит, сработал CAS другого потока. |
| **без ожидания** (wait-free) | **Каждый** поток завершает свою операцию за ограниченное число собственных шагов, что бы ни делали остальные. |
| **SPSC** (single-producer, single-consumer, один производитель, один потребитель) | Очередь, в которую добавляет ровно один поток и из которой достаёт ровно один поток. |
| **кольцевой буфер** (ring buffer) | Массив фиксированного размера, используемый как очередь: индексы идут по кругу, `index % capacity`. |
| **освобождение памяти** (memory reclamation) | Освобождение памяти, которую другие потоки ещё могут читать. Самая трудная часть неблокирующих структур с указателями. |
| **использование после освобождения** (use-after-free) | Чтение или запись памяти, которая уже освобождена. Неопределённое поведение. |
| **проблема ABA** (ABA problem) | CAS видит значение A, не замечает, что оно успело смениться на B и обратно на A, и срабатывает, хотя должен был не сработать. |
| **стек Трайбера** (Treiber stack) | Классический неблокирующий стек: связный список, указатель `head` которого меняется через CAS. |
| **`AtomicPtr<T>`** | Атомарный сырой указатель `*mut T` с методами `load`, `store`, `swap` и `compare_exchange`. |
| **освобождение по эпохам** (epoch-based reclamation) | Способ безопасно освобождать память в неблокирующих структурах: память освобождается только после того, как каждый поток покинул «эпоху», в которой мог её видеть. Это реализует `crossbeam-epoch`. |

## 1. Что значит «неблокирующий»

«Неблокирующий» звучит как «не использует блокировок», но это обещание о **продвижении**. Представьте, что OS (Operating System, операционная система) останавливает поток в самый неудачный момент и не запускает его снова целую секунду:

| Вид | Если один поток остановлен в неудачный момент… | Пример |
|---|---|---|
| **блокирующий** | другие могут ждать его вечно | `Mutex`: остановленный поток держит блокировку |
| **неблокирующий** | другие всё равно завершают свои операции | цикл CAS: неудачный CAS означает, что *кто-то другой* преуспел |
| **без ожидания** | каждый завершает за ограниченное число шагов | `fetch_add` (одна `ldadd` на вашем Mac); кольцевой буфер из раздела 2 |

Спин-блокировка из Урока 14.7 не использует `Mutex`, но она **не** неблокирующая: если владелец остановлен, каждый ожидающий ждёт. С другой стороны, цикл CAS может повторяться много раз, поэтому он неблокирующий, но не «без ожидания».

Зачем это нужно? Неблокирующий код не может попасть во взаимную блокировку, его безопасно использовать там, где поток могут прервать (например, внутри обработчика сигнала, Урок 13.7), и он избегает плохого случая, когда вытесненный владелец блокировки останавливает всех. Цена: написать его правильно гораздо труднее, и, как покажет раздел 4, он не становится автоматически быстрее.

## 2. Неблокирующий кольцевой буфер SPSC

Простейшая полезная неблокирующая структура — очередь с **одним** производителем и **одним** потребителем. Представьте круглый стол с фиксированным числом тарелок. Повар кладёт еду на следующую тарелку; гость ест с самой старой полной тарелки. Хватает двух чисел:

```text
 capacity = 4            head = 5 (next to read)      tail = 7 (next to write)
 slot:      0      1      2      3
          [ 4 ]  [ 5 ]  [ 6 ]  [ 3 ]
                   ▲             ▲
                head%4=1      tail%4=3
 items 5 and 6 are in the ring (tail - head = 2); slots 3 and 0 are free.
```

- `tail` меняет **только производитель**; `head` меняет **только потребитель**. Индексы только растут; ячейка — это `index % capacity`.
- Пусто: `head == tail`. Полно: `tail - head == capacity`.
- Производитель записывает ячейку и **потом** публикует её, сдвигая `tail` с `Release`. Потребитель читает `tail` с `Acquire` и только после этого читает ячейку. Это приём «данные плюс флаг» из Урока 14.6, где флаг — `tail`.
- В обратную сторону потребитель публикует «я закончил с этой ячейкой», сдвигая `head` с `Release`, а производитель читает `head` с `Acquire`, прежде чем снова использовать ячейку.

Поскольку у каждого индекса один писатель, CAS не нужен: работу делают обычные атомарные чтения и записи. Каждый `push` и `pop` завершается за несколько шагов, поэтому этот кольцевой буфер даже **без ожидания**.

Чтобы небезопасная часть была маленькой, эта версия хранит значения `T: Copy + Default`: в каждой ячейке всегда лежит допустимое значение (в начале её заполняют `T::default()`), а копирование значения наружу не оставляет ничего, что нужно уничтожать. (Настоящие кольцевые буферы для любого `T` используют `MaybeUninit<T>` и должны аккуратно уничтожать оставшиеся элементы; `ArrayQueue` из `crossbeam` так и делает.)

Создайте `src/spsc.rs` и добавьте `pub mod spsc;` в `src/lib.rs`:

```rust
//! A bounded single-producer, single-consumer (SPSC) ring buffer without locks.

use std::cell::UnsafeCell;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};

struct Ring<T> {
    slots: Box<[UnsafeCell<T>]>,
    head: AtomicUsize, // the next slot to read; only the consumer changes it
    tail: AtomicUsize, // the next slot to write; only the producer changes it
}

// SAFETY: the producer writes only slots that the consumer cannot read yet
// (between head and tail they are "free"), and publishes them with a Release
// store of `tail`; the consumer reads only slots it acquired from `tail`.
unsafe impl<T: Send> Sync for Ring<T> {}

pub struct Producer<T> {
    ring: Arc<Ring<T>>,
}

pub struct Consumer<T> {
    ring: Arc<Ring<T>>,
}

/// Creates a ring buffer that holds up to `capacity` items.
pub fn channel<T: Copy + Default>(capacity: usize) -> (Producer<T>, Consumer<T>) {
    assert!(capacity > 0, "capacity must be at least 1");
    let slots = (0..capacity).map(|_| UnsafeCell::new(T::default())).collect();
    let ring = Arc::new(Ring {
        slots,
        head: AtomicUsize::new(0),
        tail: AtomicUsize::new(0),
    });
    (
        Producer {
            ring: Arc::clone(&ring),
        },
        Consumer { ring },
    )
}

impl<T: Copy> Producer<T> {
    /// Adds `item`, or gives it back when the ring is full.
    pub fn push(&mut self, item: T) -> Result<(), T> {
        let ring = &*self.ring;
        let tail = ring.tail.load(Ordering::Relaxed); // only we write it
        // Acquire: pairs with the Release store of `head` in pop, so the
        // consumer has finished reading the slot we are about to reuse.
        let head = ring.head.load(Ordering::Acquire);
        if tail - head == ring.slots.len() {
            return Err(item); // full
        }
        let slot = &ring.slots[tail % ring.slots.len()];
        // SAFETY: this slot is free: the consumer reads only below `tail`.
        unsafe { *slot.get() = item };
        // Release: publishes the slot we just wrote to the consumer.
        ring.tail.store(tail + 1, Ordering::Release);
        Ok(())
    }
}

impl<T: Copy> Consumer<T> {
    /// Takes the oldest item, or `None` when the ring is empty.
    pub fn pop(&mut self) -> Option<T> {
        let ring = &*self.ring;
        let head = ring.head.load(Ordering::Relaxed); // only we write it
        // Acquire: pairs with the Release store of `tail` in push.
        let tail = ring.tail.load(Ordering::Acquire);
        if head == tail {
            return None; // empty
        }
        let slot = &ring.slots[head % ring.slots.len()];
        // SAFETY: the producer published this slot and will not touch it
        // until we move `head` past it.
        let item = unsafe { *slot.get() };
        // Release: tells the producer that the slot may be reused.
        ring.head.store(head + 1, Ordering::Release);
        Some(item)
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn full_and_empty() {
        let (mut tx, mut rx) = channel::<u32>(2);
        assert_eq!(rx.pop(), None);
        assert_eq!(tx.push(1), Ok(()));
        assert_eq!(tx.push(2), Ok(()));
        assert_eq!(tx.push(3), Err(3)); // full
        assert_eq!(rx.pop(), Some(1));
        assert_eq!(tx.push(3), Ok(()));
        assert_eq!(rx.pop(), Some(2));
        assert_eq!(rx.pop(), Some(3));
        assert_eq!(rx.pop(), None);
    }

    #[test]
    fn keeps_order_across_threads() {
        const N: u64 = 1000;
        let (mut tx, mut rx) = channel::<u64>(8);
        let producer = thread::spawn(move || {
            for i in 0..N {
                while tx.push(i).is_err() {
                    std::hint::spin_loop(); // full: wait for the consumer
                }
            }
        });
        let mut expected = 0;
        while expected < N {
            if let Some(x) = rx.pop() {
                assert_eq!(x, expected, "items must come out in order");
                expected += 1;
            }
        }
        producer.join().unwrap();
    }
}
```

**Что происходит:**

1. `Ring<T>` хранит ячейки (`UnsafeCell`, потому что оба потока трогают их через общий `Arc`) и два индекса. Обещание `unsafe impl Sync` объяснено в его комментарии `SAFETY:`: два потока никогда не трогают одну ячейку одновременно, а каждая передача идёт через пару release/acquire.
2. `channel` создаёт **один** `Producer` и **один** `Consumer`, которые делят кольцо. Ни один из типов не реализует `Clone`, а `push`/`pop` принимают `&mut self`. Поэтому сам компилятор гарантирует «один производитель, один потребитель»: второго производителя получить нельзя, и вызвать `push` из двух потоков на одном производителе тоже нельзя. Именно это делает простые записи `tail` и `head` корректными.
3. `push` читает свой `tail` с `Relaxed` (никто другой его не пишет), а `head` — с `Acquire`. Если кольцо полно, он возвращает элемент в `Err` вместо ожидания: что делать (крутиться, уступить, выбросить элемент), решает вызывающий код. Это снова обратное давление.
4. `push` записывает ячейку, а затем записывает `tail + 1` с `Release`; `pop` читает `tail` с `Acquire`, читает ячейку и записывает `head + 1` с `Release`. Ни одна сторона ничего не ждёт и ничего не повторяет: без ожидания.
5. Оба дескриптора — `Send` (`Arc<Ring<T>>` является `Send`, потому что `Ring<T>` — `Send + Sync`), поэтому производитель можно переместить в другой поток, как делает второй тест.

Запустите тесты через `cargo test --lib` и под Miri через `cargo +nightly miri test --lib` (Урок 14.7):

```text
test spsc::tests::full_and_empty ... ok
test spsc::tests::keeps_order_across_threads ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Miri тоже сообщает `test result: ok. 2 passed`: в этих запусках нет ни гонки данных, ни неопределённого поведения. Небольшая программа, которая им пользуется, — `src/bin/spsc_demo.rs`:

```rust
use p14_concurrency::spsc;
use std::thread;

fn main() {
    let (mut tx, mut rx) = spsc::channel::<u64>(4);
    let producer = thread::spawn(move || {
        for i in 1..=10 {
            while tx.push(i).is_err() {
                thread::yield_now(); // full: let the consumer run
            }
        }
    });
    let mut got = Vec::new();
    while got.len() < 10 {
        match rx.pop() {
            Some(x) => got.push(x),
            None => thread::yield_now(), // empty: let the producer run
        }
    }
    producer.join().unwrap();
    println!("received {got:?}");
}
```

```text
received [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

`thread::yield_now()` говорит OS: «мне сейчас нечего делать, запусти кого-нибудь другого». Это чуть вежливее чистого вращения, когда другая сторона может не выполняться.

## 3. Указатели всё усложняют: стек Трайбера

У кольцевого буфера был фиксированный массив и два индекса с одним писателем у каждого. Большинство неблокирующих структур амбициознее: добавляют и достают **многие** потоки, а структура растёт за счёт узлов в куче. Классический пример — **стек Трайбера**: связный список, у которого `head` — это `AtomicPtr<Node<T>>`.

**Push** прост и безопасен: создать узел, направить его `next` на текущую вершину и через CAS сменить вершину со «старой вершины» на «мой узел». Если между делом добавил другой поток, CAS не срабатывает, и мы повторяем (цикл CAS, Урок 14.5). Его пишет Упражнение 3.

**Pop** выглядит так же просто: прочитать `head`, прочитать `head.next`, через CAS сменить вершину с `head` на `next` и вернуть значение узла. Но в нём прячутся две проблемы:

```text
 1. Use-after-free (memory reclamation)
    thread A: reads head = N1, is about to read N1.next ...      (A is paused)
    thread B: pops N1 and FREES it
    thread A: reads N1.next  ← reads freed memory: undefined behaviour

 2. ABA
    thread A: reads head = N1, next = N2, is about to CAS(N1 → N2)   (A is paused)
    thread B: pops N1, pops N2 (frees N2), pushes N1 again (same address!)
    stack is now: N1 → N3
    thread A: CAS(N1 → N2) SUCCEEDS, because head is N1 again
    head is now N2, a freed node. The stack is corrupted.
```

Проблема ABA называется так потому, что значение прошло путь от A к B и обратно к A, а CAS не может этого отличить. Обе проблемы растут из одного вопроса: **когда безопасно освободить узел, на который ещё могут смотреть другие потоки?** Сборщик мусора ответил бы на него за вас; в Rust его нет, поэтому неблокирующим структурам нужна схема **освобождения памяти**:

- **Освобождение по эпохам** (`crossbeam-epoch`): потоки «закрепляют» текущую эпоху, пока работают; удалённый узел освобождается только после того, как каждый поток ушёл дальше эпохи, в которой узел был удалён.
- **Указатели опасности** (hazard pointers): каждый поток публикует указатели, которые читает прямо сейчас; никто не освобождает узел, который есть в этом списке.
- **Обойти вопрос**: спроектировать API (Application Programming Interface, программный интерфейс) так, чтобы узел освобождал только поток, владеющий им исключительно. Упражнение 3 делает так стек с `push` и `take_all` (который забирает весь список разом), но без одиночного `pop`.

> 🔒 **Безопасность:** ошибки использования после освобождения — одни из самых опасных ошибок памяти, потому что атакующий иногда может управлять тем, что запишется в освобождённую память. В неблокирующем коде они прячутся за редкими таймингами. Если вы пишете неблокирующую структуру с `unsafe`-указателями, запускайте каждый тест под Miri, который сообщает об использовании после освобождения, гонках данных и утечках памяти.

## 4. Быстрее ли неблокирующее? Измерьте

Сравним три очереди с одним производителем и одним потребителем, которые передают 5 миллионов чисел: `Mutex<VecDeque>`, `crossbeam::queue::ArrayQueue` (неблокирующая ограниченная очередь, допускающая много производителей и много потребителей) и наш кольцевой буфер. Создайте `src/bin/queue_race.rs`:

```rust
use crossbeam::queue::ArrayQueue;
use p14_concurrency::spsc;
use std::collections::VecDeque;
use std::hint::spin_loop;
use std::sync::Mutex;
use std::thread;
use std::time::Instant;

const N: u64 = 5_000_000;

fn main() {
    // 1. A VecDeque behind a Mutex.
    let queue = Mutex::new(VecDeque::with_capacity(1024));
    let start = Instant::now();
    thread::scope(|s| {
        s.spawn(|| {
            for i in 0..N {
                loop {
                    let mut q = queue.lock().unwrap();
                    if q.len() < 1024 {
                        q.push_back(i);
                        break;
                    }
                }
            }
        });
        let mut sum = 0;
        let mut got = 0;
        while got < N {
            if let Some(x) = queue.lock().unwrap().pop_front() {
                sum += x;
                got += 1;
            }
        }
        assert_eq!(sum, N * (N - 1) / 2);
    });
    println!("Mutex<VecDeque> : {:.1?}", start.elapsed());

    // 2. crossbeam's lock-free ArrayQueue (many producers and consumers allowed).
    let queue = ArrayQueue::new(1024);
    let start = Instant::now();
    thread::scope(|s| {
        s.spawn(|| {
            for i in 0..N {
                while queue.push(i).is_err() {
                    spin_loop();
                }
            }
        });
        let mut sum = 0;
        let mut got = 0;
        while got < N {
            if let Some(x) = queue.pop() {
                sum += x;
                got += 1;
            }
        }
        assert_eq!(sum, N * (N - 1) / 2);
    });
    println!("ArrayQueue      : {:.1?}", start.elapsed());

    // 3. Our SPSC ring buffer.
    let (mut tx, mut rx) = spsc::channel::<u64>(1024);
    let start = Instant::now();
    thread::scope(|s| {
        s.spawn(move || {
            for i in 0..N {
                while tx.push(i).is_err() {
                    spin_loop();
                }
            }
        });
        let mut sum = 0;
        let mut got = 0;
        while got < N {
            if let Some(x) = rx.pop() {
                sum += x;
                got += 1;
            }
        }
        assert_eq!(sum, N * (N - 1) / 2);
    });
    println!("our SPSC ring   : {:.1?}", start.elapsed());
}
```

`cargo run --release --bin queue_race`, два запуска:

```text
Mutex<VecDeque> : 254.3ms
ArrayQueue      : 41.8ms
our SPSC ring   : 254.6ms
Mutex<VecDeque> : 416.2ms
ArrayQueue      : 33.0ms
our SPSC ring   : 345.7ms
```

**Что происходит:**

1. Каждая версия проверяет, что сумма всего полученного правильная (`N * (N - 1) / 2`), так что все они действительно передают все 5 миллионов чисел.
2. `ArrayQueue` примерно в 6–12 раз быстрее мьютекса.
3. Наш кольцевой буфер, простой и без ожидания, **так же медленен, как мьютекс**. Неблокирующее — не значит быстрое!
4. Почему? Два эффекта кэша из Урока 14.5. Во-первых, `head` и `tail` лежат рядом в памяти, в **одной кэш-линии**: каждая запись `tail` производителем отбирает линию у потребителя, а каждая запись `head` возвращает её обратно (ложное разделение). Во-вторых, каждый `push` читает `head`, а каждый `pop` — `tail`, даже когда место или данные заведомо есть, поэтому каждой операции нужна самая свежая кэш-линия другого ядра.
5. `ArrayQueue` избегает обоих эффектов: она разносит свои индексы по разным кэш-линиям (`CachePadded`) и гораздо реже трогает данные другой стороны. В Упражнении 2 вы примените эти два приёма; на тестовой машине это ускорило наш кольцевой буфер примерно в 10 раз, чуть быстрее `ArrayQueue`.

> ⚡ **Производительность:** в конкурентном коде цена складывается в основном из **перемещений кэш-линий** между ядрами, а не из числа инструкций. Посчитайте, как часто каждая операция пишет в линию, которую читает другое ядро. Неблокирующая схема, которая пишет в общие линии при каждой операции, может проиграть простой блокировке.

> 💡 **Совет:** в настоящих программах используйте проверенные крейты: `crossbeam::queue::ArrayQueue` (ограниченная) и `SegQueue` (неограниченная) для очередей MPMC (multi-producer, multi-consumer, много производителей, много потребителей), `crossbeam::channel` для каналов с блокирующим `recv` и `crossbeam-epoch`, если вам действительно нужно строить структуру с освобождаемыми узлами. Пишите свой неблокирующий код, чтобы учиться, или когда измерение доказало, что он нужен, — и тогда проверяйте его через Miri и loom.

## Частые ошибки

1. **«Нет `Mutex` — значит, неблокирующий».** Спин-блокировка не использует `Mutex` и всё равно блокирующая. Неблокируемость — это гарантия продвижения.
2. **Два производителя на структуре SPSC.** Простая запись `tail` корректна только с одним писателем. Обеспечивайте это типами (нет `Clone`, `&mut self`), а не комментарием.
3. **Освобождение узла, который ещё может читать другой поток.** Использование после освобождения. Используйте схему освобождения памяти или API, где узлы освобождает только исключительный владелец.
4. **Вера в то, что CAS по указателю обнаружит «ничего не изменилось».** ABA: тот же адрес может вернуться с другим содержимым.
5. **Уверенность, что неблокирующее быстрее.** Измеряйте. Следите за ложным разделением и за чтением индекса другой стороны при каждой операции.

## Попробуйте сами

1. Запустите `cargo test --lib` и `cargo +nightly miri test --lib`. В `spsc.rs` замените запись `tail` с `Release` в `push` на `Relaxed` и снова запустите Miri. Он сообщает `Undefined Behavior: Data race detected` между записью ячейки в `push` и чтением ячейки в `pop`: без release потребитель может прочитать ячейку раньше, чем станет видна запись производителя. Верните как было.
2. Попробуйте нарушить правило «один производитель»: в `spsc_demo.rs` напишите `let tx2 = tx.clone();`. Компилятор отвечает ``error[E0599]: no method named `clone` found for struct `Producer<T>` in the current scope``. Эта ошибка — гарантия одного производителя в действии. Удалите строку.
3. Запустите `queue_race` с `spsc::channel::<u64>(16)` вместо `1024` (крошечное кольцо). Как меняется время? (На тестовой машине наше кольцо заняло около 214–235 мс, чуть быстрее, чем с 1024 ячейками; в этой первой версии цена — в общей кэш-линии, а не в размере.) Потом попробуйте ещё раз после Упражнения 2.

## Упражнения

### Упражнение 1 — Определите гарантию продвижения (легко)

Какая из операций блокирующая, неблокирующая или без ожидания? (1) `AtomicU64::fetch_add` на вашем Mac; (2) цикл `compare_exchange_weak`, который прибавляет 1 с верхним пределом (Урок 14.5); (3) `SpinLock::lock` (Урок 14.7); (4) `Producer::push` кольцевого буфера; (5) `Mutex::lock`.

<details>
<summary>Подсказка</summary>

Для каждой спросите: если другой поток остановлен в самый неудачный момент, может ли эта операция всё равно завершиться? И ограничено ли число шагов?

</details>

<details>
<summary>Ответ</summary>

1. Без ожидания: одна инструкция `ldadd`, всегда выполняется за один шаг.
2. Неблокирующая: она может повторяться, но каждый неудачный CAS означает, что сработал CAS другого потока, поэтому какой-то поток всегда продвигается. Не «без ожидания», потому что один невезучий поток может повторять много раз.
3. Блокирующая: если владелец блокировки остановлен, ожидающие никогда её не получат.
4. Без ожидания: фиксированное число шагов и никаких повторов; когда кольцо полно, она возвращает `Err` вместо ожидания.
5. Блокирующая: остановленный владелец блокировки задерживает всех, кто её ждёт.

</details>

### Упражнение 2 — Сделайте кольцевой буфер быстрым (средне)

Внесите два изменения в `src/spsc.rs` и измерьте через `queue_race` до и после: (1) оберните `head` и `tail` в `crossbeam::utils::CachePadded`, чтобы они жили в разных кэш-линиях; (2) пусть производитель запоминает последнее прочитанное значение `head` (поле `head_seen`) и снова читает `head`, только когда кольцо *выглядит* полным; пусть потребитель делает то же самое с `tail` (`tail_seen`), когда кольцо *выглядит* пустым. Все тесты должны по-прежнему проходить, и под Miri тоже.

<details>
<summary>Подсказка</summary>

`CachePadded::new(AtomicUsize::new(0))`; `CachePadded<T>` разыменовывается в `T`, поэтому `ring.head.load(…)` остаётся прежним. Запомненное значение может быть только *слишком старым*, но никогда слишком новым: если производитель думает, что кольцо полно, в нём на самом деле может быть место, поэтому перед отказом он должен проверить снова. Может ли слишком старое `head_seen` заставить производителя перезаписать ячейку, которая ещё нужна потребителю?

</details>

<details>
<summary>Ответ</summary>

Изменённые части `src/spsc.rs`:

```rust
use crossbeam::utils::CachePadded;
use std::cell::UnsafeCell;
use std::sync::Arc;
use std::sync::atomic::{AtomicUsize, Ordering};

struct Ring<T> {
    slots: Box<[UnsafeCell<T>]>,
    head: CachePadded<AtomicUsize>, // the next slot to read; only the consumer changes it
    tail: CachePadded<AtomicUsize>, // the next slot to write; only the producer changes it
}

pub struct Producer<T> {
    ring: Arc<Ring<T>>,
    head_seen: usize, // the last value of `head` we loaded
}

pub struct Consumer<T> {
    ring: Arc<Ring<T>>,
    tail_seen: usize, // the last value of `tail` we loaded
}
```

В `channel` создавайте индексы через `CachePadded::new(AtomicUsize::new(0))`, а дескрипторы — с `head_seen: 0` и `tail_seen: 0`. Новый `push`:

```rust
    pub fn push(&mut self, item: T) -> Result<(), T> {
        let ring = &*self.ring;
        let tail = ring.tail.load(Ordering::Relaxed); // only we write it
        if tail - self.head_seen == ring.slots.len() {
            // Looks full. Acquire: pairs with the Release store of `head` in
            // pop, so the consumer has finished reading the slot we reuse.
            self.head_seen = ring.head.load(Ordering::Acquire);
            if tail - self.head_seen == ring.slots.len() {
                return Err(item); // really full
            }
        }
        let slot = &ring.slots[tail % ring.slots.len()];
        // SAFETY: this slot is free: the consumer reads only below `tail`.
        unsafe { *slot.get() = item };
        // Release: publishes the slot we just wrote to the consumer.
        ring.tail.store(tail + 1, Ordering::Release);
        Ok(())
    }
```

Новое начало `pop`:

```rust
        let head = ring.head.load(Ordering::Relaxed); // only we write it
        if head == self.tail_seen {
            // Looks empty. Acquire: pairs with the Release store of `tail` in push.
            self.tail_seen = ring.tail.load(Ordering::Acquire);
            if head == self.tail_seen {
                return None; // really empty
            }
        }
```

Остальная часть `pop` остаётся прежней. Результаты на тестовой машине (первые две строки не меняются):

```text
Mutex<VecDeque> : 266.6ms
ArrayQueue      : 39.0ms
our SPSC ring   : 13.1ms
Mutex<VecDeque> : 283.6ms
ArrayQueue      : 43.6ms
our SPSC ring   : 28.0ms
```

Только со вторым изменением (без выравнивания) кольцу требовалось около 40 мс; оба изменения вместе дали 13–30 мс вместо примерно 250–350 мс до них. `cargo test --lib` и `cargo +nightly miri test --lib` по-прежнему проходят.

Почему это работает: выравнивание помещает каждый индекс в свою кэш-линию, поэтому записи `tail` производителем больше не отбирают линию, в которой лежит `head`. Запоминание индекса другой стороны значит, что большинство операций трогают только линию своего ядра; линию другой стороны приходится забирать, только когда кольцо выглядит полным или пустым. Запомненное значение — всегда *более старое* значение того же индекса, а индексы только растут, поэтому производитель может решить, что места *меньше*, чем на самом деле, но никогда — что больше: он никогда не перезапишет непрочитанную ячейку. Когда же он использует свежее значение `head`, оно получено чтением `Acquire`, поэтому чтения этих ячеек потребителем произошли-до.

</details>

### Упражнение 3 — Стек только с добавлением и `take_all` (сложно)

Напишите `src/push_stack.rs` (добавьте `pub mod push_stack;` в `src/lib.rs`): неблокирующий стек `PushStack<T>` на `AtomicPtr<Node<T>>` с тремя операциями: `new()`, `push(&self, value)` (push Трайбера с циклом CAS) и `take_all(&self) -> Vec<T>`, который убирает **все** значения разом (сначала самые новые), обменивая вершину на нулевой указатель. Реализуйте `Drop`, чтобы оставшиеся узлы освобождались. Напишите тесты: один поток (добавить 1, 2, 3 → `take_all` даёт `[3, 2, 1]`); четыре потока добавляют по 100 значений, пока пятый 50 раз вызывает `take_all`, и в конце каждое значение из 0..400 забрано ровно один раз; и тест, который оставляет на стеке `String` в момент его уничтожения. Запустите всё под Miri.

<details>
<summary>Подсказка</summary>

`Box::into_raw(Box::new(node))` даёт `*mut Node<T>`; `Box::from_raw(ptr)` забирает владение обратно (и освобождает память при уничтожении). В `push` перед каждым CAS внутри цикла ставьте `(*node).next = head`. Почему `take_all` избегает и использования после освобождения, и ABA? (Кто ещё может добраться до узлов после `swap`?)

</details>

<details>
<summary>Ответ</summary>

```rust
//! A lock-free stack that supports `push` and `take_all`, but no single `pop`.

use std::ptr;
use std::sync::atomic::{AtomicPtr, Ordering};

struct Node<T> {
    value: T,
    next: *mut Node<T>,
}

pub struct PushStack<T> {
    head: AtomicPtr<Node<T>>,
}

// SAFETY: nodes are owned by the stack; values are moved in by `push` and
// out by `take_all`, so the stack may be sent or shared when T may be sent.
unsafe impl<T: Send> Send for PushStack<T> {}
unsafe impl<T: Send> Sync for PushStack<T> {}

impl<T> PushStack<T> {
    pub const fn new() -> Self {
        PushStack {
            head: AtomicPtr::new(ptr::null_mut()),
        }
    }

    pub fn push(&self, value: T) {
        let node = Box::into_raw(Box::new(Node {
            value,
            next: ptr::null_mut(),
        }));
        let mut head = self.head.load(Ordering::Relaxed);
        loop {
            // SAFETY: `node` is ours alone until the CAS below publishes it.
            unsafe { (*node).next = head };
            // Release: publishes the node's contents to `take_all`.
            match self
                .head
                .compare_exchange_weak(head, node, Ordering::Release, Ordering::Relaxed)
            {
                Ok(_) => return,
                Err(actual) => head = actual, // another push won: retry
            }
        }
    }

    /// Takes every value pushed so far, newest first.
    pub fn take_all(&self) -> Vec<T> {
        // Acquire: pairs with the Release CAS in push.
        let mut node = self.head.swap(ptr::null_mut(), Ordering::Acquire);
        let mut values = Vec::new();
        while !node.is_null() {
            // SAFETY: the swap made this list ours alone; no other thread can
            // reach these nodes any more, so we may free them.
            let boxed = unsafe { Box::from_raw(node) };
            node = boxed.next;
            values.push(boxed.value);
        }
        values
    }
}

impl<T> Default for PushStack<T> {
    fn default() -> Self {
        Self::new()
    }
}

impl<T> Drop for PushStack<T> {
    fn drop(&mut self) {
        self.take_all(); // free the nodes that are still on the stack
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn newest_first() {
        let stack = PushStack::new();
        stack.push(1);
        stack.push(2);
        stack.push(3);
        assert_eq!(stack.take_all(), vec![3, 2, 1]);
        assert!(stack.take_all().is_empty());
    }

    #[test]
    fn many_threads_lose_nothing() {
        let stack = PushStack::new();
        let mut taken = Vec::new();
        thread::scope(|s| {
            for t in 0..4 {
                let stack = &stack;
                s.spawn(move || {
                    for i in 0..100 {
                        stack.push(t * 100 + i);
                    }
                });
            }
            s.spawn(|| {
                for _ in 0..50 {
                    taken.extend(stack.take_all()); // take while others push
                }
            });
        });
        taken.extend(stack.take_all());
        taken.sort();
        assert_eq!(taken, (0..400).collect::<Vec<_>>());
    }

    #[test]
    fn drop_frees_remaining_nodes() {
        let stack = PushStack::new();
        stack.push(String::from("left on the stack"));
        // dropping `stack` must free the node and the String (Miri checks it)
    }
}
```

`cargo test --lib` и `cargo +nightly miri test --lib` проходят все пять тестов библиотеки этапа (два для `spsc`, три для `push_stack`). Без реализации `Drop` Miri ловит утечку:

```text
error: memory leaked: alloc42350 (Rust heap, size: 32, align: 8), allocated here:
  --> src/push_stack.rs:28:34
   |
28 |           let node = Box::into_raw(Box::new(Node {
```

Почему это работает: `push` — это push Трайбера: новый узел указывает на увиденную нами вершину, и CAS устанавливает его, только если вершина всё ещё та же; иначе мы повторяем с новой вершиной. CAS с `Release` публикует содержимое узла. `take_all` заменяет вершину нулём **одним** `swap`, поэтому весь список разом становится нашим: ни один другой поток больше не может добраться до этих узлов, и возможны только добавления *новых* узлов. Поэтому никто не может прочитать узел, который мы освобождаем (нет использования после освобождения), и никакой CAS не сравнит с узлом, который удалили и использовали снова (нет ABA). `swap` с `Acquire` делает видимым для нас содержимое каждого добавленного узла. `Drop` освобождает то, что осталось, а Miri подтверждает, что ничего не утекает.

Зачем обе строки `unsafe impl`? `AtomicPtr<T>` реализует `Send` и `Sync` для **любого** `T`, поэтому без них компилятор сам сделал бы `PushStack<Rc<i32>>` типом `Send`: можно было бы добавить значения `Rc` в одном потоке и уничтожить их в другом — гонка данных на счётчике ссылок. Две реализации с ограничением `T: Send` заменяют автоматические; с ними отправка `PushStack<Rc<i32>>` в другой поток не компилируется: ``error[E0277]: `Rc<{integer}>` cannot be sent between threads safely``.

</details>

## Вопросы для самопроверки

1. Что обещает «неблокирующий» и почему спин-блокировка не неблокирующая?

<details>
<summary>Ответ</summary>

Что какой-то поток всегда продвигается, даже если другие потоки остановлены в любом месте. Спин-блокировка не неблокирующая: если поток, который её держит, остановлен, все остальные потоки бесконечно крутятся без продвижения.

</details>

2. Почему кольцевому буферу SPSC не нужно сравнение с обменом?

<details>
<summary>Ответ</summary>

У каждого индекса ровно один писатель: `tail` меняет только производитель, `head` — только потребитель. Единственный писатель может использовать простые атомарные записи; release/acquire на индексах безопасно передаёт ячейки.

</details>

3. Как код гарантирует, что производитель и потребитель только по одному?

<details>
<summary>Ответ</summary>

`channel` создаёт ровно один `Producer` и один `Consumer`, ни один не реализует `Clone`, а `push`/`pop` принимают `&mut self`, поэтому даже одним дескриптором не могут одновременно пользоваться два потока. Это обеспечивает компилятор.

</details>

4. Опишите проблему ABA своими словами.

<details>
<summary>Ответ</summary>

Поток читает значение A и позже делает CAS, ожидая A. Тем временем другие потоки поменяли его на B и обратно на A (например, адрес освобождённого узла использовали снова). CAS срабатывает, потому что сравнивает только значение, но ситуация уже не та, что видел поток, и структура портится.

</details>

5. Почему первая версия кольцевого буфера была так же медленна, как мьютекс?

<details>
<summary>Ответ</summary>

Из-за перемещений кэш-линий: `head` и `tail` делили одну кэш-линию (ложное разделение), а каждая операция читала индекс другой стороны. Поэтому каждый push и pop перегонял кэш-линию между двумя ядрами.

</details>

6. Почему `take_all` избегает проблемы освобождения памяти, которая есть у одиночного `pop`?

<details>
<summary>Ответ</summary>

Он убирает весь список одним атомарным `swap`. После этого ни один другой поток не может добраться ни до одного из этих узлов, поэтому забравший их поток владеет ими исключительно и может безопасно их освободить. Одиночный `pop` должен читать `head.next`, пока другие потоки могут достать и освободить тот же узел.

</details>

## Итоги

- Неблокирующий значит: какой-то поток всегда продвигается; без ожидания значит: каждый поток завершает за ограниченное число шагов. Одно лишь «нет `Mutex`» не означает ни того, ни другого.
- Кольцевому буферу SPSC нужны только два индекса с одним писателем у каждого и передачи через release/acquire. Система типов (нет `Clone`, `&mut self`) гарантирует одного производителя и одного потребителя.
- Неблокирующие структуры с узлами в куче сталкиваются с освобождением памяти (использование после освобождения) и ABA. Решения: освобождение по эпохам, указатели опасности или API, где узлы освобождает только исключительный владелец.
- Неблокирующее — не значит автоматически быстрое. Ложное разделение и обращение к кэш-линии другой стороны при каждой операции сделали наше первое кольцо таким же медленным, как мьютекс; выравнивание и запомненные индексы ускорили его примерно в 10 раз.
- В настоящих программах используйте `crossbeam` (`ArrayQueue`, `SegQueue`, `channel`, `epoch`), а любой неблокирующий код с `unsafe` проверяйте через Miri и loom.

## Источники и что почитать

- [Rust Atomics and Locks, chapter 5 — Building Our Own Channels](https://marabos.nl/atomics/building-channels.html) — Mara Bos шаг за шагом строит каналы с тем же рассуждением о безопасности, что и наш кольцевой буфер.
- [Rust Atomics and Locks, chapter 6 — Building Our Own "Arc"](https://marabos.nl/atomics/building-arc.html) — неблокирующий счётчик ссылок и вопрос, когда можно освобождать память.
- [crossbeam::queue на docs.rs](https://docs.rs/crossbeam/latest/crossbeam/queue/index.html) — `ArrayQueue` и `SegQueue`, неблокирующие очереди, которые используют на практике.
- [crossbeam-epoch на docs.rs](https://docs.rs/crossbeam-epoch) — освобождение памяти по эпохам для неблокирующих структур данных.
- [std::sync::atomic::AtomicPtr](https://doc.rust-lang.org/std/sync/atomic/type.AtomicPtr.html) — атомарный указатель, на котором строится стек Трайбера.

---

[🏠 Начало](../START-HERE.md) · [← 14.7 Пишем спин-блокировку](07-building-a-spin-lock.md) · [📋 Этап 14](README.md) · [14.9 Тестирование конкурентного кода с loom →](09-testing-concurrent-code-with-loom.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
