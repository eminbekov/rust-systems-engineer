<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 14.4 Пулы потоков, scoped-потоки и rayon](04-thread-pools-scoped-threads-and-rayon.md) · [📋 Этап 14](README.md) · [14.6 Порядок памяти →](06-memory-ordering.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/14-concurrency-and-async/05-atomics.md) <!-- nav -->

# Урок 14.5 — Атомарные операции

> **Этап 14 — Конкурентность и асинхронность подробно** · Урок 5 из 16 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что такое атомарная операция и какие атомарные типы есть в Rust.
- Базовые операции: `load`, `store`, `swap` и семейство `fetch_…`, а также почему `fetch_add` молча переходит через ноль.
- Как работает сравнение с обменом (`compare_exchange`), как написать цикл CAS (compare-and-swap, сравнение с обменом) и когда этот цикл за вас пишут `try_update` и `update`.
- Типичные применения: уникальные идентификаторы, флаг остановки, счётчик с ограничением, два числа, упакованные в одну атомарную переменную.
- Что на самом деле делает процессор вашего Mac для атомарной операции.

## Перед началом

- Вы прошли [Урок 14.4 — Пулы потоков, scoped-потоки и rayon](04-thread-pools-scoped-threads-and-rayon.md).
- Вы видели *потерянное обновление* с `load` + `store` в [Уроке 14.1](01-why-concurrency-is-hard.md) и первый счётчик `AtomicUsize` в [Уроке 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md).
- Вы знаете внутреннюю изменяемость из [Урока 06.4](../06-rust-advanced/04-interior-mutability.md), битовые операции из [Урока 12.4](../12-computer-systems/04-bitwise-operations.md) и немного умеете читать ассемблер arm64 ([Урок 12.7](../12-computer-systems/07-reading-assembly.md)).
- Продолжайте работать в `~/projects/rust/code/p14-concurrency`; каждый пример — новый файл в `src/bin/`.

## Новые слова

| Слово | Значение |
|---|---|
| **атомарная операция** (atomic operation) | Операция с памятью, которую другие потоки видят только как «не начата» или «завершена», но никогда наполовину. |
| **атомарный тип** (atomic type) | Тип вроде `AtomicU64` или `AtomicBool`, методы которого — атомарные операции. Его можно менять через общую ссылку `&`. |
| **`Ordering`** | Аргумент каждого атомарного метода. Говорит, какие *другие* операции с памятью упорядочены относительно этой. В этом уроке мы всегда используем `Relaxed`; остальные объясняет Урок 14.6. |
| **`Relaxed`** | Самый слабый порядок: операция над *этой одной переменной* атомарна, и больше ничего не обещается. Достаточно для счётчиков и идентификаторов. |
| **чтение-изменение-запись** (read-modify-write, RMW) | Одна атомарная операция, которая читает значение, вычисляет новое и записывает его без промежутка: `swap`, `fetch_add`, `compare_exchange`. |
| **переход через границу** (wrap around) | Число выходит за свой максимум и начинается снова с минимума: для `u8` 255 + 1 даёт 0. |
| **CAS** (compare-and-swap, сравнение с обменом) | Атомарная операция: «если значение всё ещё `current`, замени его на `new`; иначе скажи мне настоящее значение». В Rust это `compare_exchange`. |
| **цикл CAS** (CAS loop) | Прочитать значение, вычислить новое, попробовать записать его через CAS и повторить, если другой поток тем временем изменил значение. |
| **ложная неудача** (spurious failure) | `compare_exchange_weak` может не сработать, даже если значение совпадало. Это разрешено, чтобы на некоторых CPU (Central Processing Unit, центральный процессор) он был быстрее. В цикле это неважно. |
| **LSE** (Large System Extensions, расширения для больших систем) | Часть набора инструкций arm64 (начиная с ARMv8.1) с отдельными инструкциями для атомарных операций, например `ldadd` (загрузить и прибавить). В чипах Apple M она есть. |
| **ложное разделение** (false sharing) | Два потока пишут *разные* переменные, которые случайно лежат в одной кэш-линии, поэтому линия всё равно прыгает между ядрами, как если бы переменная была общей. |

## 1. Что значит «атомарный»

В Уроке 14.1 четыре потока выполняли `let v = counter.load(); counter.store(v + 1);` и потеряли три четверти обновлений. Каждый шаг был безопасен, но другой поток мог выполниться **между** этими шагами.

У **атомарной операции** нет «между». Когда поток выполняет `fetch_add(1)`, любой другой поток видит счётчик либо до сложения, либо после, но никогда посередине, и два вызова `fetch_add` никогда не прочитают одно и то же старое значение. Это гарантирует железо процессора (раздел 5).

