<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 14.5 Атомарные операции](05-atomics.md) · [📋 Этап 14](README.md) · [14.7 Пишем спин-блокировку →](07-building-a-spin-lock.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/14-concurrency-and-async/06-memory-ordering.md) <!-- nav -->

# Урок 14.6 — Порядок памяти

> **Этап 14 — Конкурентность и асинхронность подробно** · Урок 6 из 16 · ⏱ около 135 минут

## Что вы узнаете

- Почему другие потоки могут видеть ваши записи в другом порядке, чем вы их сделали, и почему ваш Mac на arm64 делает это чаще многих других машин.
- Что значит *«происходит-до»* (happens-before) и как его создают запись `Release` и чтение `Acquire`.
- Что добавляет `SeqCst` и тот единственный классический случай, когда он действительно нужен.
- Во что превращаются `Relaxed`, `Release`, `Acquire` и `SeqCst` на процессоре вашего Mac.
- Как крейт `loom` находит ошибку порядка, которую пропускает обычный тест, и как выбирать порядок.

## Перед началом

- Вы прошли [Урок 14.5 — Атомарные операции](05-atomics.md). Вы знаете `load`, `store`, `fetch_add`, `compare_exchange` и то, что мы везде использовали `Ordering::Relaxed`.
- Вы знаете кэши и кэш-линии из [Урока 12.9](../12-computer-systems/09-memory-hierarchy-and-caches.md) и умеете читать несколько инструкций arm64 ([Урок 12.7](../12-computer-systems/07-reading-assembly.md)).
- Продолжайте работать в `~/projects/rust/code/p14-concurrency`. Для раздела 6 добавьте `loom` как *dev-зависимость* (крейт, который используют только тесты):

```bash
cargo add --dev loom@0.7
```

## Новые слова

| Слово | Значение |
|---|---|
| **переупорядочивание** (reordering) | Операции с памятью становятся видны другим потокам в другом порядке, чем их записала программа. Это разрешено и компилятору, и CPU (Central Processing Unit, центральный процессор) — в рамках правил. |
| **модель памяти** (memory model) | Правила языка, которые говорят, какие значения может вернуть чтение, когда потоки делят память. Rust использует ту же модель, что и C++. |
| **сильно / слабо упорядоченный** (strongly / weakly ordered) | *Сильно упорядоченный* CPU переупорядочивает очень мало (x86-64). *Слабо упорядоченный* CPU (arm64, процессор вашего Mac) переупорядочивает гораздо больше. |
| **x86-64** | 64-битное семейство процессоров большинства ПК (персональных компьютеров) и многих серверов; их выпускают Intel и AMD. |
| **буфер записи** (store buffer) | Небольшая очередь внутри каждого ядра CPU, где записи ждут, прежде чем попасть в кэш, который видят другие ядра. |
| **«происходит-до»** (happens-before) | Гарантия между двумя операциями: всё, что сделала первая, видно второй. |
| **`Release`** | Порядок для записи: все операции с памятью до неё в этом потоке становятся видимыми не позже этой записи. |
| **`Acquire`** | Порядок для чтения: если оно прочитало значение, записанное записью `Release`, то после этого чтения видно всё, что было до той записи. |
| **синхронизируется-с** (synchronizes-with) | Связь, которую создаёт чтение `Acquire`, прочитавшее значение записи `Release`. Она даёт «происходит-до» между двумя потоками. |
| **`AcqRel`** | `Acquire` и `Release` вместе, для операций чтения-изменения-записи вроде `fetch_add` или `compare_exchange`. |
| **`SeqCst`** (sequentially consistent, последовательно согласованный) | Самый сильный порядок: все операции `SeqCst` всех потоков выстраиваются в **один** порядок, с которым согласны все потоки. |
| **барьер** (fence) | `std::sync::atomic::fence(ordering)`: правило порядка без собственного обращения к памяти. |
| **лакмусовый тест** (litmus test) | Крошечная программа из двух-трёх потоков, которая проверяет, может ли случиться одно конкретное переупорядочивание. |
| **проверка моделей** (model checker) | Инструмент, который запускает небольшую конкурентную программу на каждом допустимом чередовании и переупорядочивании. `loom` — такой инструмент; подробно его изучает Урок 14.9. |

## 1. Сюрприз: ваши записи путешествуют отдельно

Вот приём, который вы будете писать много раз: один поток готовит **данные**, затем ставит **флаг** «данные готовы»; другой поток ждёт флаг и затем читает данные.

```text
 writer                      reader
 DATA = 42                   if READY == true:
 READY = true                    use DATA      ← expects 42
```

Выглядит очевидно правильным. Но это неверно, если не выбрать порядки внимательно, потому что **ничто не обещает, что другие потоки увидят две записи в порядке программы**:

