<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 19.9 Производительность асинхронного кода и ввода-вывода](09-async-and-io-performance.md) · [📋 Этап 19](README.md) · [19.11 Контрольная точка →](11-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/19-performance-and-profiling/10-project-make-a-slow-program-fast.md) <!-- nav -->

# Урок 19.10 — Проект: ускорить медленную программу

> **Этап 19 — Производительность и профилирование** · Урок 10 из 11 · ⏱ около 8 часов (4 занятия)

## Что вы узнаете

- Как работать как инженер по производительности: заморозить вывод, измерить, профилировать, изменить одну вещь, проверить тестами, измерить снова.
- Как защитить поведение программы «золотыми» тестами, прежде чем трогать её код.
- Как находить настоящие горячие точки программы профилировщиком, а не догадками.
- Как ускорить правильный, но медленный анализатор логов как минимум **в 10 раз**, не изменив ни одного байта его вывода.

## Перед началом

- Вы прошли [Урок 19.9 — Производительность асинхронного кода и ввода-вывода](09-async-and-io-performance.md) и предыдущие уроки этого этапа, особенно [Урок 19.3 — Профилировщики и флеймграфы](03-profilers-and-flame-graphs.md) (`samply`, `sample`, `inferno`) и [Урок 19.5 — Профилирование памяти и меньше выделений](05-memory-profiling-and-allocations.md).
- Установлены `hyperfine`, `samply` и `rustfilt` (Уроки 09.11 и 19.3).
- Вы знаете золотые, или снапшот-тесты ([Урок 09.5 — Снапшот-тестирование](../09-testing-debugging-benchmarking/05-snapshot-testing.md)) и хеш-таблицы ([Урок 04.9](../04-structs-enums-collections-errors/09-hash-maps.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **золотой тест** (golden test) | Тест, который сравнивает весь вывод программы с сохранённым файлом («золотым»), про который известно, что он правильный. |
| **золотой эталон** (golden master) | Сохранённый вывод *старой* программы, который служит золотым файлом, пока вы её переписываете. Он замораживает текущее поведение, каким бы оно ни было. |
| **базовая линия** (baseline) | Измеренное время программы до каких-либо изменений. С ним сравнивается каждое следующее число. |
| **ускорение** (speed-up) | Старое время, делённое на новое. 6,8 с → 0,68 с — это ускорение в 10 раз (10×). |
| **горячая точка** (hot spot) | Небольшая часть кода, на которую уходит большая часть времени. |
| **линейный поиск** (linear search) | Поиск элемента перебором элементов списка один за другим. При n элементах это в среднем около n/2 сравнений. |
| **закон Амдала** (Amdahl's law) | Если часть программы занимает долю *p* времени, то, сделав бесконечно быстрой только её, можно получить ускорение не больше 1 / (1 − *p*). Вы уже встречали его в Уроке 09.11. |

## 1. Метод

Ускорение программы — это цикл, а не одна хитрая идея:

```text
            ┌──────────────────────────────────────────────┐
            ▼                                              │
  freeze the output ──▶ measure ──▶ profile ──▶ change ONE thing
   (golden tests)       (baseline)  (hot spot)       │
                                                     ▼
                                     tests still green? ──no──▶ undo
                                                     │yes
                                                     ▼
                                     measure: faster? ──no──▶ undo
                                                     │yes
                                                     └──▶ commit, and loop again
```

Цикл работает благодаря трём правилам:

1. **Сначала тесты.** Более быстрая программа, которая печатает другой ответ, — сломанная программа. Золотые тесты ловят каждый изменённый байт.
2. **Профилируйте, не гадайте.** В Уроке 19.3 догадка «это `HashMap`» оказалась неверной. Смотрите, куда на самом деле уходит время.
3. **Одно изменение за раз.** Если вы измените пять вещей и программа станет медленнее, вы не узнаете, какое изменение было плохим.

## 2. Программа: `p19-logstat`

Программа читает **журнал доступа** веб-сервера (access log, одна строка на HTTP-запрос; HTTP — HyperText Transfer Protocol, протокол передачи гипертекста) и печатает сводку: сколько запросов, какие коды статуса, какие эндпоинты используются чаще всего и насколько они быстрые, и какие клиенты присылают больше всего запросов. Она правильная. И медленная.

Создайте пакет и его папки:

```bash
cd ~/projects/rust
cargo new code/p19-logstat
cd code/p19-logstat
mkdir -p src/bin tests/data
```

### Генератор логов

`src/bin/gen.rs` пишет фейковый журнал с одними и теми же псевдослучайными числами при каждом запуске, поэтому ваш журнал совпадает с журналом из этого урока. **Не меняйте этот файл.**

```rust
//! Writes a fake web-server access log to standard output.
//! Usage: gen <number of lines>
use std::io::{self, BufWriter, Write};

/// xorshift: a tiny pseudo-random generator, so every run writes the same log.
struct Rng(u64);

impl Rng {
    fn next(&mut self) -> u64 {
        self.0 ^= self.0 << 13;
        self.0 ^= self.0 >> 7;
        self.0 ^= self.0 << 17;
        self.0
    }
    fn below(&mut self, n: u64) -> u64 {
        self.next() % n
    }
}

const PATHS: [&str; 12] = [
    "/",
    "/login",
    "/api/users",
    "/api/users/{}",
    "/api/users/{}/orders",
    "/api/orders/{}",
    "/api/products",
    "/api/products/{}",
    "/api/search",
    "/static/app.js",
    "/static/style.css",
    "/health",
];
const METHODS: [&str; 4] = ["GET", "GET", "POST", "DELETE"];
const STATUSES: [u16; 8] = [200, 200, 200, 201, 304, 404, 429, 500];

fn main() -> io::Result<()> {
    let lines: u64 = std::env::args()
        .nth(1)
        .and_then(|s| s.parse().ok())
        .unwrap_or(1000);
    let mut rng = Rng(2026);
    let mut out = BufWriter::new(io::stdout().lock());
    for i in 0..lines {
        if rng.below(1000) == 0 {
            writeln!(out, "garbage line {i}")?; // about 1 in 1000 lines is broken
            continue;
        }
        let secs = i / 50;
        let (h, m, s) = (10 + secs / 3600 % 14, secs / 60 % 60, secs % 60);
        // A few clients are very busy, most are not.
        let client = if rng.below(4) == 0 {
            rng.below(8)
        } else {
            rng.below(5_000)
        };
        let ip = format!(
            "10.{}.{}.{}",
            client / 65536,
            client / 256 % 256,
            client % 256
        );
        let method = METHODS[rng.below(4) as usize];
        let template = PATHS[rng.below(12) as usize];
        let mut path = template.replace("{}", &rng.below(100_000).to_string());
        if rng.below(10) == 0 {
            path.push_str(&format!("?page={}", rng.below(20)));
        }
        let status = STATUSES[rng.below(8) as usize];
        let mut micros = 200 + rng.below(20_000);
        if rng.below(100) == 0 {
            micros += rng.below(900_000); // 1 in 100 requests is very slow
        }
        let bytes = rng.below(50_000);
        writeln!(
            out,
            "2026-11-18T{h:02}:{m:02}:{s:02}Z {ip} {method} {path} {status} {}.{:03} {bytes}",
            micros / 1000,
            micros % 1000
        )?;
    }
    out.flush()
}
```

В каждой строке семь полей, разделённых одиночными пробелами: время, IP-адрес клиента (Internet Protocol, межсетевой протокол), метод, путь, код статуса, задержка в миллисекундах с тремя знаками после точки и размер ответа в байтах:

```text
2026-11-18T10:00:00Z 10.0.12.189 GET /static/style.css?page=19 500 0.992 8989
2026-11-18T10:00:00Z 10.0.6.127 DELETE /api/orders/54708 201 7.680 27991
2026-11-18T10:00:00Z 10.0.10.241 POST /static/style.css 200 7.937 29033
```

### Медленный анализатор

`src/lib.rs` — программа, которую вы будете ускорять. Прочитайте её внимательно: каждая строка её вывода теперь и есть спецификация.

```rust
//! logstat: reads a web-server access log and prints a summary.
//!
//! This version is correct but slow. Do not change what it prints.

use std::collections::HashMap;
use std::io::{self, BufRead, Write};

/// All numbers collected for one endpoint, such as `GET /api/users/{id}`.
struct Endpoint {
    key: String,
    latencies_us: Vec<u64>,
    total_bytes: u64,
}

/// Reads log lines from `input` and writes the report to `out`.
pub fn run(input: impl BufRead, out: &mut impl Write) -> io::Result<()> {
    let mut lines = 0u64;
    let mut bad = 0u64;
    let mut status_classes: HashMap<String, u64> = HashMap::new();
    let mut endpoints: Vec<Endpoint> = Vec::new();
    let mut clients: Vec<(String, u64)> = Vec::new();

    for line in input.lines() {
        let line = line?;
        lines += 1;
        let fields: Vec<String> = line.split(' ').map(|s| s.to_string()).collect();
        if fields.len() != 7 {
            bad += 1;
            continue;
        }
        let ip = fields[1].clone();
        let method = fields[2].clone();
        let path = fields[3].clone();
        let (Ok(status), Ok(latency_ms), Ok(bytes)) = (
            fields[4].parse::<u16>(),
            fields[5].parse::<f64>(),
            fields[6].parse::<u64>(),
        ) else {
            bad += 1;
            continue;
        };
        let latency_us = (latency_ms * 1000.0).round() as u64;

        *status_classes
            .entry(format!("{}xx", status / 100))
            .or_insert(0) += 1;

        let key = format!("{} {}", method, normalize(&path));
        match endpoints.iter_mut().find(|e| e.key == key) {
            Some(e) => {
                e.latencies_us.push(latency_us);
                e.total_bytes += bytes;
            }
            None => endpoints.push(Endpoint {
                key,
                latencies_us: vec![latency_us],
                total_bytes: bytes,
            }),
        }

        match clients.iter_mut().find(|(c, _)| *c == ip) {
            Some((_, count)) => *count += 1,
            None => clients.push((ip, 1)),
        }
    }

    writeln!(out, "lines: {lines}")?;
    writeln!(out, "bad lines: {bad}")?;
    for class in ["2xx", "3xx", "4xx", "5xx"] {
        let count = status_classes.get(class).copied().unwrap_or(0);
        writeln!(out, "status {class}: {count}")?;
    }
    writeln!(out, "unique clients: {}", clients.len())?;

    endpoints.sort_by(|a, b| {
        b.latencies_us
            .len()
            .cmp(&a.latencies_us.len())
            .then(a.key.cmp(&b.key))
    });
    writeln!(
        out,
        "top endpoints (requests, p50, p95, p99 in ms, average bytes):"
    )?;
    for e in endpoints.iter().take(10) {
        let n = e.latencies_us.len() as u64;
        writeln!(
            out,
            "  {:<28} {:>8} {:>9} {:>9} {:>9} {:>7}",
            e.key,
            n,
            ms(percentile(&e.latencies_us, 50.0)),
            ms(percentile(&e.latencies_us, 95.0)),
            ms(percentile(&e.latencies_us, 99.0)),
            e.total_bytes / n
        )?;
    }

    clients.sort_by(|a, b| b.1.cmp(&a.1).then(a.0.cmp(&b.0)));
    writeln!(out, "top clients:")?;
    for (ip, count) in clients.iter().take(5) {
        writeln!(out, "  {ip:<15} {count:>8}")?;
    }
    Ok(())
}

/// Replaces every path segment made only of digits with `{id}` and drops the query string:
/// `/api/users/42/orders?page=2` becomes `/api/users/{id}/orders`.
fn normalize(path: &str) -> String {
    let without_query = path.split('?').next().unwrap_or("").to_string();
    let parts: Vec<String> = without_query
        .split('/')
        .map(|part| {
            if !part.is_empty() && part.chars().all(|c| c.is_ascii_digit()) {
                "{id}".to_string()
            } else {
                part.to_string()
            }
        })
        .collect();
    parts.join("/")
}

/// The `p`-th percentile with the nearest-rank method (Lesson 19.1).
fn percentile(values: &[u64], p: f64) -> u64 {
    let mut sorted = values.to_vec();
    sorted.sort();
    let rank = (p / 100.0 * sorted.len() as f64).ceil() as usize;
    sorted[rank.saturating_sub(1)]
}

/// Formats microseconds as milliseconds with three decimals: 12345 -> "12.345".
fn ms(us: u64) -> String {
    format!("{}.{:03}", us / 1000, us % 1000)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn normalize_replaces_ids_and_drops_the_query() {
        assert_eq!(
            normalize("/api/users/42/orders?page=2"),
            "/api/users/{id}/orders"
        );
        assert_eq!(normalize("/"), "/");
    }

    #[test]
    fn percentile_uses_nearest_rank() {
        let values: Vec<u64> = (1..=100).collect();
        assert_eq!(percentile(&values, 50.0), 50);
        assert_eq!(percentile(&values, 99.0), 99);
    }
}
```

**Что происходит:**

1. `run` читает журнал строка за строкой и разбивает каждую строку по одиночным пробелам. Строка, в которой не ровно семь полей или у которой статус, задержка или размер не разбираются, считается **плохой строкой**.
2. Задержка переводится в целые микросекунды. Целые числа (а не числа с плавающей точкой) делают перцентили и вывод точными, в каком бы порядке ни собирались значения (вспомните Урок 19.7).
3. Каждый запрос относится к **эндпоинту**: метод плюс путь, в котором числовые сегменты заменены на `{id}`, а строка запроса убрана.
4. Эндпоинты и клиенты хранятся в обычных `Vec` и ищутся перебором.
5. В конце эндпоинты сортируются по числу запросов (при равенстве — по имени), и печатаются первые 10 с задержками p50, p95 и p99 и средним размером; затем первые 5 клиентов (при равенстве — по IP).

`src/main.rs` открывает файл и вызывает `run`:

```rust
use std::fs::File;
use std::io::{self, BufReader};
use std::process;

fn main() -> io::Result<()> {
    let Some(path) = std::env::args().nth(1) else {
        eprintln!("usage: p19-logstat <access.log>");
        process::exit(2);
    };
    let input = BufReader::new(File::open(path)?);
    p19_logstat::run(input, &mut io::stdout().lock())
}
```

### Золотые тесты

`tests/data/small.log` содержит вручную подобранные граничные случаи. Создайте его ровно из этих десяти строк; строка 4 пустая, а в строке 9 после IP-адреса стоят **два** пробела:

```text
2026-11-18T10:00:00Z 10.0.0.1 GET /api/users/42 200 12.345 1000
2026-11-18T10:00:01Z 10.0.0.2 GET /api/users/7?tab=orders 404 0.500 20
2026-11-18T10:00:02Z 10.0.0.1 POST /api/users 201 100.000 300

garbage line
2026-11-18T10:00:03Z 10.0.0.3 GET /api/users/42 abc 1.000 10
2026-11-18T10:00:04Z 10.0.0.1 GET /api/v2/users/42/ 500 3.001 7
2026-11-18T10:00:05Z 10.0.0.2 GET /api/users/12a 304 7.250 0
2026-11-18T10:00:06Z 10.0.0.3  GET /health 200 1.000 2
2026-11-18T10:00:07Z 10.0.0.2 GET /api/users/42 200 2.000 3000
```

`tests/data/small.expected` — то, что печатает для него исходная программа:

```text
lines: 10
bad lines: 4
status 2xx: 3
status 3xx: 1
status 4xx: 1
status 5xx: 1
unique clients: 2
top endpoints (requests, p50, p95, p99 in ms, average bytes):
  GET /api/users/{id}                 3     2.000    12.345    12.345    1340
  GET /api/users/12a                  1     7.250     7.250     7.250       0
  GET /api/v2/users/{id}/             1     3.001     3.001     3.001       7
  POST /api/users                     1   100.000   100.000   100.000     300
top clients:
  10.0.0.1               3
  10.0.0.2               3
```

`tests/golden.rs` запускает настоящий бинарник, как это сделал бы пользователь (Урок 09.2):

```rust
//! Golden tests: the program must print exactly what the original slow version printed.
use std::fs;
use std::path::Path;
use std::process::Command;

/// Runs the logstat binary on `log` and returns what it printed.
fn logstat(log: &Path) -> String {
    let output = Command::new(env!("CARGO_BIN_EXE_p19-logstat"))
        .arg(log)
        .output()
        .expect("failed to start p19-logstat");
    assert!(output.status.success(), "p19-logstat failed: {output:?}");
    String::from_utf8(output.stdout).expect("output is not UTF-8")
}

#[test]
fn small_log_with_edge_cases() {
    let expected = fs::read_to_string("tests/data/small.expected").unwrap();
    assert_eq!(logstat(Path::new("tests/data/small.log")), expected);
}

#[test]
fn generated_log_matches_the_original_output() {
    // Make the same 20,000-line log as in milestone M1.
    let generated = Command::new(env!("CARGO_BIN_EXE_gen"))
        .arg("20000")
        .output()
        .expect("failed to start gen");
    let log = std::env::temp_dir().join(format!("p19-logstat-{}.log", std::process::id()));
    fs::write(&log, generated.stdout).unwrap();
    let actual = logstat(&log);
    fs::remove_file(&log).unwrap();
    let expected = fs::read_to_string("tests/data/gen20000.expected").unwrap();
    assert_eq!(actual, expected);
}
```

**Что происходит:**

1. `env!("CARGO_BIN_EXE_p19-logstat")` Cargo задаёт для интеграционных тестов: это путь к только что собранному бинарнику. То же работает для `gen`.
2. Первый тест проверяет граничные случаи: пустую строку, мусор, плохой статус, строку запроса, завершающий `/`, сегмент не только из цифр (`12a`), двойной пробел.
3. Второй тест генерирует 20 000 строк, запускает анализатор и сравнивает результат с `tests/data/gen20000.expected` — **золотым эталоном**, который вы создадите на вехе M1 из исходной программы.

## Частые ошибки

> ⚠️ **Частая ошибка:** «улучшить» разбор строк. `split_whitespace()` вместо `split(' ')` примет строку с двойным пробелом, и `bad lines` изменится. Обрезка строк, приём других форматов чисел или другое округление задержки — всё это меняет вывод. Золотые тесты вам об этом скажут; слушайте их.

> ⚠️ **Частая ошибка:** сортировать с неполным порядком. Исходная программа сортирует по количеству, а затем по имени. Если убрать второе правило, два эндпоинта с одинаковым количеством могут поменяться местами при другом запуске.

> ⚠️ **Частая ошибка:** измерять отладочную сборку или сравнивать два запуска, сделанных, когда компьютер был занят. Используйте `--release` (Урок 19.8) и `hyperfine` с несколькими запусками (Урок 09.11).

> ⚠️ **Частая ошибка:** менять три вещи сразу. Когда результат медленнее или тест падает, вы не знаете почему. Делайте коммит после каждого успешного шага.

## Проект

### Что вы создадите

Версию `p19-logstat`, которая печатает **в точности** те же байты, что и исходная, для любого входа, и работает **как минимум в 10 раз быстрее** на журнале из миллиона строк. Вы ведёте короткий `README.md`, где каждый шаг записан с измеренным временем, как лабораторный журнал.

### Контракт

- **Использование:** `p19-logstat <access.log>`. Без аргумента программа печатает `usage: p19-logstat <access.log>` в стандартный вывод ошибок и завершается с кодом 2. Файл, который не удаётся открыть, — ошибка (код возврата не 0).
- **Вывод:** байт в байт тот же, что у исходной программы, для любого входа. В частности:
  - Поля разделены **одиночными** пробелами; строка, в которой не ровно 7 полей, плохая. Статус должен разбираться как `u16`, задержка — как `f64`, размер — как `u64`; иначе строка плохая. Пустые и плохие строки всё равно учитываются в `lines`.
  - Задержка в микросекундах — это `(latency_ms * 1000.0).round()`.
  - Ключ эндпоинта: метод, один пробел, путь без части начиная с первого `?`, в котором каждый непустой сегмент только из цифр заменён на `{id}`.
  - Перцентили: метод ближайшего ранга по отсортированным задержкам этого эндпоинта. Средний размер: целочисленное деление.
  - Порядок: эндпоинты по убыванию числа запросов, затем по возрастанию ключа (первые 10); клиенты по убыванию количества, затем по возрастанию IP (первые 5).
- **Скорость:** на журнале, созданном `gen 1000000`, при сборке с `--release` `hyperfine` показывает, что ваша версия как минимум в 10 раз быстрее исходной.
- **Разрешено:** любые изменения в `src/lib.rs` и `src/main.rs`, включая сигнатуру `run`, только стандартная библиотека, без `unsafe`. **Запрещено:** менять `src/bin/gen.rs`, золотые файлы или тесты (тесты можно *добавлять*).
- **Качество:** `cargo test`, `cargo clippy --all-targets -- -D warnings` и `cargo fmt --check` проходят; один коммит Git на каждую веху.

### Пошаговый план

#### M1 — Подготовка и заморозка вывода

**Цель:** пакет из раздела 2 работает, а текущий вывод сохранён как золотой эталон.

Создайте файлы из раздела 2 и `.gitignore` с такими строками (большие файлы и бинарники не должны попасть в Git):

```text
/target
big.log
logstat-orig
*.json.gz
*.sample.txt
```

Соберите, попробуйте оба бинарника и создайте золотой эталон из **исходной** программы:

```bash
cargo build --release
./target/release/gen 3
./target/release/gen 20000 > gen20000.log
./target/release/p19-logstat gen20000.log > tests/data/gen20000.expected
rm gen20000.log
cargo test
```

<details>
<summary>Подсказка</summary>

Запускайте бинарники из `./target/release/`, а не через `cargo run`: в пакете два бинарника, `gen` и `p19-logstat`, поэтому `cargo run` без `--bin` не знает, какой запускать. Если падает `small_log_with_edge_cases`, проверьте в `small.log` пустую строку 4 и два пробела в строке 9.

</details>

**Готово, когда:** `./target/release/gen 3` печатает три строки журнала из раздела 2, а `cargo test` заканчивается так:

```text
test small_log_with_edge_cases ... ok
test generated_log_matches_the_original_output ... ok
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.53s
```

Коммит: `git add code/p19-logstat` и `git commit -m "p19-logstat: slow original with golden tests"` (из `~/projects/rust`).

#### M2 — Измерение базовой линии

**Цель:** вы знаете, насколько быстра исходная программа, и храните её копию для сравнения.

```bash
./target/release/gen 1000000 > big.log
./target/release/p19-logstat big.log
cp target/release/p19-logstat logstat-orig
hyperfine -N --warmup 1 --runs 5 './logstat-orig big.log'
```

`big.log` занимает около 67 МБ (мегабайт). Анализатор должен напечатать в точности это (если нет, ваш `gen.rs` отличается от урока):

```text
lines: 1000000
bad lines: 1003
status 2xx: 500101
status 3xx: 124344
status 4xx: 249390
status 5xx: 125162
unique clients: 5000
top endpoints (requests, p50, p95, p99 in ms, average bytes):
  GET /                           41992    10.305    19.412    20.195   24989
  GET /api/products/{id}          41875    10.256    19.329    20.182   24915
  GET /static/style.css           41796    10.362    19.364    48.355   24991
  GET /api/users/{id}/orders      41788    10.345    19.398    20.199   25125
  GET /api/users/{id}             41727    10.368    19.427    20.187   24993
  GET /api/users                  41723    10.335    19.386    44.195   25208
  GET /api/search                 41654    10.299    19.387    20.191   24918
  GET /api/orders/{id}            41646    10.245    19.380    26.649   24987
  GET /static/app.js              41599    10.397    19.396    52.844   25030
  GET /health                     41577    10.296    19.392    20.184   24986
top clients:
  10.0.0.6           31716
  10.0.0.0           31650
  10.0.0.4           31617
  10.0.0.7           31483
  10.0.0.5           31327
```

Начните `README.md` с таблицы и запишите в неё базовую линию:

```text
| Step | Change | Time (mean ± σ) | Speed-up |
|---|---|---|---|
| 0 | original | 3.54 s ± 0.30 s | 1.0× |
```

<details>
<summary>Подсказка</summary>

На M4 базовая линия была такой:

```text
Benchmark 1: ./logstat-orig big.log (5 runs)
                mean     ±       σ          min     …     max
  Wall Time    3.539 s   ±   0.301 s      3.313 s   …   4.050 s
```

Ваше число будет другим; важно, чтобы каждое следующее число измерялось так же и на том же компьютере.

</details>

**Готово, когда:** вывод выше совпадает в точности, `logstat-orig` существует, а в README есть строка базовой линии. Коммит.

#### M3 — Профилирование и поиск горячей точки

**Цель:** вы знаете, какие строки `src/lib.rs` занимают больше всего времени, — по профилю, а не по догадке.

Добавьте в `Cargo.toml` профиль для профилирования из Урока 19.3, соберите и запишите профиль:

```toml
[profile.profiling]
inherits = "release"
debug = true
```

```bash
cargo build --profile profiling
samply record --save-only -o slow.json.gz ./target/profiling/p19-logstat big.log > /dev/null
samply load slow.json.gz
```

В Firefox Profiler откройте *Call Tree*, отметьте *Invert call stack* и посмотрите на верхние строки. Затем снимите отметку и пройдите от `main` вниз по самым большим числам. Двойной щелчок по самой горячей функции показывает строки исходного кода с числом сэмплов.

Если вы предпочитаете текст, `sample` из Урока 19.3 даёт тот же ответ:

```bash
./target/profiling/p19-logstat big.log > /dev/null &
sample $! 2 -mayDie -f slow.sample.txt
grep -A 8 'Sort by top of stack' slow.sample.txt | rustfilt
```

Запишите в README три функции с наибольшим *собственным* временем (self) и строку в `src/lib.rs`, которая отвечает за каждую.

<details>
<summary>Подсказка</summary>

На M4 наверху инвертированного дерева была сама `p19_logstat::run`, затем `_platform_memcmp` и `DYLD-STUB$$memcmp` («заглушка», stub, — крошечный переход в системную библиотеку). Вместе они дали около 80% всех сэмплов. `memcmp` сравнивает байты памяти; спросите себя, какая строка снова и снова сравнивает строки и сколько раз на одну строку журнала. Следом идут функции аллокатора (`_xzm_xzone_malloc_tiny`, `_xzm_free_main`) и `mach_absolute_time`.

</details>

**Готово, когда:** README называет горячую точку и строку кода за ней и одним предложением объясняет, *почему* она медленная (подумайте: сколько сравнений на строку журнала при 5 000 клиентов?). Коммит.

#### M4 — Исправление самой большой горячей точки

**Цель:** убрать горячую точку из M3 одним точечным изменением, при зелёных тестах.

Меняйте только то, на что указал профиль. Затем:

```bash
cargo test
cargo build --release
hyperfine -N --warmup 1 --runs 5 './logstat-orig big.log' './target/release/p19-logstat big.log'
```

<details>
<summary>Подсказка</summary>

`Vec`, который для каждой строки просматривается с начала, — это **линейный поиск**. Структура данных из Урока 04.9 находит ключ примерно за постоянное время. Порядок вывода не зависит от того, как вы храните счётчики, потому что его определяет итоговая сортировка (количество, затем IP). Для подсчёта используйте API `entry` (API — Application Programming Interface, программный интерфейс; здесь методы `entry` у `HashMap`).

</details>

**Готово, когда:** `cargo test` проходит, а `hyperfine` показывает явное ускорение. На M4 одно это изменение дало около 5×:

```text
Benchmark 1: orig (5 runs)
  Wall Time    4.061 s   ±   0.582 s      3.419 s   …   4.772 s
Benchmark 2: first-fix (5 runs)
  Wall Time    0.838 s   ±   0.104 s      0.741 s   …   0.972 s            -79.4%  (4.8x faster)
```

(Здесь командам даны короткие имена через `-n orig` и `-n first-fix`.) Добавьте строку в README. Коммит.

#### M5 — Снова профилирование, до 10×

**Цель:** достичь ускорения как минимум в 10 раз, по одному измеренному шагу.

Профилируйте новую версию точно так же, как в M3. Картина изменилась: старая горячая точка исчезла, и наверх вышло то, что пряталось за ней. Повторяйте цикл из раздела 1: выберите самую большую из оставшихся горячих точек, измените одну вещь, `cargo test`, измерьте, добавьте строку в README, сделайте коммит.

<details>
<summary>Подсказка 1 — куда смотреть</summary>

После M4 большая часть оставшегося времени уходит на **выделение памяти**: `malloc`, `free` и копирование байтов (`memmove`). Посчитайте, сколько выделений в куче делает цикл на *одну* строку журнала: `String` из `lines()`, `Vec<String>` и его семь `String`, три `clone()`, `normalize` (`String`, `Vec<String>`, по `String` на каждый сегмент, `join`) и два вызова `format!`. Урок 19.5 показал, как измерить это с помощью `dhat`.

</details>

<details>
<summary>Подсказка 2 — идеи, по одной за раз</summary>

- Работайте со срезами `&str` строки вместо владеющих `String`; `split(' ')` уже даёт срезы.
- Прочитайте весь файл в одну `String` (`std::fs::read_to_string`) и идите по ней через `.lines()` срезами или переиспользуйте одну `String` с `read_line`.
- Собирайте ключ эндпоинта в **один переиспользуемый** буфер `String` (`clear()`, затем `push_str`) и ищите его в `HashMap` через `get_mut(key.as_str())`; новую `String` выделяйте, только когда ключ новый.
- Считайте классы статусов в маленьком массиве вместо `HashMap<String, u64>` с `format!` на каждую строку.
- В конце сортируйте каждый список задержек **один раз** и берите из него все три перцентиля.

</details>

<details>
<summary>Подсказка 3 — ловушки</summary>

- И `str::lines()`, и `BufRead::lines()` убирают завершающий `\r`; `split('\n')` — нет. Выберите способ, который даёт те же строки.
- Сохраните `split(' ')` и правило «ровно 7 полей». С итератором проверьте, что 8-го поля нет.
- Оставьте `parse::<f64>()` для задержки: собственный парсер должен округлять в точности как `(x * 1000.0).round()` для любого входа, а это сложнее, чем кажется.

</details>

**Готово, когда:** `cargo test` проходит, а `hyperfine` показывает, что ваша версия как минимум `10.0x faster`, чем `./logstat-orig` (пример формата: `-90.4%  (10.5x faster)`). В README по строке на каждый шаг. Аккуратная версия только на стандартной библиотеке достигает на M4 примерно 20×. Коммит.

#### M6 — Наведение порядка и выводы

**Цель:** код чистый, а README рассказывает историю.

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```

Допишите README: таблица шагов, находки профиля в каждом круге и два-три предложения о том, какое изменение дало больше всего и почему (используйте закон Амдала: почему после первого исправления второе дало гораздо больше, чем дало бы в самом начале?).

**Готово, когда:** все три команды проходят — `fmt` ничего не выводит, у Clippy нет предупреждений; `logstat-orig`, `big.log` и файлы профилей не попали в Git (`git status` их не показывает), а README закончен. Коммит.

### Дополнительные цели

- **20× и больше**, по-прежнему только на стандартной библиотеке. Профилируйте после каждого шага.
- **Меньше выделений, с измерением:** используйте `dhat` (Урок 19.5), чтобы показать число выделений до и после; стремитесь к числу, которое не растёт с числом строк.
- **Настройки сборки:** попробуйте `lto`, `codegen-units = 1` и PGO (profile-guided optimization, оптимизация по профилю) из Урока 19.8 на итоговой версии. Больше ли выигрыш, чем шум?
- **Параллельно:** разделите файл на куски по границам строк, обработайте их в нескольких потоках с `std::thread::scope` (Урок 14.4; `rayon` нарушил бы правило контракта «только стандартная библиотека») и объедините результаты. Вывод должен остаться байт в байт тем же, поэтому продумайте, как при объединении сохранить все счётчики и списки задержек полными.
- **Отображение в память:** читайте файл через отображение в память (Урок 13.11) вместо `read_to_string` и измерьте, помогает ли это.

## Вопросы для самопроверки

1. Почему золотые тесты пишут *до* того, как менять код?

<details>
<summary>Ответ</summary>

Они замораживают текущее поведение. Тогда каждое следующее изменение проверяется автоматически: если вывод изменился хоть на байт, тест падает, и вы знаете, что этот шаг что-то сломал.

</details>

2. Профиль показывает `memcmp` почти наверху, но ваш код никогда не вызывает `memcmp`. Откуда он?

<details>
<summary>Ответ</summary>

Из сравнения строк: `==` для двух `String` сравнивает их байты через `memcmp`. В исходной программе линейный поиск по 5 000 IP-адресам клиентов делает тысячи таких сравнений на каждую строку журнала.

</details>

3. Первое исправление убрало около 80% времени. Какое наибольшее ускорение оно могло дать по закону Амдала и сколько дало?

<details>
<summary>Ответ</summary>

Не больше 1 / (1 − 0,8) = 5×. Оно дало около 4,8×, близко к пределу. Чтобы идти дальше, нужно ускорять оставшиеся 20%.

</details>

4. Почему программа хранит задержки в целых микросекундах, а не в миллисекундах `f64`?

<details>
<summary>Ответ</summary>

Целые числа точны. Перцентили целых чисел не зависят от округления, и оптимизированная версия, которая собирает значения в другом порядке, всё равно печатает те же числа. С числами с плавающей точкой другой порядок операций может изменить последние цифры (Урок 19.7).

</details>

5. Зачем хранить `logstat-orig` и измерять оба бинарника в одной команде `hyperfine`?

<details>
<summary>Ответ</summary>

Оба работают в одинаковых условиях (в тот же момент, при той же загрузке машины, с тем же файлом в кэше), поэтому сравнение честное, а `hyperfine` сам считает ускорение.

</details>

## Итоги

- Работа над производительностью — это цикл: заморозить вывод, измерить, профилировать, изменить одну вещь, проверить тестами, измерить, сделать коммит.
- Золотые тесты на весь вывод превращают вопрос «всё ещё правильно?» в команду.
- Следующее изменение выбирает профиль, а не интуиция; после каждого исправления картина меняется.
- Типичные большие выигрыши в таких программах: правильная структура данных вместо линейного поиска и гораздо меньше выделений памяти на элемент.
- Закон Амдала говорит, сколько исправление *может* дать; README с измеренной строкой на каждый шаг показывает, сколько оно *дало*.

## Источники и что почитать

- [The Rust Performance Book](https://nnethercote.github.io/perf-book/) — особенно главы «Heap Allocations», «Hashing» и «Standard Library Types».
- [samply на GitHub](https://github.com/mstange/samply) — профилировщик из вехи M3.
- [`std::collections::HashMap`](https://doc.rust-lang.org/std/collections/struct.HashMap.html) — API `entry` и `get_mut` с заимствованными ключами.
- [The Cargo Book — Environment variables (`CARGO_BIN_EXE_<name>`)](https://doc.rust-lang.org/cargo/reference/environment-variables.html) — как интеграционные тесты находят бинарники пакета.
- [hyperfine на GitHub](https://github.com/sharkdp/hyperfine) — параметры для сравнения команд.

---

[🏠 Начало](../START-HERE.md) · [← 19.9 Производительность асинхронного кода и ввода-вывода](09-async-and-io-performance.md) · [📋 Этап 19](README.md) · [19.11 Контрольная точка →](11-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