В стандартной библиотеке в `std::sync::atomic` есть такие типы:

| Тип | Хранит |
|---|---|
| `AtomicBool` | `bool` |
| `AtomicI8`, `AtomicU8`, `AtomicI16`, `AtomicU16`, `AtomicI32`, `AtomicU32`, `AtomicI64`, `AtomicU64` | целые числа этого размера |
| `AtomicIsize`, `AtomicUsize` | целые размером с указатель |
| `AtomicPtr<T>` | сырой указатель `*mut T` (его использует Урок 14.8) |

В них особенны две вещи:

- **Они меняются через `&self`.** Не нужны ни `let mut`, ни `&mut`. Как и `Cell` (Урок 06.4), они построены на `UnsafeCell`, но в отличие от `Cell` они `Sync`: много потоков могут одновременно держать `&AtomicU64`. Поэтому их можно положить в `Arc`, заимствовать в scoped-потоках или хранить в `static`.
- **Каждый метод принимает `Ordering`.** Пока мы всегда передаём `Ordering::Relaxed`: «сделай эту одну операцию атомарной, больше мне ничего не нужно». Это верно всегда, когда атомарная переменная — *просто число* (счётчик, идентификатор, статистика). Когда атомарная переменная должна сообщать «*другие* данные готовы», `Relaxed` недостаточно; это тема Урока 14.6.

## 2. Базовые операции

Создайте `src/bin/atomic_basics.rs`:

```rust
use std::sync::atomic::{AtomicBool, AtomicU32, Ordering};

fn main() {
    let x = AtomicU32::new(10); // no `mut` needed: atomics change through `&self`

    println!("load           -> {}", x.load(Ordering::Relaxed));
    x.store(20, Ordering::Relaxed);
    println!("store(20)      -> now {}", x.load(Ordering::Relaxed));

    let old = x.swap(30, Ordering::Relaxed);
    println!("swap(30)       -> old {old}, now {}", x.load(Ordering::Relaxed));

    let old = x.fetch_add(5, Ordering::Relaxed);
    println!("fetch_add(5)   -> old {old}, now {}", x.load(Ordering::Relaxed));

    let old = x.fetch_sub(15, Ordering::Relaxed);
    println!("fetch_sub(15)  -> old {old}, now {}", x.load(Ordering::Relaxed));

    let old = x.fetch_max(100, Ordering::Relaxed);
    println!("fetch_max(100) -> old {old}, now {}", x.load(Ordering::Relaxed));

    let flag = AtomicBool::new(false);
    let old = flag.fetch_or(true, Ordering::Relaxed);
    println!("fetch_or(true) -> old {old}, now {}", flag.load(Ordering::Relaxed));
}
```

```text
load           -> 10
store(20)      -> now 20
swap(30)       -> old 20, now 30
fetch_add(5)   -> old 30, now 35
fetch_sub(15)  -> old 35, now 20
fetch_max(100) -> old 20, now 100
fetch_or(true) -> old false, now true
```

**Что происходит:**

1. `load` читает, `store` пишет. Каждая по отдельности атомарна; вместе это два шага (потерянное обновление).
2. `swap` записывает новое значение и возвращает старое за **один** шаг.
3. Методы `fetch_…` (`fetch_add`, `fetch_sub`, `fetch_and`, `fetch_or`, `fetch_xor`, `fetch_max`, `fetch_min`) меняют значение и возвращают **старое** за один шаг. Это атомарные операции чтения-изменения-записи.
4. Возвращённое старое значение часто и есть самое полезное. `fetch_or(true)` на флаге говорит, были ли *вы* первым, кто его поставил: `old false` может получить только один поток.

Самое частое применение — глобальный счётчик в `static`. `static` живёт всю программу и виден всем потокам, и это безопасно, потому что атомарной переменной не нужен `&mut`. Создайте `src/bin/next_id.rs`:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread;

static NEXT_ID: AtomicU64 = AtomicU64::new(1);

/// Returns a new ID that no other call ever returned.
fn next_id() -> u64 {
    NEXT_ID.fetch_add(1, Ordering::Relaxed)
}

fn main() {
    thread::scope(|s| {
        for name in ["a", "b", "c"] {
            s.spawn(move || {
                let ids: Vec<u64> = (0..3).map(|_| next_id()).collect();
                println!("thread {name} got {ids:?}");
            });
        }
    });
    println!("next free id: {}", NEXT_ID.load(Ordering::Relaxed));
}
```

```text
thread a got [1, 2, 3]
thread c got [4, 5, 6]
thread b got [7, 8, 9]
next free id: 10
```

**Что происходит:** `fetch_add` возвращает старое значение и сдвигает счётчик за один шаг, поэтому никакие два вызова не получат одно и то же число, каким бы ни был тайминг. Какой поток получит какие числа, меняется от запуска к запуску; то, что каждое число встречается ровно один раз, не меняется никогда. Это проверяет Упражнение 1.

### `fetch_add` переходит через границу

Обычная целочисленная арифметика в Rust паникует при переполнении в отладочной сборке ([Урок 12.2](../12-computer-systems/02-integers-in-memory.md)). Атомарные `fetch_add` и `fetch_sub` **никогда** не паникуют: они переходят через границу и в отладочной, и в релизной сборке. Создайте `src/bin/wrap.rs`:

```rust
use std::sync::atomic::{AtomicU8, Ordering};