- **Компилятор** может поменять местами две записи в разные переменные или два чтения, если сам этот поток этого не заметит. Он делает это ради скорости.
- **CPU** тоже так делает. У каждого ядра есть **буфер записи**: записи ждут там, прежде чем попасть в кэш, который читают другие ядра. Слабо упорядоченный CPU может выпустить из буфера более позднюю запись (`READY`) раньше более ранней (`DATA`), например потому что кэш-линия `READY` уже была в ядре, а линия `DATA` — нет. Чтения тоже могут выполняться раньше.

Представьте два письма, отправленные одно за другим из двух разных почтовых отделений. Вы отправили первое письмо первым, но никто не обещает, что оно придёт первым.

Внутри **одного** потока всё по-прежнему выглядит по порядку — это обещано. Вопрос лишь в том, что видят **другие** потоки. Ответ для `Relaxed`: никаких обещаний о *других* переменных.

## 2. Увидеть это на своём Mac

**Лакмусовый тест** повторяет этот приём снова и снова и считает плохие исходы. Два потока должны начинать каждый раунд одновременно, поэтому программа использует два дополнительных счётчика для начала и конца раундов; интересная часть помечена комментариями от 1 до 4. Создайте `src/bin/message_passing.rs`:

```rust
use std::hint::spin_loop;
use std::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use std::thread;

/// Keeps a value alone in its own 128-byte block (a cache line on Apple M chips).
#[repr(align(128))]
struct Padded<T>(T);

static DATA: Padded<AtomicU64> = Padded(AtomicU64::new(0));
static READY: Padded<AtomicBool> = Padded(AtomicBool::new(false));

const ROUNDS: u64 = 100_000;

/// Waits until `counter` has reached `r`.
fn wait_for(counter: &AtomicU64, r: u64) {
    while counter.load(Ordering::SeqCst) < r {
        spin_loop();
    }
}

/// In every round the writer stores DATA = r and then READY = true, while the
/// reader looks at READY and, if it is true, at DATA. Returns how often the
/// reader saw READY == true but an OLD value in DATA.
fn run(flag_store: Ordering, flag_load: Ordering) -> u64 {
    let start = AtomicU64::new(0); // the round that may start now
    let writer_done = AtomicU64::new(0);
    let reader_done = AtomicU64::new(0);
    thread::scope(|s| {
        s.spawn(|| {
            for r in 1..=ROUNDS {
                wait_for(&start, r);
                DATA.0.store(r, Ordering::Relaxed); // 1. write the data
                READY.0.store(true, flag_store); // 2. say "it is ready"
                writer_done.store(r, Ordering::SeqCst);
            }
        });
        let reader = s.spawn(|| {
            let mut stale = 0;
            for r in 1..=ROUNDS {
                wait_for(&start, r);
                let ready = READY.0.load(flag_load); // 3. is it ready?
                let data = DATA.0.load(Ordering::Relaxed); // 4. read the data
                if ready && data != r {
                    stale += 1; // "ready", but the data is old!
                }
                reader_done.store(r, Ordering::SeqCst);
            }
            stale
        });
        for r in 1..=ROUNDS {
            READY.0.store(false, Ordering::SeqCst); // reset for the new round
            start.store(r, Ordering::SeqCst);
            wait_for(&writer_done, r);
            wait_for(&reader_done, r);
        }
        reader.join().unwrap()
    })
}

fn main() {
    let relaxed = run(Ordering::Relaxed, Ordering::Relaxed);
    println!("Relaxed flag        : old data seen {relaxed} times in {ROUNDS} rounds");
    let rel_acq = run(Ordering::Release, Ordering::Acquire);
    println!("Release/Acquire flag: old data seen {rel_acq} times in {ROUNDS} rounds");
}
```

`cargo run --release --bin message_passing`, три запуска:

```text
Relaxed flag        : old data seen 3685 times in 100000 rounds
Release/Acquire flag: old data seen 0 times in 100000 rounds
Relaxed flag        : old data seen 8935 times in 100000 rounds
Release/Acquire flag: old data seen 0 times in 100000 rounds
Relaxed flag        : old data seen 3093 times in 100000 rounds
Release/Acquire flag: old data seen 0 times in 100000 rounds
```

**Что происходит:**

1. `#[repr(align(128))]` помещает `DATA` и `READY` в разные кэш-линии, чтобы CPU мог перемещать их независимо, как две несвязанные переменные в настоящей программе.
2. Главный поток начинает раунд `r`: сбрасывает `READY` и ставит `start = r`; затем ждёт, пока оба потока сообщат, что закончили раунд. `wait_for` крутится в цикле с `spin_loop()` — подсказкой для CPU, что мы активно ждём (её объясняет Урок 14.7). Вспомогательные счётчики используют `SeqCst`, самый сильный порядок, поэтому они не могут помешать эксперименту.
3. В каждом раунде писатель записывает `DATA = r`, а затем `READY = true`; в тот же момент читатель читает `READY`, а затем `DATA`.
4. С `Relaxed` на флаге читатель тысячи раз видел `READY == true` вместе с `DATA` **предыдущего** раунда — в нескольких процентах раундов. Ваши числа будут другими, но на чипе Apple M они редко бывают нулём. То же самое происходит и в отладочной сборке (`cargo run --bin message_passing`), которая ничего не оптимизирует, — значит, здесь переупорядочивает именно **CPU**.
5. С `Release` на записи флага и `Acquire` на его чтении плохой исход **никогда** не случается. Раздел 3 объясняет почему.

