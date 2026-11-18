<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 19.8 Настройки сборки для скорости: профили, LTO и PGO](08-build-settings-for-speed.md) · [📋 Этап 19](README.md) · [19.10 Проект: ускорить медленную программу →](10-project-make-a-slow-program-fast.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/19-performance-and-profiling/09-async-and-io-performance.md) <!-- nav -->

# Урок 19.9 — Производительность асинхронного кода и ввода-вывода

> **Этап 19 — Производительность и профилирование** · Урок 9 из 11 · ⏱ около 135 минут

## Что вы узнаете

- Почему множество мелких чтений и записей медленно и как это исправляет буферизация (`BufWriter`, `BufReader`).
- Как считать системные вызовы с помощью `strace` в Linux-контейнере.
- Почему ожидание операций одна за другой тратит время впустую и как ждать многого сразу — с ограничением.
- Как узнать, блокирует ли задача асинхронный рантайм, и три способа это исправить.

## Перед началом

- Вы прошли [Урок 19.8 — Настройки сборки для скорости](08-build-settings-for-speed.md).
- Вы выполнили нагрузочные тесты из [Урока 19.2 — Нагрузочное тестирование сервисов](02-load-testing.md), включая «ожидание против блокировки» и `spawn_blocking`.
- Вы знаете системные вызовы ([Урок 13.3](../13-operating-systems/03-system-calls.md)), future ([Урок 14.10](../14-concurrency-and-async/10-how-futures-work.md)), задачи Tokio ([Урок 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)) и axum ([Урок 16.3](../16-backend-services/03-web-services-with-axum.md)).
- Создайте пакет и добавьте зависимости:

```bash
cd ~/projects/rust
cargo new code/p19-io
cd code/p19-io
cargo add tokio@1 --features full
cargo add futures@0.3
mkdir src/bin
```

## Новые слова

| Слово | Значение |
|---|---|
| **I/O** (Input/Output, ввод-вывод) | Чтение и запись данных за пределами памяти программы: файлы, терминал, сеть. |
| **буфер** (buffer) | Блок памяти, где данные собираются перед записью или хранятся после чтения, чтобы один большой системный вызов заменил много мелких. |
| **буферизованный ввод-вывод** (buffered I/O) | Ввод-вывод через буфер. `BufWriter` собирает записи; `BufReader` читает большими блоками и выдаёт маленькие куски из памяти. |
| **сброс буфера** (flush) | Немедленная запись всего, что ждёт в буфере. |
| **построчная буферизация** (line buffering) | Буфер, который сбрасывается на каждом переводе строки. Так работает стандартный вывод Rust. |
| **strace** | Инструмент Linux, который запускает программу и показывает (или считает) каждый её системный вызов. |
| **веерный запрос** (fan-out) | Одновременный запуск многих независимых операций (например, запросов к другим сервисам) с последующим сбором их результатов. |
| **ограничение конкурентности** (concurrency limit) | Наибольшее число операций, которым разрешено выполняться одновременно. |
| **простой рантайма** (stall) | Момент, когда асинхронный рантайм не может запустить готовые задачи, потому что рабочий поток занят одной задачей, которая не уступает его. |
| **кооперативное планирование** (cooperative scheduling) | Задача выполняется, пока не дойдёт до `.await`, который ещё не готов. Рантайм не может прервать её посередине; задача должна сама вернуть поток. |
| **уступить** (yield) | Добровольно вернуть поток рантайму, чтобы могли выполниться другие задачи (`tokio::task::yield_now().await`). |

## 1. Каждый системный вызов стоит денег: буферизованная запись

Запись в файл, конвейер или терминал — это **системный вызов** (Урок 13.3): программа просит ядро сделать это за неё. Системный вызов стоит гораздо дороже обычного вызова функции, потому что процессор должен переключиться в ядро и обратно. Записать 10 байт или 8 000 байт стоит почти одинаково, поэтому число вызовов важнее числа байтов.

Вот программа, которая печатает миллион коротких строк тремя способами. Поместите её в `src/bin/write_lines.rs`:

```rust
use std::io::{self, BufWriter, Write};

fn main() -> io::Result<()> {
    let mode = std::env::args().nth(1).unwrap_or_default();
    let n = 1_000_000;
    match mode.as_str() {
        // One `println!` per line.
        "println" => {
            for i in 0..n {
                println!("line {i}");
            }
        }
        // Lock stdout once, but still write each line straight through.
        "locked" => {
            let mut out = io::stdout().lock();
            for i in 0..n {
                writeln!(out, "line {i}")?;
            }
        }
        // Lock once and collect output in a 8 KiB buffer.
        "buffered" => {
            let mut out = BufWriter::new(io::stdout().lock());
            for i in 0..n {
                writeln!(out, "line {i}")?;
            }
            out.flush()?;
        }
        _ => eprintln!("usage: write_lines <println|locked|buffered>"),
    }
    Ok(())
}
```

**Что происходит:**

1. `println!` блокирует стандартный вывод, пишет строку и снова снимает блокировку — для каждой строки. Стандартный вывод **буферизуется построчно**: на каждом `\n` он отдаёт строку ядру одним системным вызовом `write`.
2. `io::stdout().lock()` берёт блокировку один раз на весь цикл. `writeln!` — это `println!` для любого объекта записи. Построчная буферизация остаётся.
3. `BufWriter::new(...)` ставит впереди буфер на 8 КиБ. Теперь `writeln!` только копирует байты в память; когда буфер заполнен, `BufWriter` записывает его целиком одним системным вызовом.
4. `out.flush()?` записывает последний, частично заполненный буфер. `BufWriter` сбрасывается и при уничтожении (drop), но тогда он не может сообщить об ошибке, поэтому сбрасывайте его сами.

`hyperfine` по умолчанию выбрасывает вывод программы (отправляет его в `/dev/null`), поэтому мы измеряем только саму программу:

```bash
cargo build --release
hyperfine -N --warmup 2 --runs 10 -L mode println,locked,buffered './target/release/write_lines {mode}'
```

```text
Benchmark 1: ./target/release/write_lines println (10 runs)
  Wall Time    1.282 s   ±   0.204 s      0.984 s   …   1.597 s
Benchmark 2: ./target/release/write_lines locked (10 runs)
  Wall Time    1.429 s   ±   0.371 s      1.020 s   …   2.421 s            +11.4%
Benchmark 3: ./target/release/write_lines buffered (10 runs)
  Wall Time    0.026 s   ±   0.005 s      0.021 s   …   0.037 s            -98.0%  (49.1x faster)
```

(Строки `hyperfine` о памяти здесь и ниже опущены.) Буферизованная версия **примерно в 50 раз быстрее**. То, что блокировка берётся только один раз, ничего заметно не изменило: блокировка дешёвая, системные вызовы — нет.

### Считаем системные вызовы

Чтобы *увидеть* системные вызовы, используйте `strace`. Это инструмент Linux, поэтому запустите его в контейнере из официального образа Rust (Урок 11.12). `--cap-add SYS_PTRACE` разрешает `strace` наблюдать за другим процессом; `strace -c` считает вызовы вместо того, чтобы печатать каждый, а `-e trace=write` ограничивает подсчёт вызовом `write`:

```bash
docker run --rm --name course-19-09-strace --cap-add SYS_PTRACE \
  -v "$PWD":/work -w /work -v course-cargo-registry:/usr/local/cargo/registry \
  -e CARGO_TARGET_DIR=/tmp/target rust:1.100 bash -c '
    apt-get update -qq >/dev/null && apt-get install -y -qq strace >/dev/null 2>&1
    cargo build --release -q --bin write_lines
    for m in println buffered; do
      echo "== $m"
      strace -c -e trace=write /tmp/target/release/write_lines $m > /dev/null
    done'
```

```text
== println
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    4.655557           4   1000000           write
------ ----------- ----------- --------- --------- ----------------
100.00    4.655557           4   1000000           total
== buffered
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
100.00    0.005009           2      2172           write
------ ----------- ----------- --------- --------- ----------------
100.00    0.005009           2      2172           total
```

**Что происходит:**

1. `CARGO_TARGET_DIR=/tmp/target` оставляет сборку для Linux внутри контейнера, чтобы она не смешивалась с вашей папкой `target/` для macOS.
2. `println`: ровно **1 000 000** вызовов `write`, по одному на строку, хотя вывод идёт в `/dev/null`, а не в терминал.
3. `buffered`: **2 172** вызова на те же 11,9 МБ (мегабайт). (Не ровно 11,9 МБ ÷ 8 КиБ ≈ 1 450, потому что построчный буфер стандартного вывода за `BufWriter` иногда делит блок на два вызова; это не важно.)
4. Сам `strace` сильно замедляет каждый системный вызов (по 4 мкс), поэтому не сравнивайте его время с `hyperfine`. Используйте его для **подсчёта**.

