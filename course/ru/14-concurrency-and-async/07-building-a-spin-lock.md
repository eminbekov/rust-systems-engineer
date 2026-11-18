<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 14.6 Порядок памяти](06-memory-ordering.md) · [📋 Этап 14](README.md) · [14.8 Неблокирующие структуры данных →](08-lock-free-data-structures.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/14-concurrency-and-async/07-building-a-spin-lock.md) <!-- nav -->

# Урок 14.7 — Пишем спин-блокировку

> **Этап 14 — Конкурентность и асинхронность подробно** · Урок 7 из 16 · ⏱ около 135 минут

## Что вы узнаете

- Как построить блокировку из одного атомарного флага и цикла.
- Как обернуть данные в `UnsafeCell`, почему `unsafe impl Sync` приходится писать самому и как это обосновать.
- Как тип guard-объекта с `Deref`, `DerefMut` и `Drop` делает блокировку безопасной в использовании.
- Почему захват блокировки требует `Acquire`, а освобождение — `Release`: это видно по потерянным обновлениям на вашем Mac и по Miri.
- Почему в обычной программе спин-блокировку почти никогда не стоит использовать — с измерениями против `std::sync::Mutex`.

## Перед началом

- Вы прошли [Урок 14.6 — Порядок памяти](06-memory-ordering.md). Вы знаете пары release/acquire.
- Вы знаете `UnsafeCell` из [Урока 06.4](../06-rust-advanced/04-interior-mutability.md), `unsafe` и Miri из [Урока 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md), а `Deref` и `Drop` — из [Урока 05.9](../05-rust-intermediate/09-smart-pointers.md).
- Продолжайте работать в `~/projects/rust/code/p14-concurrency`. Этот урок добавляет в пакет **библиотечную** часть: файл `src/lib.rs` рядом с `src/main.rs`. Программы из `src/bin/` смогут использовать её как крейт `p14_concurrency` (Cargo превращает дефис в подчёркивание).

## Новые слова

| Слово | Значение |
|---|---|
| **спин-блокировка** (spin lock) | Блокировка, при которой ожидающий поток не засыпает: он крутится в цикле («спинится») и снова и снова проверяет блокировку, пока она не освободится. |
| **вращение** (spinning) | Активное ожидание в цикле с тратой CPU (Central Processing Unit, центральный процессор), пока не изменится условие. |
| **проверка с установкой** (test-and-set) | Атомарная операция, которая ставит флаг и возвращает его старое значение за один шаг. `swap(true)` на `AtomicBool` — такая операция. |
| **`UnsafeCell<T>`** | Базовый строительный блок внутренней изменяемости: даёт сырой указатель `*mut T` из `&self`. Он не `Sync`, поэтому потокобезопасность приходится обещать самому. |
| **`unsafe impl`** | Реализация трейта, правила которого компилятор проверить не может, например `Sync`. Вы обещаете, что правила соблюдаются. |
| **комментарий `SAFETY:`** | Комментарий над `unsafe`-кодом, объясняющий, почему код корректен: какое правило должно соблюдаться и почему оно соблюдается. |
| **guard-объект** (guard) | Значение, которое доказывает, что вы держите блокировку, даёт доступ к данным и снимает блокировку в своём `Drop`. |
| **`spin_loop()`** | `std::hint::spin_loop`: подсказка процессору, что поток крутится в цикле активного ожидания, чтобы тот мог сэкономить энергию или дать поработать другому аппаратному потоку. |
| **TTAS** (test-and-test-and-set, проверка и проверка с установкой) | Спин-блокировка, которая ждёт обычными чтениями и пробует дорогой `swap`, только когда блокировка выглядит свободной. |
| **вытеснение** (preemption) | OS (Operating System, операционная система) останавливает выполняющийся поток, чтобы дать поработать другому. Вытесненный владелец блокировки задерживает всех, кто крутится в ожидании. |
| **переподписка** (oversubscription) | Готовых к работе потоков больше, чем ядер CPU, поэтому какие-то потоки всегда ждут ядро. |
| **билетная блокировка** (ticket lock) | Честная спин-блокировка: каждый поток берёт номер и ждёт, пока этот номер не обслужат, как очередь на почте. |
| **детектор гонок данных** (data race detector) | Часть Miri, которая сообщает о двух несинхронизированных обращениях к одной памяти, хотя бы одно из которых — запись, как о неопределённом поведении. |

## 1. Идея