> ⚠️ **Частая ошибка:** «у меня работает, значит, порядок правильный». На ПК с x86-64 эта программа с `Relaxed` почти никогда не показала бы старые данные, потому что железо x86-64 сохраняет порядок записей и порядок чтений. Но программа и там неверна: её может переупорядочить *компилятор*, а следующий CPU может оказаться не таким добрым. Пишите код, корректный по правилам Rust, а не по вашему нынешнему железу. Ваш слабо упорядоченный Mac — хорошее место, чтобы находить такие ошибки.

## 3. «Происходит-до»: `Release` и `Acquire`

Правила модели памяти построены на одной идее: **«происходит-до»**. Если операция A происходит-до операции B, то B видит всё, что сделала A (и всё, что было до A).

Внутри одного потока каждая операция происходит-до следующей в порядке программы. Между потоками «происходит-до» создаётся парой:

- **запись `Release`**: «сделай всё, что я сделал до этой записи, видимым не позже самой записи»;
- **чтение `Acquire`**, которое **прочитало значение этой записи**: «после этого чтения я вижу всё, что другой поток сделал до своего release».

Когда чтение acquire читает значение, записанное с release, эти две операции **синхронизируются друг с другом**, и всё, что было до release, происходит-до всего, что после acquire:

```text
 writer thread                              reader thread
 ─────────────                              ─────────────
 DATA.store(42, Relaxed)                    
        │  (program order)                  
 READY.store(true, Release) ─ ─ ─ ─ ─ ─ ─▶ READY.load(Acquire) reads true
                         synchronizes-with          │  (program order)
                                            DATA.load(Relaxed) → must see 42
```

Поэтому версия с release/acquire ни разу не увидела старых данных. Обратите внимание на три детали:

1. Сам `DATA` может оставаться `Relaxed`. Флаг несёт порядок для всего, что было до него.
2. Гарантия существует, только **если чтение acquire прочитало записанное с release значение**. Если читатель видит `READY == false`, он ничего не узнаёт о `DATA` и не должен им пользоваться. Поэтому: всегда сначала проверяйте флаг, потом читайте данные.
3. Это пара. Запись `Release` с чтением `Relaxed` или запись `Relaxed` с чтением `Acquire` не дают никакой гарантии.

Именно так мьютекс устроен внутри: `lock()` заканчивается операцией **acquire**, а `unlock()` (уничтожение guard-объекта) — это операция **release**. Поэтому всё, что один поток сделал под блокировкой, происходит-до всего, что сделает следующий владелец. Вот почему данные внутри `Mutex` всегда видны правильно и почему защёлка из Урока 14.3 (Упражнение 3) позволила главному потоку увидеть обновления счётчика. В Уроке 14.7 вы сами напишете эти два порядка.

Для операций чтения-изменения-записи (`swap`, `fetch_add`, `compare_exchange`) есть **`AcqRel`**: часть чтения — `Acquire`, часть записи — `Release`. Используйте его, когда одна операция должна и получить данные, и передать их дальше, например счётчик, где последний пришедший поток читает результаты, опубликованные остальными.

> 📏 **Соглашение:** рядом с каждым порядком, кроме `Relaxed`, пишите короткий комментарий, который называет его пару: `// Release: pairs with the Acquire load in take()`. Порядки без пары — признак ошибки.

## 4. `SeqCst`: один порядок для всех

Release/acquire — это про **пары** потоков. Некоторым алгоритмам нужно больше. Вот классический случай, он называется *буферизация записи* (store buffering): поток A ставит `X = 1`, а затем читает `Y`; поток B ставит `Y = 1`, а затем читает `X`. Могут ли **оба** прочитать 0? В порядке программы это выглядит невозможным: кто выполняется вторым, должен увидеть запись первого. Создайте `src/bin/store_buffer.rs`:

```rust
use std::hint::spin_loop;
use std::sync::atomic::{AtomicU64, Ordering};
use std::thread;

#[repr(align(128))]
struct Padded<T>(T);

static X: Padded<AtomicU64> = Padded(AtomicU64::new(0));
static Y: Padded<AtomicU64> = Padded(AtomicU64::new(0));

const ROUNDS: u64 = 100_000;

fn wait_for(counter: &AtomicU64, r: u64) {
    while counter.load(Ordering::SeqCst) < r {
        spin_loop();
    }
}

/// Thread A: X = 1, then read Y.   Thread B: Y = 1, then read X.
/// Returns how often BOTH threads read 0.
fn run(store: Ordering, load: Ordering) -> u64 {
    let start = AtomicU64::new(0);
    let a_done = AtomicU64::new(0);
    let b_done = AtomicU64::new(0);
    let (seen_a, seen_b) = (AtomicU64::new(0), AtomicU64::new(0));
    let mut both_zero = 0;
    thread::scope(|s| {
        s.spawn(|| {
            for r in 1..=ROUNDS {
                wait_for(&start, r);
                X.0.store(1, store);
                seen_a.store(Y.0.load(load), Ordering::SeqCst);
                a_done.store(r, Ordering::SeqCst);
            }
        });
        s.spawn(|| {
            for r in 1..=ROUNDS {
                wait_for(&start, r);
                Y.0.store(1, store);
                seen_b.store(X.0.load(load), Ordering::SeqCst);
                b_done.store(r, Ordering::SeqCst);
            }
        });
        for r in 1..=ROUNDS {
            X.0.store(0, Ordering::SeqCst);
            Y.0.store(0, Ordering::SeqCst);
            start.store(r, Ordering::SeqCst);
            wait_for(&a_done, r);
            wait_for(&b_done, r);
            if seen_a.load(Ordering::SeqCst) == 0 && seen_b.load(Ordering::SeqCst) == 0 {
                both_zero += 1;
            }
        }
    });
    both_zero
}

fn main() {
    let relaxed = run(Ordering::Relaxed, Ordering::Relaxed);
    println!("Relaxed        : both read 0 in {relaxed} of {ROUNDS} rounds");
    let rel_acq = run(Ordering::Release, Ordering::Acquire);
    println!("Release/Acquire: both read 0 in {rel_acq} of {ROUNDS} rounds");
    let seq_cst = run(Ordering::SeqCst, Ordering::SeqCst);
    println!("SeqCst         : both read 0 in {seq_cst} of {ROUNDS} rounds");
}
```

`cargo run --release --bin store_buffer`, два запуска:

```text
Relaxed        : both read 0 in 31199 of 100000 rounds
Release/Acquire: both read 0 in 8 of 100000 rounds
SeqCst         : both read 0 in 0 of 100000 rounds
Relaxed        : both read 0 in 31251 of 100000 rounds
Release/Acquire: both read 0 in 53 of 100000 rounds
SeqCst         : both read 0 in 0 of 100000 rounds
```

**Что происходит:**

1. С `Relaxed` оба потока читают 0 примерно в 30 % раундов. Запись каждого потока ждёт в буфере записи его ядра, а его чтение уже берёт старое значение из кэша.
2. С `Release`/`Acquire` это всё ещё случается (редко, но случается). Так можно: запись release упорядочивает только то, что было **до** неё, а чтение acquire — только то, что идёт **после** него. Ничто не запрещает более позднему чтению acquire выполниться раньше более ранней записи release. К тому же ни один поток здесь не читает значение, записанное другим с release, поэтому связи «синхронизируется-с» нет вообще.
3. С `SeqCst` это не случается никогда. `SeqCst` добавляет одно правило: все операции `SeqCst` всех потоков образуют **один-единственный порядок**, с которым согласны все потоки. В этом порядке одна из двух записей идёт первой, а чтение другого потока идёт после неё, поэтому хотя бы один поток обязан увидеть 1.

Когда нужен `SeqCst`? Когда корректность зависит от того, что несколько потоков согласны о порядке операций над **разными** переменными, как здесь («я поднял свой флаг, значит, могу проверить твой»). На практике это редкость: флаги с данными, блокировки, счётчики ссылок и каналы работают с release/acquire. Многие ставят `SeqCst` «на всякий случай»; это всегда корректно, но прячет рассуждение и может быть медленнее.

`std::sync::atomic::fence(ordering)` — порядок без собственного обращения к памяти. `fence(SeqCst)` между записью и чтением в каждом потоке выше тоже запрещает исход «оба нуля»; вы используете его в Упражнении 3.

## 5. Что это значит на вашем Mac

Во что превращается каждый порядок на arm64? Вот что выдал компилятор для маленьких функций, которые делают один `store` или один `load` над `AtomicU64` с каждым порядком (получено через `--emit asm`, как в Уроке 14.5; полная программа — в «Попробуйте сами», шаг 3):

| Rust | Инструкция arm64 | Смысл |
|---|---|---|
| `store(…, Relaxed)` | `str` | обычная запись; может стать видимой позже более поздних записей |
| `store(…, Release)` | `stlr` | «запись-освобождение»: все более ранние чтения и записи видны до неё |
| `store(…, SeqCst)` | `stlr` | та же инструкция |
| `load(Relaxed)` | `ldr` | обычное чтение; может выполниться раньше более ранних чтений |
| `load(Acquire)` | `ldapr` | «чтение-захват»: ни одно более позднее чтение или запись не выполнится раньше него |
| `load(SeqCst)` | `ldar` | как `ldapr`, и ещё не может переместиться раньше более ранней `stlr` |