> ⚡ **Производительность:** программа, которая пишет много вывода (анализатор логов, конвертер, отчёт), должна писать через один `BufWriter` вокруг заблокированного `stdout`, созданный один раз в `main` и передаваемый дальше. Часто это самый дешёвый крупный выигрыш в программе командной строки.

## 2. Чтение большими блоками

У чтения та же проблема, только в обратную сторону. `Read::bytes()` у `File` даёт итератор по байтам, и каждый байт — это один системный вызов `read`. Поместите это в `src/bin/count_lines.rs`:

```rust
use std::fs::File;
use std::io::{self, BufReader, Read};

fn main() -> io::Result<()> {
    let args: Vec<String> = std::env::args().collect();
    let (mode, path) = (args[1].as_str(), &args[2]);
    let file = File::open(path)?;
    let lines = match mode {
        // Every byte is one `read` system call.
        "raw" => file.bytes().filter(|b| matches!(b, Ok(b'\n'))).count(),
        // BufReader reads 8 KiB at a time and hands out bytes from memory.
        "buffered" => BufReader::new(file)
            .bytes()
            .filter(|b| matches!(b, Ok(b'\n')))
            .count(),
        _ => panic!("usage: count_lines <raw|buffered> FILE"),
    };
    println!("{lines} lines");
    Ok(())
}
```

Clippy (Урок 05.6) знает эту ловушку:

```bash
cargo clippy --bin count_lines
```

```text
warning: calling .bytes() is very inefficient when data is not in memory
  --> src/bin/count_lines.rs:10:18
   |
10 |         "raw" => file.bytes().filter(|b| matches!(b, Ok(b'\n'))).count(),
   |                  ^^^^^^^^^^^^
   |
   = help: consider using `BufReader`
```

Упражнение 1 измеряет разницу. `BufReader` также даёт `read_line` и `lines()`, которыми вы пользуетесь с Этапа 05. Самый быстрый способ прочитать целый файл, который помещается в памяти, — `std::fs::read` (или `read_to_string`): он читает его несколькими большими вызовами.

## 3. Ждать по очереди или вместе

Первые два раздела были о том, что процессор делает слишком много системных вызовов. У асинхронного кода другая типичная проблема: он **ждёт** неправильно.

Представьте обработчик, которому нужны цены 20 товаров от другого сервиса, и каждый запрос занимает 50 мс. Поместите это в `src/bin/fan_out.rs`:

```rust
use std::time::{Duration, Instant};

use futures::stream::{self, StreamExt};

/// Pretends to ask another service for the price of one item: it just waits 50 ms.
async fn fetch_price(item: u32) -> u32 {
    tokio::time::sleep(Duration::from_millis(50)).await;
    item * 10
}

#[tokio::main]
async fn main() {
    let items: Vec<u32> = (1..=20).collect();

    // 1. One after another.
    let start = Instant::now();
    let mut total = 0;
    for &item in &items {
        total += fetch_price(item).await;
    }
    println!("one by one:     total {total} in {:?}", start.elapsed());

    // 2. All at the same time.
    let start = Instant::now();
    let prices = futures::future::join_all(items.iter().map(|&item| fetch_price(item))).await;
    let total: u32 = prices.iter().sum();
    println!("all at once:    total {total} in {:?}", start.elapsed());

    // 3. At most 5 at the same time.
    let start = Instant::now();
    let total: u32 = stream::iter(items.iter().copied())
        .map(fetch_price)
        .buffer_unordered(5)
        .fold(0, |acc, price| async move { acc + price })
        .await;
    println!("5 at a time:    total {total} in {:?}", start.elapsed());
}
```

`cargo run --release --bin fan_out`:

```text
one by one:     total 2100 in 1.03218175s
all at once:    total 2100 in 51.334334ms
5 at a time:    total 2100 in 206.643667ms
```

**Что происходит:**

1. Цикл `for` ждёт каждый future, прежде чем создать следующий. Ожидания складываются: 20 × 50 мс = 1 секунда. Почти всё это время процессор простаивает.
2. `join_all` берёт все 20 future и опрашивает их вместе (Урок 14.10): все 20 таймеров идут одновременно, поэтому вся пачка занимает 50 мс. Дополнительные потоки не нужны: одна задача ждёт 20 вещей сразу.
3. `stream::iter(...).map(fetch_price)` создаёт асинхронный поток (stream) из future, которые ещё не запущены. `buffer_unordered(5)` выполняет не больше 5 из них одновременно и выдаёт результаты в порядке завершения. 20 товаров группами по 5 занимают 4 × 50 мс ≈ 200 мс.
4. `fold` складывает результаты по мере поступления. Все три версии дают одинаковую сумму.