Блокировке нужен один бит состояния: свободна или занята. Хватает `AtomicBool`:

```text
 lock():   loop { old = swap(locked, true); if old == false { return } }   ← I flipped it: it is mine
 unlock(): store(locked, false)
```

`swap(true)` ставит флаг и возвращает старое значение за **один** атомарный шаг (*проверка с установкой*). Если старое значение было `false`, блокировка была свободна и теперь наша. Если `true` — её держит кто-то другой, и мы пробуем снова. Поскольку обмен атомарен, два потока никогда не увидят `false` оба.

Это и есть **спин-блокировка**. Она никогда ничего не просит у OS. Поэтому это простейшая из возможных блокировок и, как покажет раздел 6, обычно плохая. И всё же написать её — лучший способ понять, что обязана делать любая блокировка.

## 2. Тип блокировки и обещание `Sync`

Создайте `src/lib.rs`:

```rust
//! Small concurrency building blocks written in Phase 14.

pub mod spin_lock;
```

и файл модуля `src/spin_lock.rs` (тесты в конце объясняются в разделе 5):

```rust
//! A spin lock: the simplest lock, built from one atomic flag.

use std::cell::UnsafeCell;
use std::hint::spin_loop;
use std::ops::{Deref, DerefMut};
use std::sync::atomic::{AtomicBool, Ordering};

pub struct SpinLock<T> {
    locked: AtomicBool,
    value: UnsafeCell<T>,
}

// SAFETY: only one thread at a time can reach `value` (through a Guard),
// so sharing the lock is fine as long as T itself may move between threads.
unsafe impl<T: Send> Sync for SpinLock<T> {}

impl<T> SpinLock<T> {
    pub const fn new(value: T) -> Self {
        SpinLock {
            locked: AtomicBool::new(false),
            value: UnsafeCell::new(value),
        }
    }

    pub fn lock(&self) -> Guard<'_, T> {
        // Acquire: pairs with the Release store in Guard::drop.
        while self.locked.swap(true, Ordering::Acquire) {
            spin_loop(); // tell the CPU that we are waiting in a loop
        }
        Guard { lock: self }
    }
}

/// Proof that we hold the lock. Unlocks when dropped.
pub struct Guard<'a, T> {
    lock: &'a SpinLock<T>,
}

impl<T> Deref for Guard<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        // SAFETY: the guard exists, so this thread holds the lock.
        unsafe { &*self.lock.value.get() }
    }
}

impl<T> DerefMut for Guard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        // SAFETY: the guard exists, so this thread holds the lock.
        unsafe { &mut *self.lock.value.get() }
    }
}

impl<T> Drop for Guard<'_, T> {
    fn drop(&mut self) {
        // Release: everything we did with the value becomes visible
        // to the next thread whose Acquire swap sees `false`.
        self.lock.locked.store(false, Ordering::Release);
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn no_update_is_lost() {
        let counter = SpinLock::new(0u64);
        thread::scope(|s| {
            for _ in 0..4 {
                s.spawn(|| {
                    for _ in 0..100 {
                        *counter.lock() += 1;
                    }
                });
            }
        });
        assert_eq!(*counter.lock(), 400);
    }

    #[test]
    fn works_with_non_copy_data() {
        let names = SpinLock::new(Vec::new());
        thread::scope(|s| {
            for t in 0..4 {
                let names = &names;
                s.spawn(move || names.lock().push(format!("thread {t}")));
            }
        });
        let mut v = names.lock().clone();
        v.sort();
        assert_eq!(v, ["thread 0", "thread 1", "thread 2", "thread 3"]);
    }
}
```

**Что происходит** (тип и `Sync`):

1. `SpinLock<T>` хранит флаг и данные. Данные лежат в `UnsafeCell<T>`, потому что мы хотим менять их через `&self` (многие потоки делят `&SpinLock`). `UnsafeCell::get()` возвращает сырой указатель `*mut T`; превратить его в ссылку — `unsafe`, и *мы* должны гарантировать, что два потока не сделают этого одновременно.
2. `new` — это `const fn`, поэтому `SpinLock` может жить даже в `static`.
3. `UnsafeCell<T>` **не** `Sync`: компилятор не может знать, как вы будете им пользоваться. Поэтому и `SpinLock<T>` не `Sync`, и делить его между потоками нельзя. Мы обещаем это через `unsafe impl<T: Send> Sync for SpinLock<T> {}`. Обещание верно, потому что добраться до значения можно только через `Guard`, а одновременно может существовать только один guard-объект.
4. Почему `T: Send`, а не `T: Sync`? Через блокировку разные потоки получают `&mut T` по очереди. Это похоже на **перемещение** значения от потока к потоку, а именно это и разрешает `Send`. У `std::sync::Mutex<T>` такое же ограничение.
5. Комментарий `// SAFETY:` называет причину обещания. Он должен быть у каждого блока `unsafe` и каждого `unsafe impl` (соглашение из Урока 06.6).