Так что порядки — это **настоящие инструкции** с настоящей ценой, а не просто заметки для компилятора. Разница между `ldapr` и `ldar` — ровно случай буферизации записи из раздела 4: `ldapr` может выполниться раньше более ранней `stlr` того же потока, а `ldar` — нет.

> ⚡ **Производительность:** на arm64 `Relaxed` самый дешёвый, release/acquire стоит чуть дороже, а `SeqCst` может стоить ещё дороже, потому что запрещает больше переупорядочиваний. Не оптимизируйте порядки наугад: сначала сделайте код правильным с понятными парами release/acquire, затем измерьте (Урок 05.12), прежде чем что-то ослаблять.

## 6. Проверка порядков с loom

Лакмусовым программам понадобилось 100 000 раундов и везение, чтобы показать ошибку. Обычный тест приёма с флагом обычно прошёл бы. **loom** не полагается на везение: он снова и снова запускает небольшой тест, заставляя каждый запуск идти по другому **чередованию**, а для атомарных операций ещё и позволяет чтениям возвращать любое более старое значение, которое допускает модель памяти. Внутри `loom::model(|| { … })` вы используете собственные версии `Arc`, `thread` и атомарных типов из loom.

Вот первый взгляд (Урок 14.9 показывает, как подключить loom для целого крейта). Создайте файл интеграционного теста `tests/loom_flag.rs`:

```rust
use loom::sync::Arc;
use loom::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use loom::thread;

/// The writer stores the data and then the flag; the reader checks both.
fn publish(flag_store: Ordering, flag_load: Ordering) {
    loom::model(move || {
        let data = Arc::new(AtomicU64::new(0));
        let ready = Arc::new(AtomicBool::new(false));

        let (d, r) = (Arc::clone(&data), Arc::clone(&ready));
        let writer = thread::spawn(move || {
            d.store(42, Ordering::Relaxed);
            r.store(true, flag_store);
        });

        if ready.load(flag_load) {
            assert_eq!(data.load(Ordering::Relaxed), 42, "ready, but old data");
        }
        writer.join().unwrap();
    });
}

#[test]
fn relaxed_flag() {
    publish(Ordering::Relaxed, Ordering::Relaxed);
}

#[test]
fn release_acquire_flag() {
    publish(Ordering::Release, Ordering::Acquire);
}
```

Запускайте в релизном режиме (loom прогоняет модель много раз, поэтому любит оптимизированный код): `cargo test --release --test loom_flag`.

```text
running 2 tests
test relaxed_flag ... FAILED
test release_acquire_flag ... ok

failures:

---- relaxed_flag stdout ----

thread 'relaxed_flag' (5774265) panicked at tests/loom_flag.rs:18:13:
assertion `left == right` failed: ready, but old data
  left: 0
 right: 42
```

**Что происходит:**

1. `loom::model` запускает замыкание много раз. В каждом запуске loom решает, какой поток когда выполняется и какое допустимое значение возвращает каждое атомарное чтение.
2. С `Relaxed` loom нашёл запуск, в котором читатель увидел `ready == true`, но `data == 0`, — сразу и на любой машине, даже на той, где железо никогда бы этого не показало.
3. С `Release`/`Acquire` ни один допустимый запуск не нарушает проверку, поэтому тест проходит.

> 🧪 **Тестирование:** используйте loom для небольших критичных частей конкурентного кода: флага, блокировки, очереди с двумя потоками. Держите модели крошечными (два-три потока, несколько операций), потому что число запусков растёт очень быстро. Одно ограничение, которое надо знать: loom обращается с чтениями и записями `SeqCst` как с `AcqRel`, поэтому может поднять ложную тревогу для кода, который полагается на обращения `SeqCst` (как в разделе 4). `fence(SeqCst)` моделируется правильно.

## 7. Как выбрать порядок

| Ситуация | Порядок |
|---|---|
| Счётчик, идентификатор, статистика, флаг остановки без данных | `Relaxed` |
| Флаг или указатель «данные готовы» | запись `Release`, чтение `Acquire` |
| Захват блокировки / освобождение блокировки | `Acquire` при захвате, `Release` при освобождении (Урок 14.7) |
| Чтение-изменение-запись, которая и получает, и публикует данные | `AcqRel` |
| Несколько потоков должны согласиться о порядке операций над разными переменными | `SeqCst` (или `fence(SeqCst)`) |
| Вы не уверены | используйте `Mutex` или `SeqCst` с комментарием «TODO: check» и тестом loom |

Лучший порядок часто — никакой: используйте `Mutex`, канал или `OnceLock`, внутри которых уже правильные порядки. Пишите собственный код на атомарных операциях, только когда измерение показывает, что простые инструменты слишком медленные, и тогда следуйте книге *Rust Atomics and Locks* Mara Bos (бесплатно в интернете; см. источники).