fn main() {
    let small = AtomicU8::new(254);
    for _ in 0..3 {
        let old = small.fetch_add(1, Ordering::Relaxed);
        println!("{old} + 1 = {}", small.load(Ordering::Relaxed));
    }
}
```

```text
254 + 1 = 255
255 + 1 = 0
0 + 1 = 1
```

У `AtomicU64`, который считает миллион в секунду, переход через границу займёт больше 500 000 лет, так что это нормально. У `AtomicU8` или `AtomicU16`, а также у `fetch_sub` на счётчике, который может дойти до нуля, это настоящая ошибка.

> 🔒 **Безопасность:** счётчик, перешедший через границу, может выдать уже занятый идентификатор или превратить «0 свободных мест» в «4 миллиарда свободных мест» после одного лишнего `fetch_sub`. Когда значение не должно переходить через границу, не используйте `fetch_add`: используйте цикл CAS с проверкой из следующего раздела (Упражнение 2).

## 3. Сравнение с обменом

`fetch_add` умеет только прибавлять. А если новое значение зависит от правила, например «прибавь 1, но не больше 3»? Для этого есть **сравнение с обменом** (CAS), в Rust — `compare_exchange`:

```text
 compare_exchange(current, new):
   in ONE atomic step:
     if value == current { value = new;  return Ok(current) }
     else                {                return Err(actual value) }
```

Вы говорите: «я думаю, что значение всё ещё `current`; если я прав, сделай его `new`». Если другой поток тем временем изменил значение, ничего не записывается, и вы узнаёте настоящее значение. Тогда вы вычисляете снова и повторяете. Это и есть **цикл CAS**.

Вот ограничитель, который допускает одновременно не больше `max` владельцев, например открытых соединений. Создайте `src/bin/limiter.rs`:

```rust
use std::sync::atomic::{AtomicU32, Ordering};
use std::thread;

/// Allows at most `max` holders at the same time.
struct Limiter {
    current: AtomicU32,
    max: u32,
}

impl Limiter {
    fn try_acquire(&self) -> bool {
        let mut seen = self.current.load(Ordering::Relaxed);
        loop {
            if seen == self.max {
                return false; // full: give up, do not wait
            }
            match self.current.compare_exchange_weak(
                seen,     // "I believe the value is still `seen`..."
                seen + 1, // "...then please make it `seen + 1`"
                Ordering::Relaxed,
                Ordering::Relaxed,
            ) {
                Ok(_) => return true,
                Err(actual) => seen = actual, // someone was faster: retry
            }
        }
    }

    fn release(&self) {
        self.current.fetch_sub(1, Ordering::Relaxed);
    }
}