Без строки `unsafe impl` компилятор останавливает любую программу, которая делит блокировку между потоками. Если закомментировать её, `cargo build --bin spin_demo` (программа из раздела 4) падает так:

```text
error[E0277]: `UnsafeCell<u64>` cannot be shared between threads safely
  --> src/bin/spin_demo.rs:8:21
   |
   ...
   = help: within `SpinLock<u64>`, the trait `Sync` is not implemented for `UnsafeCell<u64>`
note: required because it appears within the type `SpinLock<u64>`
  --> src/spin_lock.rs:8:12
   |
 8 | pub struct SpinLock<T> {
   |            ^^^^^^^^
   = note: required for `&SpinLock<u64>` to implement `Send`
```

> 🔒 **Безопасность:** `unsafe impl Sync` отключает проверку компилятора для **каждого** пользователя вашего типа. Если обещание ложно, безопасный код где-то ещё может создать гонку данных: неопределённое поведение, которое в худшем случае становится уязвимостью памяти, пригодной для атаки. Держите такой код маленьким, комментируйте причину и проверяйте его через Miri (раздел 5).

## 3. Guard-объект

`lock()` возвращает не `()` и не `bool`. Он возвращает `Guard`, и guard — единственная дверь к данным:

1. `Deref` даёт `&T`, `DerefMut` даёт `&mut T`. Поэтому `*counter.lock() += 1` и `names.lock().push(…)` работают, как с `MutexGuard`. Каждый блок `unsafe` оправдан одним фактом: guard существует, только пока его поток держит блокировку.
2. `Drop` снимает блокировку. Забыть разблокировать нельзя, и разблокировать дважды нельзя, потому что значение уничтожается ровно один раз.
3. Guard заимствует блокировку (`&'a SpinLock<T>`), поэтому проверка заимствований гарантирует, что блокировка живёт дольше любого guard-объекта.

Это то же устройство, что у `std::sync::Mutex` и его `MutexGuard`: система типов превращает «пожалуйста, не забудьте правильно заблокировать и разблокировать» в правила, которые проверяет компилятор.

> 📏 **Соглашение:** делайте небезопасную часть как можно меньше и прячьте её за безопасным API (Application Programming Interface, программный интерфейс). Здесь `unsafe` всего две строки, и ни одному пользователю `SpinLock` не приходится писать `unsafe`.

## 4. Почему `Acquire` и `Release`

Блокировка защищает данные, только если следующий владелец **видит**, что записал предыдущий. Это пара release/acquire из Урока 14.6:

```text
 thread A (owner)                          thread B (next owner)
 *value += 1          (plain write)
 store(false, Release) ─ ─ ─ ─ ─ ─ ─ ─ ─▶ swap(true, Acquire) returns false
                                           *value += 1  → sees A's write
```

- **`Acquire` на обмене, который берёт блокировку**: всё, что предыдущий владелец сделал до своего release, видно нам после того, как мы получили блокировку.
- **`Release` на записи, которая освобождает блокировку**: всё, что мы сделали, держа блокировку, видно следующему владельцу.

Что будет с `Relaxed`? Создайте `src/bin/spin_demo.rs`, который использует библиотеку:

```rust
use p14_concurrency::spin_lock::SpinLock;
use std::thread;

fn main() {
    let counter = SpinLock::new(0u64);
    thread::scope(|s| {
        for _ in 0..4 {
            s.spawn(|| {
                for _ in 0..1_000_000 {
                    *counter.lock() += 1;
                }
            });
        }
    });
    println!("expected 4000000, got {}", *counter.lock());
}
```

С правильными порядками `cargo run --release --bin spin_demo` печатает:

```text
expected 4000000, got 4000000
```

Теперь в качестве эксперимента замените оба порядка в `src/spin_lock.rs` на `Ordering::Relaxed` (`swap` в `lock` и `store` в `drop`) и запустите снова, дважды:

```text
expected 4000000, got 3999969
expected 4000000, got 3999986
```