## Частые ошибки

1. **Флаг `Relaxed`, который публикует данные.** Читатель может увидеть флаг, но старые данные (раздел 2). Используйте запись `Release` и чтение `Acquire`.
2. **Только одна сторона пары.** `Release` на записи, но `Relaxed` на чтении (или наоборот) не даёт гарантии.
3. **Чтение данных без проверки флага.** Гарантия действует, только если чтение acquire *увидело* записанное с release значение.
4. **Доверие к железу.** Пропущенный порядок может никогда не проявиться на x86-64 и лишь изредка — на arm64. Код всё равно неверен. Проверяйте с loom.
5. **`SeqCst` везде, не задумываясь.** Это корректно, но прячет замысел; нельзя понять, какие операции действительно нужно упорядочить.

## Попробуйте сами

1. Запустите `message_passing` несколько раз в отладочном и релизном режиме. Затем удалите `#[repr(align(128))]` (тогда обе статические переменные могут оказаться в одной кэш-линии) и запустите снова. На тестовой машине старые данные тогда не появились ни разу (0 за три запуска). Почему важна кэш-линия? (Подсказка: между ядрами путешествует целая кэш-линия.)
2. В `message_passing.rs` замените на `Acquire` только **чтение** флага, а запись оставьте `Relaxed`. Появляются ли старые данные? Затем сделайте наоборот (запись `Release`, чтение `Relaxed`). На тестовой машине первый вариант показал старые данные 0 или 1 раз на 100 000 раундов, второй — от 1300 до 3500 раз. Оба неверны: половина пары не даёт гарантии, даже когда ошибка редкая.
3. Напишите `src/bin/orderings_asm.rs` с шестью функциями `#[inline(never)] pub fn`, каждая из которых делает один `store(1, …)` или один `load(…)` над `&AtomicU64` с `Relaxed`, `Release`/`Acquire` и `SeqCst`; вызовите все шесть из `main`. Получите ассемблер, как в Уроке 14.5, и сравните с таблицей из раздела 5.

## Упражнения

### Упражнение 1 — Выберите порядок (легко)

Для каждого случая выберите самый слабый корректный порядок (или порядки) и объясните почему:

1. Веб-сервер считает запросы через `REQUESTS.fetch_add(1, ?)`; поток статистики печатает это число каждую секунду.
2. Рабочий поток заполняет `Vec` внутри `Mutex`, затем ставит `DONE.store(true, ?)`; главный поток крутится на `DONE.load(?)`, а затем блокирует мьютекс, чтобы прочитать `Vec`.
3. Рабочий поток записывает результаты в три значения `AtomicU64`, затем ставит `READY.store(true, ?)`; главный поток проверяет `READY.load(?)`, а затем читает три значения.
4. Флаг `stop`, который говорит рабочим потокам заканчивать; он не несёт данных.

<details>
<summary>Подсказка</summary>

В каждом случае спросите себя: *публикует* ли флаг другие данные, которыми воспользуется читатель, и не защищены ли эти данные чем-то другим?

</details>

<details>
<summary>Ответ</summary>

1. `Relaxed`. Счётчик — просто число; никто не использует его, чтобы читать другие данные.
2. Для флага здесь достаточно `Relaxed`, потому что `Vec` читается под `Mutex`, и его упорядочивает собственная пара acquire/release мьютекса. (`Release`/`Acquire` на флаге не повредили бы.)
3. `Release` для записи и `Acquire` для чтения. Флаг публикует три значения; с `Relaxed` главный поток мог бы увидеть `READY == true` и старые значения.
4. `Relaxed`. Он говорит только «стоп»; публиковать нечего.

</details>

### Упражнение 2 — Тест loom для опубликованной пачки (средне)

Напишите `tests/loom_batch.rs` с типом `Batch`: три значения `AtomicU64` и `AtomicBool` `ready` (все типы из loom). `fill(&self)` записывает 1, 2 и 3 с `Relaxed`, затем ставит `ready`. `sum(&self) -> Option<u64>` возвращает `None`, если `ready` ложно, иначе сумму. В тесте loom один поток вызывает `fill`, а главный поток вызывает `sum`; если она вернула `Some(sum)`, сумма должна быть 6; после `join` `sum()` должна быть `Some(6)`. Добейтесь, чтобы тест проходил, и проверьте, что он падает, если для `ready` использовать `Relaxed`.

<details>
<summary>Подсказка</summary>

Типы берутся из `loom::sync::atomic` и `loom::sync::Arc`, поток — из `loom::thread`. Какая запись и какое чтение образуют пару release/acquire?

</details>

<details>
<summary>Ответ</summary>