fn main() {
    let limiter = Limiter {
        current: AtomicU32::new(0),
        max: 3,
    };
    let results: Vec<bool> = (0..5).map(|_| limiter.try_acquire()).collect();
    println!("five tries with max 3: {results:?}");
    limiter.release();
    println!("after one release: {}", limiter.try_acquire());

    // Many threads: try to take, hold for a moment, give back.
    let ok = AtomicU32::new(0);
    let limiter = Limiter {
        current: AtomicU32::new(0),
        max: 3,
    };
    thread::scope(|s| {
        for _ in 0..8 {
            s.spawn(|| {
                for _ in 0..10_000 {
                    if limiter.try_acquire() {
                        ok.fetch_add(1, Ordering::Relaxed);
                        limiter.release();
                    }
                }
            });
        }
    });
    println!(
        "successful acquires: {}, current at the end: {}",
        ok.load(Ordering::Relaxed),
        limiter.current.load(Ordering::Relaxed)
    );
}
```

```text
five tries with max 3: [true, true, true, false, false]
after one release: true
successful acquires: 8209, current at the end: 0
```

**Что происходит:**

1. `try_acquire` читает текущее значение в `seen`. Если оно уже равно `max`, ответ — нет.
2. Иначе он спрашивает `compare_exchange_weak(seen, seen + 1, …)`: «если значение всё ещё `seen`, сделай его `seen + 1`». Если получилось (`Ok`), место наше.
3. Если между нашим `load` и нашим CAS другой поток изменил значение, CAS не срабатывает и возвращает `Err(actual)`. Ничего не записано. Мы берём `actual` как новое `seen`, снова проверяем ограничение и повторяем. Поэтому значение никогда не превысит `max`, каким бы ни был тайминг.
4. Два аргумента `Ordering` — для успеха и для неудачи; здесь оба `Relaxed` (Урок 14.6).
5. С 8 потоками и ограничением 3 многие попытки неудачны (число успехов меняется от запуска к запуску). В конце за каждым успешным `try_acquire` последовал один `release`, поэтому счётчик снова 0.

### `compare_exchange` или `compare_exchange_weak`?

`compare_exchange_weak` может **ложно** не сработать: вернуть `Err`, хотя значение совпадало. На процессорах arm64 без LSE CAS собирается из двух инструкций (эксклюзивная загрузка и эксклюзивная запись), и эта пара может сорваться, если её что-то потревожит; слабая версия не прячет это за дополнительным внутренним циклом, поэтому она чуть быстрее. Внутри цикла повторов ложная неудача означает лишь ещё один круг, поэтому **в циклах используйте `compare_exchange_weak`**, а `compare_exchange` — когда пробуете только один раз.

### `try_update` и `update` пишут цикл за вас

У каждого атомарного целого есть два метода, внутри которых уже есть цикл CAS:

- `update(set_order, fetch_order, |old| new)` — замыкание вычисляет новое значение из старого; оно повторяется, пока CAS не сработает. Возвращает старое значение.
- `try_update(set_order, fetch_order, |old| Some(new) или None)` — то же самое, но замыкание может вернуть `None`, чтобы сдаться. Возвращает `Ok(old)` или `Err(current)`.

Замыкание может выполниться **несколько раз** (по разу на каждую повторную попытку), поэтому в нём не должно быть побочных эффектов вроде печати или добавления в вектор. Вот генератор идентификаторов, который отказывается переходить через границу, с помощью `checked_add` (он возвращает `None` при переполнении). Создайте `src/bin/next_id_checked.rs`:

```rust
use std::sync::atomic::{AtomicU8, Ordering};

static NEXT_ID: AtomicU8 = AtomicU8::new(250); // tiny type, so we can see the end

/// Returns a new ID, or None when all IDs are used up. Never wraps around.
fn next_id() -> Option<u8> {
    NEXT_ID
        .try_update(Ordering::Relaxed, Ordering::Relaxed, |id| id.checked_add(1))
        .ok()
}

fn main() {
    let ids: Vec<Option<u8>> = (0..8).map(|_| next_id()).collect();
    println!("{ids:?}");
}
```

```text
[Some(250), Some(251), Some(252), Some(253), Some(254), None, None, None]
```

**Что происходит:** `try_update` загружает значение, вызывает замыкание и пробует CAS с результатом. Когда значение 255, `checked_add(1)` возвращает `None`, поэтому `try_update` сдаётся и возвращает `Err(255)`, а `.ok()` превращает это в `None`. Значение остаётся 255 навсегда: ни один идентификатор не выдаётся дважды. (Сам идентификатор 255 не возвращается никогда — это цена такого простого решения.)

> ⚠️ **Частая ошибка:** в старом коде и статьях вы можете встретить `fetch_update`. В Rust 1.99 его переименовали: компилятор теперь предупреждает ``use of deprecated method `…::fetch_update`: renamed to `try_update` for consistency``. Используйте `try_update` (или `update`, когда замыкание никогда не сдаётся).

## 4. Ещё два приёма

**Флаг остановки.** Рабочие потоки проверяют `AtomicBool` и останавливаются, когда он поднят. Создайте `src/bin/stop_flag.rs`:

```rust
use std::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use std::thread;
use std::time::Duration;