**Что происходит:** каждый поток по-прежнему правильно берёт блокировку (обмен всё ещё атомарен, поэтому «внутри» одновременно только один поток). Но `+= 1` — это обычные чтение и запись `u64`. Без release/acquire следующий владелец может прочитать счётчик **раньше**, чем до него дошла запись предыдущего владельца, — на вашем слабо упорядоченном Mac, — и несколько увеличений теряются. Блокировка выглядела правильной и была неверной. Сейчас же верните порядки `Acquire` и `Release`.

## 5. Проверка блокировки: тесты и Miri

Два теста в конце `src/spin_lock.rs` — обычные стресс-тесты: четыре потока нагружают блокировку, а результат должен быть точным. Тесты библиотеки запускаются через `cargo test --lib`:

```text
running 2 tests
test spin_lock::tests::no_update_is_lost ... ok
test spin_lock::tests::works_with_non_copy_data ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Стресс-тест может пройти случайно, особенно на машине, которая переупорядочивает меньше. **Miri** (Урок 06.6) выполняет программу в интерпретаторе, который проверяет каждое обращение к памяти, а его **детектор гонок данных** сообщает о несинхронизированных обращениях, даже когда результат случайно правильный. Запустите тесты под Miri (нужен тулчейн nightly с компонентом `miri`, как в Уроке 06.6):

```bash
cargo +nightly miri test --lib
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.70s
```

С экспериментом `Relaxed` из раздела 4 Miri сразу находит ошибку:

```text
error: Undefined Behavior: Data race detected between (1) non-atomic write on thread `unnamed-2` and (2) retag write of type `u64` on thread `unnamed-3` at alloc40451
  --> src/spin_lock.rs:50:18
   |
50 |         unsafe { &mut *self.lock.value.get() }
   |                  ^^^^^^^^^^^^^^^^^^^^^^^^^^^ (2) just happened here
   |
help: and (1) occurred earlier here
  --> src/spin_lock.rs:74:25
   |
74 |                         *counter.lock() += 1;
   |                         ^^^^^^^^^^^^^^^^^^^^
```

**Что происходит:** Miri отслеживает, какие операции происходят-до каких. Один поток записал счётчик (строка 74, `+= 1` в тесте), а другой поток создал `&mut` на ту же память (строка 50) без всякого «происходит-до» между ними: гонка данных, а значит, неопределённое поведение. Miri не нужно, чтобы ошибка дала неверное число; он сам видит пропущенную синхронизацию. (*Retag* — так Miri называет создание ссылки; это считается обращением к памяти.)

> 🧪 **Тестирование:** для кода с `unsafe` и атомарными операциями используйте три инструмента вместе: стресс-тесты (быстро, на настоящем железе), Miri (находит гонки данных и другое неопределённое поведение в тех запусках, которые выполняет) и loom (перебирает все чередования небольшой модели; Урок 14.9 показывает, как проверить такую блокировку с `loom::cell::UnsafeCell`).

## 6. Стоит ли пользоваться спин-блокировкой?

В обычной программе — почти никогда. Измерим почему. Создайте `src/bin/spin_vs_mutex.rs`:

```rust
use p14_concurrency::spin_lock::SpinLock;
use std::sync::Mutex;
use std::thread;
use std::time::{Duration, Instant};

const TOTAL: u64 = 2_400_000;

/// `threads` threads share TOTAL increments of one counter.
fn time_it(threads: u64, add_one: impl Fn() + Sync) -> Duration {
    let start = Instant::now();
    thread::scope(|s| {
        for _ in 0..threads {
            s.spawn(|| {
                for _ in 0..TOTAL / threads {
                    add_one();
                }
            });
        }
    });
    start.elapsed()
}

fn main() {
    for threads in [1, 4, 48] {
        let spin = SpinLock::new(0u64);
        let mutex = Mutex::new(0u64);
        let s = time_it(threads, || *spin.lock() += 1);
        let m = time_it(threads, || *mutex.lock().unwrap() += 1);
        println!("{threads:>2} threads: SpinLock {s:>9.1?}   Mutex {m:>9.1?}");
    }
}
```

`cargo run --release --bin spin_vs_mutex`, два запуска:

```text
 1 threads: SpinLock     5.3ms   Mutex    12.2ms
 4 threads: SpinLock   158.6ms   Mutex    33.3ms
48 threads: SpinLock      2.3s   Mutex    36.0ms
 1 threads: SpinLock     3.1ms   Mutex    18.5ms
 4 threads: SpinLock   117.0ms   Mutex    42.7ms