Разница между 1 с и 50 мс — не микрооптимизация: это задержка, которую чувствуют ваши пользователи. Глядя на медленный обработчик, сначала спросите: «не ждёт ли он независимые вещи одну за другой?» Для небольшого фиксированного числа разных вызовов `tokio::join!(a, b, c)` делает то же, что `join_all` (Упражнение 3).

> 🔒 **Безопасность:** «всё сразу» опасно, когда число элементов приходит от пользователя. Запрос со 100 000 элементов открыл бы 100 000 соединений с другим сервисом в один момент, что может его перегрузить (отказ в обслуживании, вызванный вашим собственным кодом). Используйте **ограничение конкурентности**: `buffer_unordered(n)` или `tokio::sync::Semaphore`, общий для всех запросов.

## 4. Задача, которая не возвращает поток

Урок 19.2 показал, что `std::thread::sleep` внутри обработчика замедляет весь сервис. Тяжёлое *вычисление* внутри задачи действует так же, и его легче пропустить, потому что код выглядит безобидно.

Tokio использует **кооперативное планирование**: задача выполняется на рабочем потоке, пока не дойдёт до `.await`, который ещё не готов. Между двумя такими точками рантайм не может отобрать поток. Цикл, который считает 300 мс без `.await`, держит свой рабочий поток 300 мс, и все остальные задачи этого потока ждут.

Это можно измерить с помощью **тикера**: задачи, которая хочет просыпаться каждые 10 мс и записывает, насколько поздно она проснулась на самом деле. Поместите это в `src/bin/stall.rs`:

```rust
use std::hint::black_box;
use std::time::{Duration, Instant};

/// Heavy computation: pure CPU work, no waiting.
fn crunch(rounds: u64) -> u64 {
    let mut x = 1u64;
    for i in 0..rounds {
        x = black_box(x.wrapping_mul(6364136223846793005).wrapping_add(i));
    }
    x
}

/// Wakes up every 10 ms and returns the worst delay it saw, in milliseconds.
async fn ticker(ticks: u32) -> u128 {
    let mut worst = Duration::ZERO;
    for _ in 0..ticks {
        let start = Instant::now();
        tokio::time::sleep(Duration::from_millis(10)).await;
        worst = worst.max(start.elapsed().saturating_sub(Duration::from_millis(10)));
    }
    worst.as_millis()
}

#[tokio::main(flavor = "current_thread")]
async fn main() {
    let mode = std::env::args().nth(1).unwrap_or_default();
    let rounds = 200_000_000;
    let start = Instant::now();
    let ticks = tokio::spawn(ticker(30));
    // Let the ticker run for a moment before the heavy work starts.
    tokio::time::sleep(Duration::from_millis(50)).await;
    let result: u64 = match mode.as_str() {
        "inline" => crunch(rounds),
        "blocking" => tokio::task::spawn_blocking(move || crunch(rounds))
            .await
            .unwrap(),
        "yield" => {
            let mut x = 0;
            for _ in 0..200 {
                x ^= crunch(rounds / 200);
                tokio::task::yield_now().await;
            }
            x
        }
        _ => panic!("usage: stall <inline|blocking|yield>"),
    };
    let worst = ticks.await.unwrap();
    black_box(result);
    println!(
        "{mode:<8} worst ticker delay {worst:>3} ms, total {:?}",
        start.elapsed()
    );
}
```

**Что происходит:**

1. `flavor = "current_thread"` запускает весь рантайм в одном потоке (Урок 14.13). Так эффект легко увидеть; в многопоточном рантайме то же самое происходит с каждым рабочим потоком.
2. `crunch` — чистая работа процессора; `black_box` не даёт компилятору её удалить (Урок 05.12).
3. `ticker` тридцать раз спит по 10 мс и запоминает наибольшую лишнюю задержку. На свободном рантайме это 1–2 мс.
4. `inline` вызывает `crunch` прямо внутри асинхронного `main`: ни одного `.await` за всё вычисление.
5. `blocking` переносит вычисление в отдельный пул потоков Tokio для блокирующей работы через `spawn_blocking` (Урок 19.2) и ждёт результат.
6. `yield` делит работу на 200 частей и после каждой вызывает `tokio::task::yield_now().await`. Этот `.await` возвращает поток рантайму, который может выполнить тикер, а затем продолжает цикл.