fn main() {
    let stop = AtomicBool::new(false);
    let rounds = AtomicU64::new(0);

    thread::scope(|s| {
        for _ in 0..3 {
            s.spawn(|| {
                while !stop.load(Ordering::Relaxed) {
                    rounds.fetch_add(1, Ordering::Relaxed); // "work"
                    thread::sleep(Duration::from_millis(1));
                }
            });
        }
        thread::sleep(Duration::from_millis(100));
        stop.store(true, Ordering::Relaxed); // ask all workers to stop
    });

    println!(
        "workers stopped after about {} rounds",
        rounds.load(Ordering::Relaxed)
    );
}
```

```text
workers stopped after about 243 rounds
```

**Что происходит:** три рабочих потока крутятся в цикле, пока не увидят `true`. Через 100 мс главный поток записывает `true`, каждый рабочий поток замечает это при следующей проверке, и область видимости заканчивается, когда все трое вернулись. Здесь достаточно `Relaxed`, потому что флаг не несёт других данных: он лишь говорит «стоп». (Обработчик `ctrlc` из Урока 06.11 использовал тот же приём.) Если бы флаг ещё должен был говорить «и нужные вам результаты теперь лежат в этом векторе», ему понадобились бы порядки из Урока 14.6.

**Упаковка двух значений в одну атомарную переменную.** Две отдельные атомарные переменные — это два отдельных шага. Если хранить `count` и `sum` в двух `AtomicU64`, читатель может увидеть новый `count` со старым `sum`. Когда два небольших числа должны меняться вместе, положите их в **одну** 64-битную атомарную переменную: например, `count` в старшие 32 бита, а `sum` в младшие 32 бита, и меняйте её через `update`. Это строит Упражнение 3.

## 5. Что делает процессор

Как железо может выполнить «прочитать, прибавить, записать» без промежутка? Посмотрим на машинный код. Создайте `src/bin/asm_demo.rs`:

```rust
use std::sync::atomic::{AtomicU64, Ordering};

#[inline(never)]
pub fn plain_add(x: &mut u64) {
    *x += 1;
}

#[inline(never)]
pub fn atomic_add(x: &AtomicU64) -> u64 {
    x.fetch_add(1, Ordering::Relaxed)
}

fn main() {
    let mut a = 0;
    plain_add(&mut a);
    let b = AtomicU64::new(0);
    atomic_add(&b);
    println!("{a} {}", b.load(Ordering::Relaxed));
}
```

Попросите компилятор выдать ассемблер вместо программы, а потом найдите файл:

```bash
cargo rustc --release --bin asm_demo -- --emit asm
find target/release -name 'asm_demo.s'
```

В этом файле найдите `plain_add` и `atomic_add`. Их тела такие (длинные искажённые имена функций здесь сокращены):

```text
plain_add:
	ldr	x8, [x0]
	add	x8, x8, #1
	str	x8, [x0]
	ret

atomic_add:
	mov	w8, #1
	ldadd	x8, x8, [x0]
	ret
```

**Что происходит:**

1. `plain_add` — три инструкции: загрузить значение в регистр `x8`, прибавить 1, записать обратно. Другое ядро может записать ту же память между `ldr` и `str`: это и есть потерянное обновление из Урока 14.1, только в железе.
2. `atomic_add` кладёт 1 в `w8` и затем выполняет **одну** инструкцию `ldadd` («загрузить и прибавить») из расширения arm64 **LSE**. Ядро CPU получает кэш-линию в исключительное владение, читает старое значение в `x8`, прибавляет и записывает обратно, прежде чем любое другое ядро сможет тронуть эту линию. Старое значение в `x8` — это возвращаемое значение `fetch_add`.
3. `#[inline(never)]` оставляет обе функции отдельными, чтобы мы могли их найти; без него компилятор вставил бы их код прямо в `main`.

> ⚡ **Производительность:** атомарная операция над линией, которую не использует ни одно другое ядро, стоит примерно как обычная. Когда многие ядра долбят **одну и ту же** атомарную переменную, каждая операция должна перегонять кэш-линию между ядрами, и всё становится намного медленнее (тот же эффект, что у нагруженной блокировки в Уроке 14.2). Два совета: пусть каждый поток считает в локальной переменной и прибавляет к общей атомарной один раз в конце; и следите за **ложным разделением**: две атомарные переменные, которыми пользуются разные потоки, но которые лежат рядом в памяти, делят одну кэш-линию и тормозят друг друга. `crossbeam::utils::CachePadded<T>` дополняет значение до целой кэш-линии, чтобы этого не было.

> 💡 **Совет:** `Mutex<u64>` проще сделать правильно, чем цикл CAS, и он быстрый, когда за него не борются. Используйте атомарные операции для отдельных чисел и флагов. Когда инвариант затрагивает больше одного значения (и они не помещаются в одну атомарную переменную), используйте блокировку.

## Частые ошибки

1. **`load`, а затем `store`, чтобы изменить значение.** Это два шага: потерянное обновление. Используйте метод `fetch_…`, `update` или цикл CAS.
2. **Забыть, что `fetch_add` и `fetch_sub` переходят через границу.** Они никогда не паникуют. Если значение должно оставаться в диапазоне, используйте `try_update` с `checked_add` / `checked_sub`.
3. **Две атомарные переменные, которые должны меняться вместе.** Каждая атомарна, пара — нет. Упакуйте их в одну атомарную переменную или используйте мьютекс.
4. **Побочные эффекты внутри замыкания `update` / `try_update`.** Оно может выполниться несколько раз. Только вычисления.
5. **`Relaxed` для флага, который публикует данные.** «Поставить `ready = true` после заполнения буфера» требует `Release` и `Acquire`. Урок 14.6 показывает ошибку и исправление.