48 threads: SpinLock      3.2s   Mutex    35.4ms
```

**Что происходит:**

1. **Один поток**: спин-блокировка быстрее. Взять свободную блокировку — один `swap`, и больше делать нечего.
2. **Четыре потока**: спин-блокировка в 3–5 раз медленнее. Каждый ожидающий поток выполняет `swap(true)` в цикле. Каждый обмен — это *запись*, поэтому каждый перетягивает кэш-линию флага к своему ядру, отбирая её у владельца, которому она потом нужна обратно, чтобы разблокировать.
3. **48 потоков** на 12 ядрах (переподписка): спин-блокировка примерно в 60–90 раз медленнее. Теперь владельца блокировки часто **вытесняет** OS: она останавливает его, чтобы запустить другой поток, возможно, крутящийся в ожидании. Пока владелец спит, все крутящиеся сжигают свои кванты времени впустую. `Mutex` усыпляет ожидающие потоки, поэтому владелец получает ядро и быстро заканчивает.

Значит, спин-блокировка имеет смысл, только когда выполняется **всё** сразу: критическая секция крошечная, за блокировку редко борются, а владельца нельзя вытеснить. Так бывает внутри ядра OS (которое может отключить вытеснение) или на микроконтроллере без OS, но почти никогда в обычной программе. Стандартный `Mutex` — лучший выбор: в Linux он сначала немного крутится, а потом засыпает через futex (Урок 14.2); в macOS системный мьютекс усыпляет ожидающие потоки.

> ⚡ **Производительность:** цикл ожидания должен хотя бы ждать обычными **чтениями** и пробовать `swap`, только когда блокировка выглядит свободной (*проверка и проверка с установкой*, Упражнение 2). Чтения позволяют всем ожидающим ядрам держать общую копию кэш-линии, поэтому они не отбирают её у владельца. На тестовой машине это ускорило случай с 48 потоками примерно в 3 раза — и он всё равно остался примерно в 25 раз медленнее `Mutex`.

## Частые ошибки

1. **`Relaxed` в блокировке.** Блокировка по-прежнему пускает по одному потоку, но данные не публикуются: потерянные обновления на arm64 и гонка данных для Miri (разделы 4 и 5).
2. **Неверное ограничение у `Sync`.** `unsafe impl<T> Sync for SpinLock<T>` (без `T: Send`) позволил бы, например, делить `Rc` между потоками через блокировку. Копируйте ограничение `std::sync::Mutex`: `T: Send`.
3. **Возвращать из `lock()` `&mut T` вместо guard-объекта.** Тогда ничто не разблокирует в нужный момент, а ссылка может пережить блокировку. Всегда возвращайте guard, который разблокирует в `Drop`.
4. **Спин-блокировка там, где потоки могут ждать долго.** Если критическая секция может блокироваться (ввод-вывод, сон, другая блокировка), ожидающие жгут CPU всё это время. Используйте `Mutex`.
5. **`unsafe` без комментария `SAFETY:`.** Следующий читатель (в том числе вы через месяц) не сможет проверить обещание.

## Попробуйте сами

1. Запустите `cargo test --lib` и `cargo +nightly miri test --lib`. Затем выполните эксперимент `Relaxed` из раздела 4 и снова запустите оба. Что из этого надёжно падает? Верните порядки.
2. Удалите строку `spin_loop();` из `lock` (оставьте пустое тело цикла `{}`; компилятор тогда предупредит, что импорт `spin_loop` не используется) и снова запустите `spin_vs_mutex`. Меняет ли подсказка числа на вашей машине? На тестовой машине запуск без подсказки лучше не стал (48 потоков: `3.6s`); разброс между запусками больше, чем разница. Верните строку.
3. В `spin_vs_mutex.rs` добавьте `12` в список чисел потоков (столько ядер у тестовой машины). Ближе ли там спин-блокировка к мьютексу? (На тестовой машине: `12 threads: SpinLock 864.9ms   Mutex 25.2ms`.)

## Упражнения

### Упражнение 1 — `try_lock` (легко)

Добавьте в `SpinLock` метод `pub fn try_lock(&self) -> Option<Guard<'_, T>>`: он берёт блокировку, только если она свободна прямо сейчас, а иначе сразу возвращает `None`. Используйте `compare_exchange` (без цикла). Напишите тест `try_lock_fails_while_locked`: пока жив guard из `lock()`, `try_lock()` возвращает `None`; после уничтожения guard он возвращает `Some`, а значение на месте.