```bash
cargo build --release --bin stall
./target/release/stall inline
./target/release/stall blocking
./target/release/stall yield
```

```text
inline   worst ticker delay 446 ms, total 799.69675ms
blocking worst ticker delay   2 ms, total 501.711208ms
yield    worst ticker delay   4 ms, total 504.812083ms
```

**Что происходит:**

1. `inline`: тикер опоздал на **446 мс**. Почти полсекунды ни одна другая задача этого рантайма не могла выполняться: ни таймер, ни ответ из сети — ничего. В веб-сервисе ждал бы каждый запрос на этом рабочем потоке.
2. `blocking`: 2 мс, как на свободном рантайме. Вычисление шло в другом потоке.
3. `yield`: 4 мс. Работа осталась в потоке рантайма, но мелкими частями; каждая часть занимает около 2 мс, поэтому никто не ждёт дольше.

Какое исправление выбрать:

| Ситуация | Исправление |
|---|---|
| Блокирующий вызов, который нельзя изменить (`std::fs`, синхронный драйвер базы данных, `std::thread::sleep`) | `tokio::task::spawn_blocking` |
| Короткое тяжёлое вычисление на каждый запрос (хеширование пароля, сжатие ответа) | `spawn_blocking` или пул потоков для вычислений, например `rayon` (Урок 14.4), с каналом обратно |
| Длинный цикл, который вы пишете сами | Разделите его и вызывайте `yield_now().await` каждые несколько миллисекунд |
| Ожидание времени или сети | Используйте асинхронную версию (`tokio::time::sleep`, `tokio::net`), никогда не блокирующую |

> 🧪 **Тестирование:** простои не видны в обычных тестах, потому что у каждого теста рантайм свой. Тикер вроде показанного выше — дешёвый детектор простоев: запустите его в тесте рядом с проверяемым кодом и проверьте, что наибольшая задержка остаётся, например, ниже 20 мс. В работающем сервисе инструмент `tokio-console` показывает для каждой задачи, сколько длятся её опросы, поэтому задача, которая долго работает, не уступая поток, сразу заметна.

## 5. Чек-лист для медленного асинхронного кода и ввода-вывода

| Симптом | Типичная причина | Что делать |
|---|---|---|
| Программа командной строки проводит большую часть времени в ядре | тысячи крошечных записей или чтений | `BufWriter` / `BufReader` (разделы 1–2) |
| Обработчик занимает *сумму* времени своих вызовов к другим сервисам | `.await` один за другим | `tokio::join!`, `join_all`, `buffer_unordered(n)` (раздел 3) |
| Под нагрузкой все эндпоинты замедляются вместе | задача блокирует рабочий поток | `spawn_blocking`, `yield_now`, асинхронные API (раздел 4) |
| Задержка растёт с числом элементов в запросе | неограниченный веерный запрос перегружает зависимость | ограничение конкурентности (`buffer_unordered(n)`, `Semaphore`) |
| Каждый запрос платит за новое TCP- и TLS-соединение | новый HTTP-клиент на каждый запрос | создайте один клиент (например, `reqwest::Client`) при запуске и используйте его везде; он держит пул открытых соединений |
| Много крошечных записей в сеть | ответ пишется в `TcpStream` по кусочкам | оберните поток в буфер (`tokio::io::BufWriter`) или сначала соберите сообщение и запишите его одним `write_all` |

HTTP (HyperText Transfer Protocol, протокол передачи гипертекста) — протокол веб-запросов; он работает поверх TCP-соединения (Transmission Control Protocol, протокол управления передачей). TLS (Transport Layer Security, протокол защиты транспортного уровня) — это слой шифрования HTTPS, то есть HTTP поверх TLS (Урок 15.12). Установка нового TCP- и TLS-соединения стоит нескольких сетевых обменов туда и обратно, поэтому общий клиент с пулом соединений так важен.

## Частые ошибки

> ⚠️ **Частая ошибка:** завершать программу через `std::process::exit`, пока данные ещё лежат в `BufWriter`. `exit` сразу останавливает процесс, `Drop` не выполняется, и вывод теряется:
>
> ```rust
> use std::io::{self, BufWriter, Write};
>
> fn main() -> io::Result<()> {
>     let mut out = BufWriter::new(io::stdout().lock());
>     writeln!(out, "important result: 42")?;
>     std::process::exit(0); // ends the process at once: `out` is never dropped
> }
> ```
>
> Это **ничего** не печатает. Вызовите `out.flush()?` перед `exit` или просто вернитесь из `main`.