## Попробуйте сами

1. В `wrap.rs` используйте `AtomicI8`, начиная со `126`. Что идёт после 127? (Ответ: `127 + 1 = -128`, наименьшее `i8`.)
2. В `limiter.rs` замените `compare_exchange_weak` на `compare_exchange`. Программа по-прежнему работает. Затем замените весь цикл CAS в `try_acquire` одной строкой: `self.current.try_update(Ordering::Relaxed, Ordering::Relaxed, |c| (c < self.max).then_some(c + 1)).is_ok()`. Запустите снова: первые две строки те же, меняется от запуска к запуску только число успешных захватов.
3. В `stop_flag.rs` удалите строку `stop.store(true, Ordering::Relaxed);`. Программа никогда не завершается. Почему? Остановите её через `Ctrl+C`.

## Упражнения

### Упражнение 1 — Уникальные идентификаторы под нагрузкой (легко)

Напишите `src/bin/id_test.rs` с функцией `next_id` из раздела 2 и тестом `ids_are_unique_across_threads`: 8 scoped-потоков берут по 1000 идентификаторов; тест собирает их все и проверяет, что идентификаторов 8000 и *различных* идентификаторов тоже 8000.

<details>
<summary>Подсказка</summary>

Каждый поток возвращает свой `Vec<u64>`; дождитесь дескрипторов через `join` и объедините их в один вектор через `flat_map`. Положите вектор в `HashSet<u64>`: множество хранит каждое значение один раз, поэтому его длина — число различных значений.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::sync::atomic::{AtomicU64, Ordering};

static NEXT_ID: AtomicU64 = AtomicU64::new(1);

fn next_id() -> u64 {
    NEXT_ID.fetch_add(1, Ordering::Relaxed)
}

fn main() {
    println!("{} {}", next_id(), next_id());
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::HashSet;
    use std::thread;

    #[test]
    fn ids_are_unique_across_threads() {
        let all: Vec<u64> = thread::scope(|s| {
            let handles: Vec<_> = (0..8)
                .map(|_| s.spawn(|| (0..1000).map(|_| next_id()).collect::<Vec<_>>()))
                .collect();
            handles
                .into_iter()
                .flat_map(|h| h.join().unwrap())
                .collect()
        });
        let unique: HashSet<u64> = all.iter().copied().collect();
        assert_eq!(all.len(), 8000);
        assert_eq!(unique.len(), 8000, "some ID was given out twice");
    }
}
```

```text
running 1 test
test tests::ids_are_unique_across_threads ... ok
```

Почему это работает: `fetch_add` выдаёт каждое значение ровно один раз. Попробуйте сломать тест: замените `fetch_add` парой `load` + `store` из Урока 14.1. Тогда тест падает с сообщением `some ID was given out twice`, потому что два потока могут прочитать одно и то же значение.

</details>

### Упражнение 2 — Ограничитель на `try_update` (средне)

Перепишите `try_acquire` из `limiter.rs` через `try_update` вместо цикла, написанного вручную (см. «Попробуйте сами», шаг 2), и напишите тест `never_more_than_max_holders`: 8 потоков по 10 000 раз пытаются захватить место; после успешного захвата поток записывает текущее число владельцев через `fetch_max` во вторую атомарную переменную `highest`, затем освобождает место. В конце `highest` должно быть не больше `max` (3), а `current` — равно 0.

<details>
<summary>Подсказка</summary>

`(c < self.max).then_some(c + 1)` возвращает `Some(c + 1)`, когда место есть, и `None` иначе. Внутри теста, после того как `try_acquire()` вернул `true`, прочитайте `limiter.current` и вызовите `highest.fetch_max(that_value, Ordering::Relaxed)`.

</details>

<details>
<summary>Ответ</summary>

Новый метод:

```rust
    fn try_acquire(&self) -> bool {
        self.current
            .try_update(Ordering::Relaxed, Ordering::Relaxed, |c| {
                (c < self.max).then_some(c + 1)
            })
            .is_ok()
    }