```rust
use loom::sync::Arc;
use loom::sync::atomic::{AtomicBool, AtomicU64, Ordering};
use loom::thread;

/// A batch of three numbers that one thread fills and another reads.
struct Batch {
    values: [AtomicU64; 3],
    ready: AtomicBool,
}

impl Batch {
    fn new() -> Self {
        Batch {
            values: [AtomicU64::new(0), AtomicU64::new(0), AtomicU64::new(0)],
            ready: AtomicBool::new(false),
        }
    }

    fn fill(&self) {
        for (i, v) in self.values.iter().enumerate() {
            v.store(i as u64 + 1, Ordering::Relaxed); // 1, 2, 3
        }
        self.ready.store(true, Ordering::Release);
    }

    /// The sum of the batch, or None if it is not ready yet.
    fn sum(&self) -> Option<u64> {
        if !self.ready.load(Ordering::Acquire) {
            return None;
        }
        Some(self.values.iter().map(|v| v.load(Ordering::Relaxed)).sum())
    }
}

#[test]
fn a_ready_batch_is_always_complete() {
    loom::model(|| {
        let batch = Arc::new(Batch::new());
        let writer = {
            let batch = Arc::clone(&batch);
            thread::spawn(move || batch.fill())
        };
        if let Some(sum) = batch.sum() {
            assert_eq!(sum, 6);
        }
        writer.join().unwrap();
        assert_eq!(batch.sum(), Some(6));
    });
}
```

`cargo test --release --test loom_batch`:

```text
test a_ready_batch_is_always_complete ... ok
```

С `Relaxed` на обеих сторонах `ready` тот же тест падает:

```text
test a_ready_batch_is_always_complete ... FAILED
assertion `left == right` failed
  left: 0
 right: 6
```

Почему это работает: запись `Release` в `ready` идёт после трёх записей значений в порядке программы, поэтому все они происходят-до любого чтения `Acquire`, которое увидело `ready == true`. С `Relaxed` loom находит запуск, где читатель видит `true`, но ещё ни одного значения (сумма 0). После `join` главный поток и так видит всё, потому что `join` сам синхронизирует.

</details>

### Упражнение 3 — Двум флагам нужен полный барьер (сложно)

Два потока хотят знать, «внутри» ли другой: каждый поднимает свой флаг, а затем проверяет флаг другого. Хотя бы один из них должен увидеть, что флаг другого поднят (иначе оба могли бы одновременно войти в критическую секцию). Напишите `tests/loom_sb.rs` с функцией `raise_then_check(flag, other, use_fence) -> bool`, которая записывает `true` во `flag` с `Release`, при необходимости вызывает `fence(Ordering::SeqCst)` и возвращает `other.load(Ordering::Acquire)`. Напишите два теста loom: без барьера (должен падать) и с барьером (должен проходить). Объясните разницу через раздел 4.

<details>
<summary>Подсказка</summary>

`fence` тоже есть в `loom::sync::atomic`. Проверка — `assert!(main_saw_a || thread_saw_b, …)`. Вспомните из раздела 6, почему здесь используется барьер, а не чтения и записи `SeqCst`.

</details>

<details>
<summary>Ответ</summary>

```rust
use loom::sync::Arc;
use loom::sync::atomic::{AtomicBool, Ordering, fence};
use loom::thread;

/// Each side raises its own flag, then looks at the other one.
/// At least one side must see the other's flag raised.
fn raise_then_check(flag: &AtomicBool, other: &AtomicBool, use_fence: bool) -> bool {
    flag.store(true, Ordering::Release);
    if use_fence {
        fence(Ordering::SeqCst); // a full barrier between the store and the load
    }
    other.load(Ordering::Acquire)
}

fn model(use_fence: bool) {
    loom::model(move || {
        let a = Arc::new(AtomicBool::new(false));
        let b = Arc::new(AtomicBool::new(false));
        let (a2, b2) = (Arc::clone(&a), Arc::clone(&b));
        let t = thread::spawn(move || raise_then_check(&a2, &b2, use_fence));
        let main_saw_a = raise_then_check(&b, &a, use_fence);
        let thread_saw_b = t.join().unwrap();
        assert!(main_saw_a || thread_saw_b, "both sides saw the other flag down");
    });
}

#[test]
fn without_fence() {
    model(false);
}

#[test]
fn with_seqcst_fence() {
    model(true);
}
```

`cargo test --release --test loom_sb`:

```text
test without_fence ... FAILED
test with_seqcst_fence ... ok
thread 'without_fence' (5777257) panicked at tests/loom_sb.rs:23:9:
both sides saw the other flag down
```

Почему это работает: это приём буферизации записи из раздела 4. Release/acquire упорядочивает только операции *до* release и *после* acquire; ничто не мешает чтению acquire каждого потока выполниться раньше его собственной записи release, поэтому оба могут прочитать `false`. `fence(SeqCst)` помещает все барьеры SeqCst в один общий порядок: какой бы барьер ни шёл первым, чтение другого потока выполняется после него и обязано увидеть запись этого потока. На настоящем железе раздел 4 показал тот же эффект с записями и чтениями `SeqCst`. В loom мы используем барьер, потому что loom обращается с обращениями `SeqCst` как с `AcqRel` и поднял бы ложную тревогу.