> ⚠️ **Частая ошибка:** `eprintln!` для сообщений о прогрессе в горячем цикле. У стандартного вывода ошибок вообще нет буфера: каждый вызов — это системный вызов. Печатайте прогресс раз в несколько тысяч элементов, а не на каждом.

> ⚠️ **Частая ошибка:** считать, что `buffer_unordered` сохраняет порядок. Он выдаёт результаты в порядке **завершения**. Если порядок важен, используйте `buffered(n)` (то же ограничение, исходный порядок) или храните индекс вместе с каждым результатом.

> ⚠️ **Частая ошибка:** «чинить» медленный обработчик добавлением рабочих потоков. Если задача блокирует, больше потоков только переносит проблему: при достаточной нагрузке все они снова заблокируются. Уберите саму блокировку.

## Попробуйте сами

1. Соберите `write_lines` и повторите сравнение с `hyperfine`. Затем выполните `./target/release/write_lines buffered > lines.txt` и проверьте файл командой `wc -l lines.txt`.
2. Измените размер буфера: `BufWriter::with_capacity(64 * 1024, ...)`. Быстрее ли это, чем 8 КиБ?
3. В `fan_out.rs` замените `buffer_unordered(5)` на `buffer_unordered(10)` и на `buffered(5)`. Предскажите время до запуска.
4. В `stall.rs` замените `#[tokio::main(flavor = "current_thread")]` на `#[tokio::main(flavor = "multi_thread", worker_threads = 2)]`. Что происходит с задержкой в режиме `inline` и почему?

## Упражнения

### Упражнение 1 — Измерьте небуферизованное чтение (легко)

Создайте `lines.txt` командой `./target/release/write_lines buffered > lines.txt` (миллион строк, 11,9 МБ). Соберите `count_lines` и сравните `raw` и `buffered` через `hyperfine` (используйте `--runs 3`: медленный вариант занимает секунды). Во сколько раз быстрее `buffered` и сколько вызовов `read` делает `raw`?

<details>
<summary>Подсказка</summary>

`hyperfine -N --warmup 1 --runs 3 -L mode raw,buffered './target/release/count_lines {mode} lines.txt'`. О числе вызовов подумайте так: один вызов на байт.

</details>

<details>
<summary>Ответ</summary>

```text
Benchmark 1: ./target/release/count_lines raw lines.txt (3 runs)
  Wall Time    3.668 s   ±   0.095 s      3.597 s   …   3.775 s
Benchmark 2: ./target/release/count_lines buffered lines.txt (3 runs)
  Wall Time    0.029 s   ±   0.000 s      0.028 s   …   0.029 s            -99.2%  (127.0x faster)
```

Обе печатают `1000000 lines`. Почему это работает: `raw` делает один системный вызов `read` на каждый байт, то есть около 11,9 миллиона вызовов (плюс один в конце, который возвращает 0, что означает «конец файла»). `BufReader` просит 8 КиБ за вызов — около 1 450 вызовов — и выдаёт байты из памяти. Замена одного слова ускорила программу в 127 раз.

</details>

### Упражнение 2 — Пишите в любой объект записи и тестируйте (средне)

Напишите библиотечную функцию `pub fn write_squares<W: Write>(out: &mut W, n: u64) -> io::Result<()>` в `src/lib.rs`, которая пишет квадраты чисел от 1 до `n`, по одному на строку. Напишите модульный тест, который пишет в `Vec<u8>` и проверяет текст для `n = 4`. Затем вызовите её из `src/main.rs` с `BufWriter` вокруг заблокированного стандартного вывода.

<details>
<summary>Подсказка</summary>

`Vec<u8>` реализует `Write`. `String::from_utf8(out).unwrap()` превращает байты в текст для `assert_eq!`. Имя пакета `p19-io` превращается в имя крейта `p19_io`.

</details>

<details>
<summary>Ответ</summary>

`src/lib.rs`:

```rust
use std::io::{self, Write};

/// Writes the squares of 1..=n, one per line, to any writer.
pub fn write_squares<W: Write>(out: &mut W, n: u64) -> io::Result<()> {
    for i in 1..=n {
        writeln!(out, "{}", i * i)?;
    }
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn writes_one_square_per_line() {
        let mut out = Vec::new();
        write_squares(&mut out, 4).unwrap();
        assert_eq!(String::from_utf8(out).unwrap(), "1\n4\n9\n16\n");
    }
}
```

`src/main.rs`:

```rust
use std::io::{self, BufWriter, Write};

use p19_io::write_squares;

fn main() -> io::Result<()> {
    let mut out = BufWriter::new(io::stdout().lock());
    write_squares(&mut out, 5)?;
    out.flush()
}
```

`cargo test --lib` и `cargo run --release --bin p19-io`:

```text
test tests::writes_one_square_per_line ... ok
```

```text
1
4
9
16
25
```

Почему это работает: функция не решает, *куда* идёт вывод и *как* он буферизуется; это решает вызывающий код. Тест использует `Vec<u8>` (быстро, без файлов), а `main` — буферизованный `stdout`. Такой дизайн делает код вывода и тестируемым, и быстрым.

</details>

### Упражнение 3 — Нагрузочный тест обработчика, который ждёт по очереди (сложно)

Добавьте axum (`cargo add axum@0.8`) и напишите `src/bin/report.rs`: сервис на `127.0.0.1:3021` с маршрутом `/report`, обработчик которого вызывает три фейковых сервиса один за другим (20 мс, 30 мс и 25 мс, через `tokio::time::sleep`) и возвращает `user=ok orders=ok stock=ok`. Измерьте его через `oha -z 5s -c 20 --no-tui` (Урок 19.2). Затем добавьте `/report2`, который делает те же три вызова через `tokio::join!`, и измерьте снова.

<details>
<summary>Подсказка</summary>

Напишите один вспомогательный `async fn lookup(name: &'static str, ms: u64) -> String`. `tokio::join!(a, b, c)` возвращает кортеж `(a_result, b_result, c_result)`. Запустите сервис в одном терминале, а `oha` — в другом.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::time::Duration;

use axum::{Router, routing::get};

/// Pretends to call another service: waits `ms` milliseconds, then answers.
async fn lookup(name: &'static str, ms: u64) -> String {
    tokio::time::sleep(Duration::from_millis(ms)).await;
    format!("{name}=ok")
}

/// Calls the three services one after another.
async fn report_sequential() -> String {
    let user = lookup("user", 20).await;
    let orders = lookup("orders", 30).await;
    let stock = lookup("stock", 25).await;
    format!("{user} {orders} {stock}\n")
}

/// Calls the three services at the same time.
async fn report_concurrent() -> String {
    let (user, orders, stock) = tokio::join!(
        lookup("user", 20),
        lookup("orders", 30),
        lookup("stock", 25)
    );
    format!("{user} {orders} {stock}\n")
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/report", get(report_sequential))
        .route("/report2", get(report_concurrent));
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3021").await.unwrap();
    println!("listening on http://127.0.0.1:3021");
    axum::serve(listener, app).await.unwrap();
}
```

```bash
cargo run --release --bin report
# in a second terminal:
oha -z 5s -c 20 --no-tui http://127.0.0.1:3021/report | grep -E 'Requests/sec|50.00%|99.00%'
oha -z 5s -c 20 --no-tui http://127.0.0.1:3021/report2 | grep -E 'Requests/sec|50.00%|99.00%'
```

```text
  Requests/sec:	251.7851
  50.00% in 80.4035 ms
  99.00% in 87.2549 ms
  Requests/sec:	631.1875
  50.00% in 31.5434 ms
  99.00% in 42.8443 ms