```

Тест в конце `src/bin/limiter.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn never_more_than_max_holders() {
        let limiter = Limiter {
            current: AtomicU32::new(0),
            max: 3,
        };
        let highest = AtomicU32::new(0);
        thread::scope(|s| {
            for _ in 0..8 {
                s.spawn(|| {
                    for _ in 0..10_000 {
                        if limiter.try_acquire() {
                            let now = limiter.current.load(Ordering::Relaxed);
                            highest.fetch_max(now, Ordering::Relaxed);
                            limiter.release();
                        }
                    }
                });
            }
        });
        assert!(highest.load(Ordering::Relaxed) <= 3);
        assert_eq!(limiter.current.load(Ordering::Relaxed), 0);
    }
}
```

```text
running 1 test
test tests::never_more_than_max_holders ... ok
```

Почему это работает: `try_update` — тот же цикл CAS: замыкание по увиденному значению решает, есть ли место, а CAS записывает `c + 1`, только если с тех пор никто не изменил значение. Поэтому `current` никогда не превысит `max`. `fetch_max` хранит наибольшее значение, которое видел любой поток; если бы ограничитель был сломан (например, `fetch_add` с проверкой *после* него), `highest` иногда доходил бы до 4 и больше.

</details>

### Упражнение 3 — Количество и сумма в одной атомарной переменной (сложно)

Напишите `src/bin/stats.rs` с типом `Stats`, который записывает значения (`record(&self, value: u32)`) и возвращает согласованную пару `snapshot(&self) -> (count, sum)`. Храните оба числа в **одном** `AtomicU64`: количество в старших 32 битах, сумму в младших 32 битах. `main` записывает 10, 20 и 30 и печатает количество, сумму и среднее. Тест: 4 потока по десять тысяч раз записывают значение 5, а пятый поток делает 10 000 снимков и каждый раз проверяет, что `sum == count * 5`. В конце снимок равен `(40000, 200000)`.

<details>
<summary>Подсказка</summary>

Упаковка: `(count << 32) | sum`. Распаковка: `count = p >> 32`, `sum = p & 0xFFFF_FFFF`. Меняйте упакованное значение через `update`: его замыкание получает старое упакованное значение и возвращает новое. Почему две отдельные атомарные переменные (`count` и `sum`) провалили бы проверку читателя?

</details>

<details>
<summary>Ответ</summary>

```rust
use std::sync::atomic::{AtomicU64, Ordering};

/// Count (high 32 bits) and sum (low 32 bits) packed into ONE atomic,
/// so a reader always sees a matching pair.
struct Stats {
    packed: AtomicU64,
}

impl Stats {
    fn new() -> Self {
        Stats {
            packed: AtomicU64::new(0),
        }
    }

    fn record(&self, value: u32) {
        self.packed
            .update(Ordering::Relaxed, Ordering::Relaxed, |p| {
                let count = (p >> 32) + 1;
                let sum = (p & 0xFFFF_FFFF) + value as u64;
                assert!(sum <= u32::MAX as u64, "sum does not fit in 32 bits");
                (count << 32) | sum
            });
    }

    /// Returns (count, sum), always from the same moment.
    fn snapshot(&self) -> (u64, u64) {
        let p = self.packed.load(Ordering::Relaxed);
        (p >> 32, p & 0xFFFF_FFFF)
    }
}

fn main() {
    let stats = Stats::new();
    for v in [10, 20, 30] {
        stats.record(v);
    }
    let (count, sum) = stats.snapshot();
    println!("count = {count}, sum = {sum}, average = {}", sum / count);
}

#[cfg(test)]
mod tests {
    use super::*;
    use std::thread;