<details>
<summary>Подсказка</summary>

`compare_exchange(false, true, success, failure)` срабатывает, только если флаг был `false`. Какой порядок нужен при успехе, а какой при неудаче? (При неудаче мы не получили блокировку и не читали данных.) `Result::ok()` и `Option::map` превращают результат в `Option<Guard>`.

</details>

<details>
<summary>Ответ</summary>

Внутри `impl<T> SpinLock<T>`:

```rust
    /// Takes the lock only if it is free right now.
    pub fn try_lock(&self) -> Option<Guard<'_, T>> {
        // Acquire on success: pairs with the Release store in Guard::drop.
        self.locked
            .compare_exchange(false, true, Ordering::Acquire, Ordering::Relaxed)
            .ok()
            .map(|_| Guard { lock: self })
    }
```

В `mod tests`:

```rust
    #[test]
    fn try_lock_fails_while_locked() {
        let lock = SpinLock::new(5);
        let guard = lock.lock();
        assert!(lock.try_lock().is_none());
        drop(guard);
        let again = lock.try_lock().expect("lock is free now");
        assert_eq!(*again, 5);
    }
```

```text
test spin_lock::tests::try_lock_fails_while_locked ... ok
```

Почему это работает: `compare_exchange` записывает `true`, только если флаг всё ещё `false`, за один атомарный шаг, поэтому два вызова `try_lock` никогда не сработают оба. При успехе используется `Acquire` по той же причине, что и в `lock()`: мы будем читать данные. Неудача может быть `Relaxed`, потому что тогда мы возвращаем `None` и ничего не трогаем. Мы используем сильный `compare_exchange`, потому что ложная неудача слабой версии заставила бы `try_lock` вернуть `None` для свободной блокировки.

</details>

### Упражнение 2 — Проверка и проверка с установкой (средне)

Измените `lock()` так, чтобы ожидающий поток крутился на `load(Ordering::Relaxed)` и пробовал `swap`, только когда увидел `false`. Все тесты должны по-прежнему проходить (и под Miri тоже). Запустите `spin_vs_mutex` до и после и сравните числа.

<details>
<summary>Подсказка</summary>

Используйте внешний `loop`: сначала попробуйте `swap(true, Acquire)`; если он вернул `false`, верните guard. Иначе выполните внутренний `while self.locked.load(Relaxed) { spin_loop(); }` и снова идите по внешнему циклу. Почему внутреннее чтение может быть `Relaxed`?

</details>

<details>
<summary>Ответ</summary>

```rust
    pub fn lock(&self) -> Guard<'_, T> {
        // Acquire: pairs with the Release store in Guard::drop.
        loop {
            // Try to take the lock: one atomic write.
            if !self.locked.swap(true, Ordering::Acquire) {
                return Guard { lock: self };
            }
            // Busy: wait with plain reads, which do not take the cache line away.
            while self.locked.load(Ordering::Relaxed) {
                spin_loop();
            }
        }
    }
```

Результаты `cargo run --release --bin spin_vs_mutex` с этой версией на тестовой машине:

```text
 1 threads: SpinLock     3.8ms   Mutex    11.9ms
 4 threads: SpinLock   334.8ms   Mutex    32.3ms
48 threads: SpinLock   885.9ms   Mutex    31.2ms
 1 threads: SpinLock     2.8ms   Mutex    10.2ms
 4 threads: SpinLock    87.1ms   Mutex    28.6ms
48 threads: SpinLock   973.3ms   Mutex    39.7ms
```

Почему это работает: внутренний `load` только *читает*, поэтому все ожидающие ядра могут держать общую копию кэш-линии, а владелец — свою; никто не отбирает линию, пока запись release владельца её не изменит. Внутреннее чтение может быть `Relaxed`, потому что оно решает только, *когда пробовать*; блокировку на самом деле берёт и синхронизирует `swap` с `Acquire`. С 48 потоками время упало с 2,3–3,2 с примерно до 0,9 с. Числа для 4 потоков сильно скачут от запуска к запуску, что типично для нагруженного вращения. Это всё равно намного медленнее `Mutex` — и это главный вывод.

</details>

### Упражнение 3 — Честная билетная блокировка (сложно)