```

Почему это работает: при последовательных вызовах каждый запрос ждёт 20 + 30 + 25 = 75 мс (плюс накладные расходы таймера: p50 80 мс). С `join!` он ждёт только самый медленный вызов, 30 мс (p50 32 мс). При 20 соединениях пропускная способность следует закону Литтла (Урок 19.1): 20 ÷ 0,080 с ≈ 250 запросов/с и 20 ÷ 0,032 с ≈ 630 запросов/с. Процессор в обоих случаях почти ничего не делал; изменилось только ожидание. Остановите сервис через Ctrl+C.

</details>

## Вопросы для самопроверки

1. Почему печать миллиона строк через `println!` медленная, даже когда вывод идёт в `/dev/null`?

<details>
<summary>Ответ</summary>

Стандартный вывод буферизуется построчно: каждая строка — один системный вызов `write`, значит миллион строк — миллион системных вызовов. `strace -c` показал ровно 1 000 000.

</details>

2. Почему нужно вызывать `flush()` у `BufWriter` самому?

<details>
<summary>Ответ</summary>

Чтобы записать последний, частично заполненный буфер и получить ошибку, если запись не удалась. Сброс при уничтожении не может сообщить об ошибке, а при `std::process::exit` уничтожения вообще не происходит.

</details>

3. Что показывает `strace -c` и почему не стоит доверять его времени?

<details>
<summary>Ответ</summary>

Таблицу: сколько раз был сделан каждый системный вызов, с ошибками и временем. `strace` сильно замедляет каждый системный вызов, поэтому его время нереалистично; использовать нужно подсчёты.

</details>

4. Двадцать независимых запросов по 50 мс: сколько они займут при ожидании одного за другим, с `join_all` и с `buffer_unordered(5)`?

<details>
<summary>Ответ</summary>

Около 1 с (20 × 50 мс), около 50 мс (все сразу) и около 200 мс (4 группы по 5).

</details>

5. Что такое кооперативное планирование и что оно означает для долгого вычисления внутри задачи?

<details>
<summary>Ответ</summary>

Задача держит свой рабочий поток, пока не дойдёт до `.await`, который ещё не готов; рантайм не может её прервать. Долгое вычисление без `.await` держит поток всё это время, и все остальные задачи этого потока ждут (простой).

</details>

6. Назовите три способа не дать тяжёлой работе остановить рантайм.

<details>
<summary>Ответ</summary>

`tokio::task::spawn_blocking`; отдельный пул потоков для вычислений, например `rayon`, с каналом обратно; разбиение цикла с регулярным вызовом `tokio::task::yield_now().await`. (А для ожидания — асинхронный API (Application Programming Interface, программный интерфейс: функции, которые даёт библиотека) вместо блокирующего.)

</details>

## Итоги

- Системные вызовы дороги; число вызовов важнее числа байтов. Стандартный вывод делает один `write` на строку.
- Оборачивайте вывод в `BufWriter` (вокруг `stdout().lock()`), а ввод — в `BufReader`; сбрасывайте буфер сами. Миллион строк: 1 000 000 → 2 172 системных вызова, в 50 раз быстрее.
- `strace -c` в Linux-контейнере считает системные вызовы; используйте его подсчёты, а не время.
- Ожидание независимых операций одна за другой складывает их задержки. Ждите их вместе (`join!`, `join_all`) и ограничивайте их число (`buffer_unordered(n)`, `Semaphore`).
- Tokio планирует кооперативно: задача без `.await` держит свой поток. Тяжёлую работу выносите в `spawn_blocking` или пул для вычислений либо регулярно уступайте поток.
- Задача-тикер — простой детектор простоев; `oha` показывает их влияние на задержку под нагрузкой.

## Источники и что почитать

- [`std::io::BufWriter`](https://doc.rust-lang.org/std/io/struct.BufWriter.html) — буферизованный объект записи, его ёмкость и правила сброса.
- [`std::io::Stdout`](https://doc.rust-lang.org/std/io/struct.Stdout.html) — стандартный вывод, его блокировка и построчная буферизация.
- [Tokio — CPU-bound tasks and blocking code](https://docs.rs/tokio/latest/tokio/index.html) — раздел документации Tokio о блокирующей работе (найдите на странице «CPU-bound»).
- [`tokio::task::spawn_blocking`](https://docs.rs/tokio/latest/tokio/task/fn.spawn_blocking.html) и [`tokio::task::yield_now`](https://docs.rs/tokio/latest/tokio/task/fn.yield_now.html) — два исправления из раздела 4.
- [`futures::stream::StreamExt`](https://docs.rs/futures/latest/futures/stream/trait.StreamExt.html) — `buffer_unordered`, `buffered` и другие адаптеры потоков.
- [Страница руководства strace(1)](https://man7.org/linux/man-pages/man1/strace.1.html) — все параметры `strace`, включая `-c` и `-e trace=`.
- [The Rust Performance Book — I/O](https://nnethercote.github.io/perf-book/io.html) — короткие советы о блокировке и буферизации.

---

[🏠 Начало](../START-HERE.md) · [← 19.8 Настройки сборки для скорости: профили, LTO и PGO](08-build-settings-for-speed.md) · [📋 Этап 19](README.md) · [19.10 Проект: ускорить медленную программу →](10-project-make-a-slow-program-fast.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