    #[test]
    fn readers_always_see_a_matching_pair() {
        let stats = Stats::new();
        thread::scope(|s| {
            for _ in 0..4 {
                s.spawn(|| {
                    for _ in 0..10_000 {
                        stats.record(5); // every value is 5...
                    }
                });
            }
            s.spawn(|| {
                for _ in 0..10_000 {
                    let (count, sum) = stats.snapshot();
                    assert_eq!(sum, count * 5); // ...so sum is always 5 * count
                }
            });
        });
        assert_eq!(stats.snapshot(), (40_000, 200_000));
    }
}
```

```text
count = 3, sum = 60, average = 20
```

`cargo test --bin stats` показывает `test tests::readers_always_see_a_matching_pair ... ok`.

Почему это работает: один `update` меняет количество и сумму одним CAS, а один `load` читает оба числа, поэтому читатель никогда не увидит количество из одного момента с суммой из другого. С двумя атомарными переменными писатель делает `count += 1`, а потом `sum += 5` — двумя шагами, и читатель, выполнившийся между ними, видит `sum == (count - 1) * 5`: проверка падает. Цена упаковки: у каждой половины всего 32 бита, поэтому сумма должна оставаться меньше примерно 4,29 миллиарда (это проверяет `assert!`; настоящая программа использовала бы блокировку или 128-битное решение).

</details>

## Вопросы для самопроверки

1. Почему `AtomicU64` можно менять через `&self`, а `u64` требует `&mut`?

<details>
<summary>Ответ</summary>

Атомарные типы построены на `UnsafeCell` (внутренняя изменяемость), и каждое изменение — атомарная операция железа, поэтому изменение через общую ссылку не может вызвать гонку данных. Поэтому они `Sync` и их могут делить многие потоки: в `Arc`, через заимствование или в `static`.

</details>

2. Что возвращает `fetch_add` и чем это полезно?

<details>
<summary>Ответ</summary>

Значение **до** сложения. Поскольку чтение и сложение происходят за один шаг, возвращённое значение уникально для этого вызова: так генератор идентификаторов выдаёт каждому вызывающему другое число.

</details>

3. Что происходит, когда `AtomicU8`, в котором лежит 255, получает `fetch_add(1)`?

<details>
<summary>Ответ</summary>

Он переходит через границу в 0 и возвращает 255. Атомарный `fetch_add` никогда не паникует при переполнении, даже в отладочной сборке.

</details>

4. Опишите цикл CAS своими словами.

<details>
<summary>Ответ</summary>

Прочитать текущее значение; вычислить из него новое; вызвать `compare_exchange(old, new)`. Если значение всё ещё `old`, новое значение записано, и работа сделана. Если другой поток его изменил, вы получаете настоящее значение, вычисляете заново из него и повторяете.

</details>

5. Почему `compare_exchange_weak` подходит для цикла?

<details>
<summary>Ответ</summary>

Он может ложно не сработать (вернуть `Err`, хотя значение совпадало), но в цикле это стоит лишь одного лишнего круга. Взамен он может быть чуть быстрее на процессорах, где CAS собирается из пары «эксклюзивная загрузка / эксклюзивная запись».

</details>

6. Какой одной инструкцией становится `fetch_add(1, Relaxed)` над `AtomicU64` на чипе Apple M и чем это отличается от `*x += 1`?

<details>
<summary>Ответ</summary>

`ldadd` из расширения arm64 LSE: одна инструкция, которая загружает, прибавляет и записывает, пока ядро держит кэш-линию в исключительном владении. `*x += 1` — три отдельные инструкции (`ldr`, `add`, `str`), и между ними может записать другое ядро.

</details>

## Итоги

- Атомарная операция неделима: другие потоки видят её как не начатую или завершённую. В Rust есть `AtomicBool`, атомарные целые и `AtomicPtr`; они меняются через `&self` и являются `Sync`.
- `load`/`store` — отдельные шаги; `swap`, `fetch_add`, `fetch_sub`, `fetch_or`, `fetch_max` и их собратья — атомарные операции чтения-изменения-записи, возвращающие старое значение.
- `fetch_add` и `fetch_sub` молча переходят через границу. Используйте `try_update` с `checked_add`, когда значение должно оставаться в диапазоне.
- `compare_exchange` записывает, только если значение всё ещё то, которого вы ждёте. Цикл CAS повторяет попытку до успеха; `update` и `try_update` (новые имена `fetch_update`) пишут цикл за вас.
- `Relaxed` достаточно, когда атомарная переменная — просто число или флаг остановки. Два значения, которые должны меняться вместе, требуют одной упакованной атомарной переменной или блокировки.
- На arm64 `fetch_add` — одна инструкция `ldadd`. Нагруженные атомарные переменные медленные, потому что кэш-линия перемещается между ядрами; остерегайтесь ложного разделения.

## Источники и что почитать

- [std::sync::atomic](https://doc.rust-lang.org/std/sync/atomic/index.html) — документация модуля: все атомарные типы и краткое введение в порядки.
- [AtomicU64 в стандартной библиотеке](https://doc.rust-lang.org/std/sync/atomic/type.AtomicU64.html) — все методы, включая `compare_exchange_weak`, `update` и `try_update`. (В Rust 1.100 `AtomicU64` определён как `Atomic<u64>` — обобщённый тип, который стабильный код пока не может назвать напрямую; поэтому сообщения компилятора пишут `Atomic::<u64>`.)
- [Rust Atomics and Locks, chapter 2 — Atomics](https://marabos.nl/atomics/atomics.html) — Mara Bos объясняет load, store, операции «получить и изменить» и сравнение с обменом на множестве небольших примеров.
- [Rust Atomics and Locks, chapter 7 — Understanding the Processor](https://marabos.nl/atomics/hardware.html) — как атомарные операции отображаются на инструкции arm64 и x86-64 и почему важны кэш-линии.
- [crossbeam_utils::CachePadded на docs.rs](https://docs.rs/crossbeam-utils/latest/crossbeam_utils/struct.CachePadded.html) — тип с дополнением, который предотвращает ложное разделение.

---

[🏠 Начало](../START-HERE.md) · [← 14.4 Пулы потоков, scoped-потоки и rayon](04-thread-pools-scoped-threads-and-rayon.md) · [📋 Этап 14](README.md) · [14.6 Порядок памяти →](06-memory-ordering.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