</details>

## Вопросы для самопроверки

1. Кто может переупорядочивать операции с памятью и что остаётся гарантированным?

<details>
<summary>Ответ</summary>

И компилятор, и CPU — в рамках правил модели памяти. Каждый поток всегда видит собственные операции в порядке программы. Что и в каком порядке видят другие потоки, гарантируют только выбранные вами порядки.

</details>

2. Что значит, что arm64 «слабо упорядочен», и чем это полезно вам как ученику?

<details>
<summary>Ответ</summary>

CPU может сделать записи видимыми другим ядрам в другом порядке, чем их записала программа, и выполнять чтения раньше. Поэтому ошибки из-за пропущенных порядков проявляются на Mac (как в лакмусовом тесте), хотя на сильно упорядоченной машине x86-64 остались бы скрытыми.

</details>

3. Объясните гарантию release/acquire одним-двумя предложениями.

<details>
<summary>Ответ</summary>

Если чтение `Acquire` прочитало значение, записанное записью `Release`, то всё, что пишущий поток сделал до этой записи, происходит-до всего, что читающий поток делает после этого чтения, и читатель всё это видит.

</details>

4. Поток записывает данные, а затем флаг с `Release`. Читатель читает флаг с `Relaxed`, а затем данные. Это корректно?

<details>
<summary>Ответ</summary>

Нет. Гарантии нужна пара: чтение должно быть `Acquire`. С чтением `Relaxed` читатель может увидеть поднятый флаг и всё равно старые данные.

</details>

5. Какой исход запрещает `SeqCst`, но допускает release/acquire?

<details>
<summary>Ответ</summary>

Исход буферизации записи: два потока записывают каждый в свою переменную, а затем читают переменную другого, и оба читают старое значение. `SeqCst` выстраивает все свои операции в один общий порядок, поэтому хотя бы один поток обязан увидеть запись другого.

</details>

6. Что делает loom, чего не может стресс-тест, и каково одно из его ограничений?

<details>
<summary>Ответ</summary>

Он систематически перебирает каждое чередование небольшой модели и позволяет атомарным чтениям возвращать любое значение, которое допускает модель памяти, поэтому находит ошибки порядка на любой машине и каждый раз. Одно ограничение: обращения `SeqCst` он считает `AcqRel` (возможны ложные тревоги); `fence(SeqCst)` моделируется правильно.

</details>

## Итоги

- Другие потоки могут видеть ваши записи в другом порядке. Переупорядочивают и компилятор, и CPU; arm64 (ваш Mac) переупорядочивает гораздо больше, чем x86-64.
- Нужная вам гарантия — «происходит-до». Её создают запись `Release` и чтение `Acquire`, которое прочитало её значение: всё, что было до release, видно после acquire.
- Используйте `Relaxed` для простых чисел и флагов остановки, release/acquire — для флагов «данные готовы» и блокировок, `AcqRel` — для чтения-изменения-записи, передающего данные дальше, а `SeqCst` — только когда потоки должны согласиться о порядке операций над разными переменными.
- На arm64 порядки — это разные инструкции: `str`/`ldr`, `stlr`, `ldapr`, `ldar`.
- Лакмусовые тесты показывают переупорядочивания на настоящем железе, но им нужно везение; loom систематически находит их в небольших моделях. Помните, что loom моделирует обращения `SeqCst` как `AcqRel`.
- В сомнительных случаях используйте `Mutex`, канал или `OnceLock`, внутри которых уже правильные порядки.

## Источники и что почитать

- [std::sync::atomic::Ordering](https://doc.rust-lang.org/std/sync/atomic/enum.Ordering.html) — официальное описание каждого порядка.
- [The Rustonomicon — Atomics](https://doc.rust-lang.org/nomicon/atomics.html) — модель памяти, «происходит-до» и почему железо переупорядочивает.
- [Rust Atomics and Locks, chapter 3 — Memory Ordering](https://marabos.nl/atomics/memory-ordering.html) — Mara Bos объясняет «происходит-до», release/acquire и `SeqCst` с понятными схемами.
- [Rust Atomics and Locks, chapter 7 — Understanding the Processor](https://marabos.nl/atomics/hardware.html) — во что превращаются порядки на arm64 и x86-64 и почему arm64 слабо упорядочен.
- [loom на docs.rs](https://docs.rs/loom) — инструмент проверки моделей из раздела 6 со списком того, что он не моделирует.
- [cppreference — std::memory_order](https://en.cppreference.com/w/cpp/atomic/memory_order) — справочное описание модели памяти C++, которой следует Rust.

---

[🏠 Начало](../START-HERE.md) · [← 14.5 Атомарные операции](05-atomics.md) · [📋 Этап 14](README.md) · [14.7 Пишем спин-блокировку →](07-building-a-spin-lock.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