Обычная спин-блокировка нечестна: выигрывает поток, которому повезло сделать обмен в нужный момент, а невезучий поток может ждать долго. Напишите `src/ticket_lock.rs` (добавьте `pub mod ticket_lock;` в `src/lib.rs`) с типом `TicketLock<T>`: два счётчика `AtomicU32`, `next_ticket` и `now_serving`. `lock()` берёт билет через `fetch_add` и крутится, пока `now_serving` не станет равен его билету; уничтожение guard-объекта сдвигает `now_serving` на один. Напишите два теста: ни одно обновление не теряется (4 потока × 100 увеличений) и, с одним потоком, `now_serving` проходит 0, 1, 2, 3, 4, когда блокировку пять раз берут и отпускают. Затем сравните её скорость со `SpinLock` (4 потока по 250 000 увеличений).

<details>
<summary>Подсказка</summary>

Какая операция публикует данные следующему владельцу, а какая их получает? `fetch_add` на `next_ticket` только раздаёт номера. Тесты внутри `ticket_lock.rs` могут читать приватные поля, потому что тестовый модуль — дочерний модуль.

</details>

<details>
<summary>Ответ</summary>

```rust
//! A ticket lock: a fair spin lock. Threads are served in arrival order.

use std::cell::UnsafeCell;
use std::hint::spin_loop;
use std::ops::{Deref, DerefMut};
use std::sync::atomic::{AtomicU32, Ordering};

pub struct TicketLock<T> {
    next_ticket: AtomicU32,
    now_serving: AtomicU32,
    value: UnsafeCell<T>,
}

// SAFETY: as for SpinLock, only the ticket holder reaches `value`.
unsafe impl<T: Send> Sync for TicketLock<T> {}

impl<T> TicketLock<T> {
    pub const fn new(value: T) -> Self {
        TicketLock {
            next_ticket: AtomicU32::new(0),
            now_serving: AtomicU32::new(0),
            value: UnsafeCell::new(value),
        }
    }

    pub fn lock(&self) -> TicketGuard<'_, T> {
        // Take a number. Relaxed: the number itself publishes nothing.
        let my_ticket = self.next_ticket.fetch_add(1, Ordering::Relaxed);
        // Acquire: pairs with the Release in TicketGuard::drop.
        while self.now_serving.load(Ordering::Acquire) != my_ticket {
            spin_loop();
        }
        TicketGuard { lock: self }
    }
}

pub struct TicketGuard<'a, T> {
    lock: &'a TicketLock<T>,
}

impl<T> Deref for TicketGuard<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        // SAFETY: we hold the current ticket.
        unsafe { &*self.lock.value.get() }
    }
}

impl<T> DerefMut for TicketGuard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        // SAFETY: we hold the current ticket.
        unsafe { &mut *self.lock.value.get() }
    }
}

impl<T> Drop for TicketGuard<'_, T> {
    fn drop(&mut self) {
        // Release: the next ticket holder sees everything we did.
        self.lock.now_serving.fetch_add(1, Ordering::Release);
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn no_update_is_lost() {
        let counter = TicketLock::new(0u64);
        thread::scope(|s| {
            for _ in 0..4 {
                s.spawn(|| {
                    for _ in 0..100 {
                        *counter.lock() += 1;
                    }
                });
            }
        });
        assert_eq!(*counter.lock(), 400);
    }

    #[test]
    fn tickets_are_served_in_order() {
        let lock = TicketLock::new(());
        for expected in 0..5 {
            assert_eq!(lock.now_serving.load(Ordering::Relaxed), expected);
            drop(lock.lock());
        }
        assert_eq!(lock.next_ticket.load(Ordering::Relaxed), 5);
    }
}
```

`cargo test --lib` и `cargo +nightly miri test --lib` проходят все пять тестов библиотеки этапа (включая три теста `spin_lock`). Небольшая программа сравнения (4 потока × 250 000 увеличений для каждой блокировки) напечатала на тестовой машине в двух запусках:

```text
SpinLock  : 1000000 in 32.1ms
TicketLock: 1000000 in 1.5s
SpinLock  : 1000000 in 21.1ms
TicketLock: 1000000 in 332.9ms
```

Почему это работает: `fetch_add` выдаёт каждому потоку свой билет в порядке прихода, и войти может только поток, чей билет равен `now_serving`. Менять `now_serving` может только владелец — на единицу, с `Release`; следующий владелец билета читает его с `Acquire`, поэтому данные передаются правильно. Блокировка **честная** (кто первый пришёл, того первым и обслужили), но у честности есть цена: блокировка должна достаться именно следующему потоку, даже если он в этот момент не выполняется, тогда как обычная спин-блокировка достаётся любому готовому потоку. Поэтому здесь она оказалась примерно в 15–50 раз медленнее.

</details>

## Вопросы для самопроверки

1. Почему `swap(true)` достаточно, чтобы безопасно взять спин-блокировку?

<details>
<summary>Ответ</summary>

Он ставит флаг и возвращает старое значение за один атомарный шаг. Старое значение `false` может получить только один поток; все остальные получают `true` и должны пробовать снова.

</details>

2. Почему `SpinLock<T>` нужен `unsafe impl Sync` и почему ограничение именно `T: Send`?

<details>
<summary>Ответ</summary>

Он содержит `UnsafeCell<T>`, который не `Sync`, поэтому компилятор не делает `SpinLock<T>` `Sync`. Мы обещаем это сами, потому что к значению одновременно имеет доступ только один guard. Ограничение `T: Send`, потому что блокировка передаёт значение (как `&mut T`) от одного потока к следующему, а это похоже на перемещение между потоками.

</details>

3. Какие две работы выполняет тип guard-объекта?

<details>
<summary>Ответ</summary>

Он даёт доступ к данным (`Deref`/`DerefMut`) только пока блокировка удерживается и снимает блокировку в `Drop`, поэтому нельзя забыть разблокировать или разблокировать дважды.

</details>

4. Что ломается, если блокировка использует `Relaxed` и для обмена, и для записи?

<details>
<summary>Ответ</summary>

Взаимное исключение по-прежнему работает, но записи предыдущего владельца не обязательно видны следующему. На arm64 часть увеличений теряется, а Miri сообщает о гонке данных.

</details>

5. Почему спин-блокировка так медленна с 48 потоками на 12 ядрах?

<details>
<summary>Ответ</summary>

Владельца блокировки часто вытесняет OS. Пока он не выполняется, никто не может освободить блокировку, и все крутящиеся потоки тратят свои кванты впустую. `Mutex` усыпляет ожидающих, поэтому владелец получает время CPU и заканчивает.

</details>

6. Что находит Miri, чего может не найти стресс-тест?

<details>
<summary>Ответ</summary>

Гонку данных: два обращения к одной памяти без «происходит-до», хотя бы одно из них — запись. Miri сам обнаруживает пропущенную синхронизацию, даже в тех запусках, где итоговый результат случайно правильный.

</details>

## Итоги

- Спин-блокировка — это `AtomicBool` плюс цикл: `swap(true)`, пока старое значение не окажется `false`; разблокировка — `store(false)`.
- Данные лежат в `UnsafeCell<T>`. Потокобезопасность вы обещаете через `unsafe impl<T: Send> Sync`, обоснованный в комментарии `SAFETY:`.
- Guard с `Deref`, `DerefMut` и `Drop` — единственный путь к данным и всегда снимает блокировку.
- Берите блокировку с `Acquire`, отпускайте с `Release`. С `Relaxed` на Mac терялись обновления, а Miri сообщал о гонке данных.
- Проверяйте небезопасный конкурентный код стресс-тестами, Miri и loom.
- Спин-блокировки сильно проигрывают при конкуренции и переподписке. В обычных программах используйте `std::sync::Mutex`; спин-блокировкам место в ядрах OS и на голом железе.

## Источники и что почитать

- [Rust Atomics and Locks, chapter 4 — Building Our Own Spin Lock](https://marabos.nl/atomics/building-spinlock.html) — Mara Bos шаг за шагом строит ту же блокировку, с guard-объектом и рассуждением о безопасности.
- [std::cell::UnsafeCell](https://doc.rust-lang.org/std/cell/struct.UnsafeCell.html) — правила получения ссылок из `UnsafeCell`.
- [std::hint::spin_loop](https://doc.rust-lang.org/std/hint/fn.spin_loop.html) — что делает подсказка для цикла ожидания.
- [The Rustonomicon — Send and Sync](https://doc.rust-lang.org/nomicon/send-and-sync.html) — когда корректно самому реализовывать `Send` и `Sync`.
- [Miri на GitHub](https://github.com/rust-lang/miri) — как устроен Miri и что проверяет его детектор гонок данных.

---

[🏠 Начало](../START-HERE.md) · [← 14.6 Порядок памяти](06-memory-ordering.md) · [📋 Этап 14](README.md) · [14.8 Неблокирующие структуры данных →](08-lock-free-data-structures.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
