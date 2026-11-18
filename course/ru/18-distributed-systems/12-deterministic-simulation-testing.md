<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 18.11 Raft: репликация журнала](11-raft-log-replication.md) · [📋 Этап 18](README.md) · [18.13 Распределённые транзакции и саги →](13-distributed-transactions-and-sagas.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/18-distributed-systems/12-deterministic-simulation-testing.md) <!-- nav -->

# Урок 18.12 — Детерминированное тестирование симуляцией

> **Этап 18 — Распределённые системы** · Урок 12 из 15 · ⏱ около 135 минут

## Что вы узнаете

- Что такое *детерминированное тестирование симуляцией* и зачем оно распределённым системам: с настоящей сетью тесты медленные, нестабильные, и их невозможно воспроизвести.
- Как крейт `turmoil` запускает целый кластер программ на Tokio в одном потоке, с симулированными часами и симулированной сетью.
- Как добавлять трудности: разделения сети, медленные и неравномерные задержки сообщений, падения с последующим перезапуском.
- Как зерно делает запуск повторяемым и как цикл по множеству зёрен находит настоящую ошибку повторных попыток.
- Чего симуляция найти не может и где здесь место проверке моделей.

## Перед началом

- Вы прошли [Урок 18.11 — Raft: репликация журнала](11-raft-log-replication.md). Там вы уже вручную выполняли маленькую детерминированную симуляцию: `Cluster` решал, когда проходит время и какие сообщения доходят.
- Вы знаете задачи Tokio, `timeout` и `sleep` ([Урок 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md), [Урок 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md)), TCP (Transmission Control Protocol, протокол управления передачей) в Tokio ([Урок 15.6](../15-networking/06-async-networking-with-tokio.md)), а также повторные попытки с задержками и идемпотентность ([Урок 18.3](03-retries-backoff-and-idempotency.md)).
- Создайте пакет:

```bash
cd ~/projects/rust
cargo new --lib code/p18-sim
cd code/p18-sim
cargo add tokio@1 --features full
cargo add --dev turmoil@0.7
```

`--dev` кладёт `turmoil` в `[dev-dependencies]`: он компилируется только для тестов и никогда не попадает в вашу настоящую программу.

## Новые слова

| Слово | Значение |
|---|---|
| **DST** (Deterministic Simulation Testing, детерминированное тестирование симуляцией) | Тестирование распределённой программы внутри симуляции, где время, сеть и случайность полностью под контролем теста, так что каждый запуск с тем же зерном в точности одинаков. |
| **недетерминированность** (non-determinism) | Всё, что может отличаться между двумя запусками одной и той же программы: планирование потоков, реальное время, случайные числа, порядок прихода сетевых пакетов. |
| **нестабильный тест** (flaky test) | Тест, который иногда проходит, а иногда падает без изменений кода (Урок 05.6). Тесты распределённого кода на настоящей сети часто нестабильны. |
| **зерно** (seed) | Число, с которого запускается генератор случайных чисел. То же зерно — те же «случайные» выборы. |
| **симулированное время** (simulated time) | Часы, которые симуляция двигает сама. Сон на 5 секунд симулированного времени почти не занимает реального времени. |
| **хост** (host) | В `turmoil` — симулированная машина, на которой работает серверная программа. Её можно уронить и перезапустить. |
| **клиент** (client) | В `turmoil` — симулированная машина, на которой работает код теста. Симуляция заканчивается, когда все клиенты завершились. |
| **разделение** (partition) | В `turmoil` — разрыв связи между двумя хостами в обе стороны (или в одну сторону, через `partition_oneway`). |
| **перезапуск** (bounce) | Перезапустить упавший хост: его программа начинается заново, с пустой памятью. |
| **трудности** (hardship) | Слово, которым `turmoil` называет внесённые проблемы: задержки, потерянные сообщения, разделения, падения. |
| **проверка моделей** (model checking) | Другая техника: инструмент перебирает *все* возможные порядки событий маленькой модели системы вместо случайных запусков. |

## 1. Зачем симулировать?

Представьте, что вы тестируете кластер Raft из двух прошлых уроков настоящими процессами на настоящих сокетах. Чтобы проверить «разделение сети посреди записи», нужно остановить сетевой трафик ровно в нужную миллисекунду. Тесту нужны настоящие тайм-ауты, поэтому он идёт секунды. А когда он падает один раз из тысячи, вы не можете воспроизвести именно этот запуск: планирование потоков, точность таймеров и порядок пакетов были другими.

Детерминированная симуляция убирает все источники недетерминированности:

| Источник недетерминированности | В симуляции |
|---|---|
| Потоки, работающие параллельно | Всё выполняется в **одном** потоке, в порядке, который выбирает симулятор. |
| Настоящие часы | **Симулированные часы**, которые двигаются, только когда их двигает симулятор. |
| Сеть | **Симулированная сеть**, которая доставляет, задерживает, теряет или блокирует сообщения так, как решит тест. |
| Случайные числа | Каждый случайный выбор берётся из генератора, запущенного с **одного зерна**. |

Результат: тест, который гоняет кластер минуты симулированного времени за миллисекунды реального, может ломать сеть как угодно, а когда падает, печатает зерно, которое воспроизводит **тот же сбой** сколько угодно раз. Именно так, как известно, была построена база данных FoundationDB, а TigerBeetle круглые сутки гоняет симулятор вокруг своей базы (см. источники).

Вы это уже делали. `Cluster` из Уроков 18.10–18.11 был детерминированной симуляцией: ни потоков, ни часов, очередь сообщений, одно зерно. Это работает, когда протокол написан в стиле sans-IO. Крейт `turmoil` переносит ту же идею на **обычный код на Tokio**, который использует сокеты, `sleep` и `timeout`.

## 2. Сервер и клиент под `turmoil`

`turmoil` даёт вам `Sim` — симулированный мир. В нём вы регистрируете **хосты** (серверные программы, записанные как асинхронные замыкания) и **клиентов** (код теста). Каждый получает своё имя, фиктивный IP-адрес (Internet Protocol, межсетевой протокол) и собственный однопоточный рантайм Tokio, чьи часы — симулированные часы. `sim.run()` выполняет всё, пока не завершатся все клиенты.

Тестируемый код должен использовать `turmoil::net::{TcpListener, TcpStream}` вместо `tokio::net`. У них те же методы, поэтому хватает одного переключателя `#[cfg]`. Вот наша система — крошечный сервер «ключ-значение» и клиент, который повторяет попытки. Замените `src/lib.rs`:

```rust
//! A tiny key-value server and a client with retries.

// Real network in the program, simulated network in the tests.
#[cfg(not(test))]
use tokio::net::{TcpListener, TcpStream};
#[cfg(test)]
use turmoil::net::{TcpListener, TcpStream};

#[cfg(test)]
mod sim;

use std::collections::HashMap;
use std::io;
use std::time::Duration;
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::time::{sleep, timeout};

pub const PORT: u16 = 7000;

/// The server's data.
#[derive(Default)]
pub struct Store {
    values: HashMap<String, i64>,
}

impl Store {
    /// Apply one request line and return the answer line.
    pub fn handle(&mut self, line: &str) -> String {
        let parts: Vec<&str> = line.split_whitespace().collect();
        match parts.as_slice() {
            ["SET", key, value] => match value.parse() {
                Ok(v) => {
                    self.values.insert(key.to_string(), v);
                    "OK".to_string()
                }
                Err(_) => "ERR not a number".to_string(),
            },
            ["GET", key] => match self.values.get(*key) {
                Some(v) => v.to_string(),
                None => "NONE".to_string(),
            },
            ["INCR", key] => self.incr(key),
            _ => "ERR unknown request".to_string(),
        }
    }

    fn incr(&mut self, key: &str) -> String {
        let v = self.values.entry(key.to_string()).or_insert(0);
        *v += 1;
        v.to_string()
    }
}

/// Serve forever: one request line and one answer line per connection.
pub async fn serve() -> io::Result<()> {
    let listener = TcpListener::bind(("0.0.0.0", PORT)).await?;
    let mut store = Store::default();
    loop {
        let (stream, _) = listener.accept().await?;
        let mut stream = BufReader::new(stream);
        let mut line = String::new();
        // A client that never finishes its line must not block us forever.
        match timeout(Duration::from_secs(1), stream.read_line(&mut line)).await {
            Ok(Ok(n)) if n > 0 => {
                let answer = store.handle(line.trim());
                let _ = stream.write_all(format!("{answer}\n").as_bytes()).await;
            }
            _ => continue, // broken or slow connection: drop it
        }
    }
}

/// Send one request and read the answer, with a time limit.
pub async fn request(host: &str, line: &str) -> io::Result<String> {
    let exchange = async {
        let stream = TcpStream::connect((host, PORT)).await?;
        let mut stream = BufReader::new(stream);
        stream.write_all(format!("{line}\n").as_bytes()).await?;
        let mut answer = String::new();
        if stream.read_line(&mut answer).await? == 0 {
            return Err(io::Error::new(io::ErrorKind::UnexpectedEof, "no answer"));
        }
        Ok(answer.trim().to_string())
    };
    match timeout(Duration::from_millis(500), exchange).await {
        Ok(result) => result,
        Err(_) => Err(io::Error::new(io::ErrorKind::TimedOut, "timed out")),
    }
}

/// Retry with exponential backoff (Lesson 18.3): 100 ms, 200 ms, 400 ms, ...
pub async fn request_with_retry(host: &str, line: &str, attempts: u32) -> io::Result<String> {
    let mut delay = Duration::from_millis(100);
    let mut last_error = io::Error::other("no attempt made");
    for _ in 0..attempts {
        match request(host, line).await {
            Ok(answer) => return Ok(answer),
            Err(e) => last_error = e,
        }
        sleep(delay).await;
        delay *= 2;
    }
    Err(last_error)
}
```

**Что происходит:**

1. Две строки `use` с `#[cfg(...)]` выбирают сетевые типы: `tokio::net`, когда крейт собирается обычным образом, и `turmoil::net`, когда он компилируется для собственных модульных тестов. `#[cfg(test)] mod sim;` добавляет тестовый модуль (его мы напишем дальше) только в тестовых сборках.
2. `Store::handle` — конечный автомат: `SET ключ число`, `GET ключ` и `INCR ключ`. Это обычная функция, её легко протестировать саму по себе.
3. `serve` принимает соединения по одному. Каждое соединение несёт одну строку запроса и получает одну строку ответа. `timeout` на 1 секунду вокруг `read_line` защищает сервер от клиента, который подключился и так и не прислал полную строку (например, потому что сломалась сеть).
4. `request` подключается, пишет и читает одну строку — всё внутри `timeout` на 500 мс. Закрытое соединение без ответа (`read_line` вернул 0) — тоже ошибка.
5. `request_with_retry` повторяет попытки с экспоненциальной задержкой, как в Уроке 18.3. (Без джиттера, чтобы пример был коротким. В симуляции джиттер тоже должен браться из генератора с зерном.)

Теперь первая симуляция. Создайте `src/sim.rs`:

```rust
use super::*;
use std::cell::RefCell;
use std::rc::Rc;
use turmoil::Builder;

/// Register the server program on the host called "server".
fn add_server(sim: &mut turmoil::Sim) {
    sim.host("server", || async {
        serve().await?;
        Ok(())
    });
}

#[test]
fn set_then_get() -> turmoil::Result {
    let mut sim = Builder::new().rng_seed(1).build();
    add_server(&mut sim);
    sim.client("client", async {
        assert_eq!(request("server", "SET apples 10").await?, "OK");
        assert_eq!(request("server", "GET apples").await?, "10");
        assert_eq!(request("server", "GET pears").await?, "NONE");
        Ok(())
    });
    sim.run()
}
```

**Что происходит:**

1. `add_server` регистрирует хост с именем `server`. Замыкание при каждом вызове возвращает новый future; `turmoil` вызывает его снова, когда хост перезапускается. `serve().await?` через `?` превращает `io::Error` в тип ошибки `turmoil::Result`.
2. `Builder::new().rng_seed(1).build()` создаёт мир. **Всегда задавайте зерно**; без него `turmoil` берёт случайное зерно у операционной системы, и запуски различаются (раздел 4 это покажет).
3. Клиент — это код теста. `"server"` работает как имя хоста: в `turmoil` есть маленький встроенный поиск имён.
4. Тест возвращает `turmoil::Result`, поэтому `?` работает и внутри клиента, и в самом тесте.

Запустите `cargo test`:

```text
test sim::set_then_get ... ok
```

Ничего настоящего здесь нет: ни один порт на вашем Mac не открылся, а весь обмен занял микросекунды реального времени.

## 3. Разделения сети

Трудности можно вносить изнутри клиента (функциями вроде `turmoil::partition("a", "b")`) или снаружи, между **шагами** симуляции. `sim.step()` продвигает мир на один тик (по умолчанию 1 мс симулированного времени) и возвращает `true`, когда все клиенты завершились. Цикл вокруг него позволяет тесту действовать в точные моменты. Добавьте в `src/sim.rs`:

```rust
#[test]
fn a_retry_survives_a_partition() -> turmoil::Result {
    let mut sim = Builder::new().rng_seed(1).build();
    add_server(&mut sim);
    sim.client("client", async {
        let answer = request_with_retry("server", "SET apples 10", 6).await?;
        assert_eq!(answer, "OK");
        Ok(())
    });

    sim.partition("client", "server");
    let mut repaired = false;
    while !sim.step()? {
        if !repaired && sim.elapsed() >= Duration::from_secs(1) {
            sim.repair("client", "server");
            repaired = true;
        }
    }
    println!("done after {:?} of simulated time", sim.elapsed());
    Ok(())
}
```

**Что происходит:**

1. Клиент отправляет один `SET` с не более чем 6 попытками.
2. Перед первым шагом тест рвёт связь между `client` и `server`. Каждая попытка неудачна: подключение через разделение сразу отклоняется, затем клиент спит 100, 200, 400 мс…
3. В цикле шагов, как только прошла 1 секунда симулированного времени, тест восстанавливает связь. Следующая попытка удаётся.
4. `sim.elapsed()` сообщает симулированное время.

Запустите с видимым выводом: `cargo test partition -- --nocapture`:

```text
running 1 test
done after 1.551s of simulated time
test sim::a_retry_survives_a_partition ... ok
```

Попытки были примерно в 0; 0,1; 0,3; 0,7 и 1,5 секунды (суммы задержек), поэтому пятая, сразу после восстановления, удалась. А тест всё равно закончился за несколько миллисекунд реального времени.

> 💡 **Совет:** по умолчанию симуляция может идти 10 секунд симулированного времени; после этого `run()` и `step()` возвращают ошибку. Для более длинных сценариев увеличьте предел через `Builder::simulation_duration`.

## 4. Задержки и зерно

Настоящие сети медленные и неравномерные. `min_message_latency` и `max_message_latency` заставляют каждое сообщение идти случайное время между двумя значениями. Теперь `timeout` на 500 мс в `request` иногда будет срабатывать, и повторные попытки пригодятся. Добавьте вспомогательную функцию, которая считает 20 посещений, и тест, который запускает её дважды на каждое зерно:

```rust
/// 20 increments over a slow network. Returns the final count and the time taken.
fn count_visits(seed: u64) -> (String, Duration) {
    let mut sim = Builder::new()
        .rng_seed(seed)
        .min_message_latency(Duration::from_millis(20))
        .max_message_latency(Duration::from_millis(300))
        .build();
    add_server(&mut sim);

    let result = Rc::new(RefCell::new(String::new()));
    let out = result.clone();
    sim.client("client", async move {
        for _ in 0..20 {
            request_with_retry("server", "INCR visits", 8).await?;
        }
        *out.borrow_mut() = request_with_retry("server", "GET visits", 8).await?;
        Ok(())
    });
    sim.run().expect("the simulation failed");
    let total = result.borrow().clone();
    (total, sim.elapsed())
}

#[test]
fn the_same_seed_gives_the_same_run() {
    for seed in [1, 2, 3] {
        let first = count_visits(seed);
        let second = count_visits(seed);
        println!("seed {seed}: {first:?}");
        assert_eq!(first, second);
    }
}
```

**Что происходит:**

1. Каждое сообщение идёт от 20 до 300 мс. Запросу нужно несколько сообщений (подключение, запрос, ответ), поэтому некоторые запросы идут дольше 500 мс и истекают по тайм-ауту.
2. Future клиента должен быть `'static`, поэтому он не может заимствовать локальную переменную. Он получает клон `Rc<RefCell<String>>` и записывает в него итоговое число; тест читает его после `run()`. (`Rc` здесь подходит, потому что всё выполняется в одном потоке.)
3. `the_same_seed_gives_the_same_run` запускает каждое зерно дважды и сравнивает результаты, включая точное симулированное время.

```text
seed 1: ("20", 4.421s)
seed 2: ("20", 4.041s)
seed 3: ("21", 5.13s)
```

Обратите внимание на две вещи. Во-первых, два запуска с одним зерном одинаковы до миллисекунды. Во-вторых, **зерно 3 насчитало 21 посещение**, хотя клиент отправил 20. Вы нашли ошибку, даже не ища её.

Детерминированность зависит от зерна. Уберите `.rng_seed(seed)` из `count_visits` и запустите тест дважды. Каждый раз он падает, с другим симулированным временем:

```text
seed 1: ("20", 4.85s)

thread 'sim::the_same_seed_gives_the_same_run' (6353819) panicked at src/sim.rs:78:9:
assertion `left == right` failed
```

Верните зерно на место.

## 5. Множество зёрен находит ошибку повторов

Превратите наблюдение в свойство и проверьте его на 50 зёрнах. Добавьте:

```rust
#[test]
fn every_increment_is_counted_once() {
    for seed in 0..50 {
        let (total, _) = count_visits(seed);
        assert_eq!(total, "20", "seed {seed}");
    }
}
```

```text
---- sim::every_increment_is_counted_once stdout ----

thread 'sim::every_increment_is_counted_once' (6362447) panicked at src/sim.rs:86:9:
assertion `left == right` failed: seed 3
  left: "21"
 right: "20"
```

21 насчитывают 11 из 50 зёрен. Что случилось в зерне 3? Запрос дошёл до сервера, тот увеличил счётчик и отправил ответ, но ответ шёл медленно, и тайм-аут клиента на 500 мс сработал раньше. Клиент не мог знать, что увеличение уже произошло, поэтому повторил попытку, и сервер посчитал ещё раз. Это ровно проблема Урока 18.3: **`INCR` не идемпотентен, поэтому повторять его небезопасно**. Тайм-аут означает «я не знаю», а не «этого не произошло».

Исправление из Урока 18.3 — **ключ идемпотентности** (idempotency key): клиент даёт каждой логической операции уникальный идентификатор и использует **тот же** идентификатор при каждом повторе. Сервер запоминает ответ для каждого идентификатора и возвращает его снова вместо повторной работы. В `src/lib.rs` добавьте поле в `Store`:

```rust
pub struct Store {
    values: HashMap<String, i64>,
    /// Request id -> the answer we gave. Used to recognise retries.
    answered: HashMap<String, String>,
}
```

и вторую ветку `INCR` в `handle`, сразу после первой:

```rust
            ["INCR", key, id] => {
                if let Some(answer) = self.answered.get(*id) {
                    return answer.clone(); // a retry: do not count it again
                }
                let answer = self.incr(key);
                self.answered.insert(id.to_string(), answer.clone());
                answer
            }
```

В `count_visits` отправляйте идентификатор с каждым увеличением:

```rust
        for i in 0..20 {
            let line = format!("INCR visits client-{i}");
            request_with_retry("server", &line, 8).await?;
        }
```

Снова запустите `cargo test`: `every_increment_is_counted_once` проходит на всех 50 зёрнах, а `the_same_seed_gives_the_same_run` теперь печатает для зерна 3 `("20", 5.13s)`.

> 🔒 **Безопасность:** словарь `answered` растёт с каждым запросом и заполняется клиентами. Клиент, отправивший миллионы разных идентификаторов, может исчерпать память сервера. Настоящие системы хранят идентификаторы только ограниченное время или в ограниченном количестве на клиента и аутентифицируют клиентов, чтобы один не мог использовать идентификаторы другого.

> ⚡ **Производительность:** 50 зёрен симулируют около 4 минут сетевого времени, а тест занимает около 4 секунд реального времени в отладочной сборке. Запускайте такие тесты с `--release` в CI (continuous integration, непрерывная интеграция), чтобы перебирать тысячи зёрен.

## 6. Падения

Третий вид трудностей: машина умирает. `sim.crash("server")` останавливает программу хоста и рвёт её соединения. `sim.bounce("server")` запускает программу снова **с самого начала**: замыкание в `add_server` вызывается снова, и новый `serve()` стартует с новым, пустым `Store`. Добавьте:

```rust
#[test]
fn a_crash_loses_an_acknowledged_write() -> turmoil::Result {
    let mut sim = Builder::new().rng_seed(1).build();
    add_server(&mut sim);
    sim.client("client", async {
        assert_eq!(request("server", "SET apples 10").await?, "OK");
        tokio::time::sleep(Duration::from_secs(2)).await;
        let answer = request_with_retry("server", "GET apples", 5).await?;
        assert_eq!(
            answer, "10",
            "the server said OK, so the write must survive"
        );
        Ok(())
    });

    let (mut crashed, mut restarted) = (false, false);
    while !sim.step()? {
        if !crashed && sim.elapsed() >= Duration::from_secs(1) {
            sim.crash("server");
            crashed = true;
        }
        if !restarted && sim.elapsed() >= Duration::from_millis(1500) {
            sim.bounce("server");
            restarted = true;
        }
    }
    Ok(())
}
```

**Что происходит:**

1. Клиент записывает `apples = 10` и получает `OK`. Затем ждёт 2 секунды.
2. На 1-й секунде тест роняет сервер; на 1,5-й перезапускает его.
3. На 2-й секунде клиент читает `apples` с повторами и ожидает 10, потому что сервер **подтвердил** запись.

```text
---- sim::a_crash_loses_an_acknowledged_write stdout ----

thread 'sim::a_crash_loses_an_acknowledged_write' (6362445) panicked at src/sim.rs:98:9:
assertion `left == right` failed: the server said OK, so the write must survive
  left: "NONE"
 right: "10"
```

Конечно: данные жили только в памяти. Это самая серьёзная ошибка, какая может быть у системы хранения: клиенту сказали «ваша запись в безопасности», а она потерялась. Исправляют её две вещи, и обе вы знаете: надёжно записывать данные на диск до ответа, с журналом упреждающей записи и `fsync` ([Урок 17.2](../17-storage-and-databases/02-durability-and-fsync.md), [Урок 17.4](../17-storage-and-databases/04-write-ahead-log.md)), и держать копии на других машинах, чтобы её не потерял и умерший диск: Raft. Соединить обе вещи — проект Урока 18.14.

Пока оставьте тест как **документацию известной ошибки**, чтобы набор тестов оставался зелёным, а ошибка оставалась на виду. Замените проверку:

```rust
        // A known bug, kept as a test: the write is gone after the restart.
        assert_eq!(answer, "NONE", "memory does not survive a crash");
```

Когда позже вы исправите ошибку, этот тест упадёт, и вы вернёте проверку обратно.

> 🧪 **Тестирование:** `turmoil` умеет симулировать и диск, с фичей `unstable-fs`: записи остаются в буфере ожидания до `sync_all()`, а `crash` выбрасывает всё, что не было синхронизировано. Она помечена как нестабильная (её API, Application Programming Interface, программный интерфейс, может измениться), поэтому курс её не использует, но это естественный следующий шаг для тестирования журнала упреждающей записи.

## 7. Чего симуляция не делает

Симуляция перебирает множество **случайных** запусков. Она очень хорошо находит ошибки, которые случаются в одном запуске из ста или тысячи. Она не доказывает, что ошибка невозможна. Вспомните ошибку с рисунка 8 из Урока 18.11: ей нужна определённая последовательность выборов и падений, и случайные запуски могут никогда на неё не попасть.

**Проверка моделей** дополняет симуляцию. Вы описываете маленькую модель протокола (состояния, сообщения, правила каждого шага), а средство проверки моделей посещает **каждое** достижимое состояние этой модели, например все порядки, в которых 3 узла могут отправить и получить 6 сообщений. Если свойство можно нарушить, оно находит кратчайшую последовательность событий, которая его нарушает. Крейт Rust `stateright` — такое средство, а язык TLA+ (Temporal Logic of Actions, темпоральная логика действий) широко используется в индустрии, в том числе для самого Raft. Цена: модель маленькая и отделена от настоящего кода, а число состояний растёт очень быстро. Используйте симуляцию для настоящего кода, а проверку моделей — для главных правил протокола.

## Частые ошибки

> ⚠️ **Частая ошибка:** забыть `rng_seed`. Без него `turmoil` берёт случайное зерно у операционной системы; два запуска различаются, и сбой нельзя воспроизвести (раздел 4). Всегда передавайте зерно и печатайте его в каждом сообщении проверки.

> ⚠️ **Частая ошибка:** использовать `tokio::net` (или `std::net`) в тестируемом коде. Эти типы открывают настоящие сокеты вне симуляции; тогда тест падает с непонятной ошибкой или, что хуже, обращается к настоящему порту на вашей машине. Каждый сокет в симулируемом коде должен браться из `turmoil::net`.

> ⚠️ **Частая ошибка:** настоящая недетерминированность внутри хоста: `std::time::SystemTime::now()`, `rand::rng()`, `std::thread::spawn` или обход `HashMap`, чтобы решить, что отправить. Тогда запуск зависит не только от зерна. Используйте `tokio::time` (оно симулируется), генераторы с зерном и отсортированные коллекции там, где важен порядок.

> ⚠️ **Частая ошибка:** ожидать, что `cfg(test)` переключит типы для интеграционных тестов в `tests/`. Такие тесты компилируют библиотеку *без* `cfg(test)`, поэтому библиотека использовала бы `tokio::net`. Именно поэтому тесты симуляции в этом уроке — модуль внутри крейта. Большие проекты используют фичу Cargo (например, `--features simulation`) или делают сеть трейтом.

## Попробуйте сами

1. Запустите `cargo test -- --nocapture` и найдите каждую строку, напечатанную тестами.
2. В `count_visits` поменяйте задержки на 0..10 мс. Насчитывает ли какое-нибудь зерно 21 без ключа идемпотентности? (Чтобы проверить, временно верните строку клиента к `"INCR visits"`.) Почему нет?
3. Добавьте `.enable_random_order()` в построитель в `count_visits`. Тогда симулятор на каждом шаге запускает хосты в случайном порядке, что исследует больше чередований. Проходят ли тесты по-прежнему?
4. В `a_retry_survives_a_partition` уменьшите число попыток с 6 до 4. Почему теперь тест падает? (Сложите задержки.)

## Упражнения

### Упражнение 1 — Тайм-аут — не сбой (легко)

Напишите тест в новом модуле `src/exercises.rs` (добавьте `#[cfg(test)] mod exercises;` в `src/lib.rs` и начните файл с тех же строк `use`, что и `src/sim.rs`). Внутри клиента заблокируйте только **ответы**: `turmoil::partition_oneway("server", "client")`. Отправьте `SET apples 10` через `request` и проверьте, что вид ошибки — `io::ErrorKind::TimedOut`. Затем восстановите связь через `turmoil::repair_oneway("server", "client")` и проверьте, что `GET apples` возвращает `"10"`.

<details>
<summary>Подсказка</summary>

`request(...).await.unwrap_err()` даёт `io::Error`; `.kind()` даёт её вид. Одностороннее разделение от сервера к клиенту пропускает запрос, но блокирует ответ, поэтому запись происходит, хотя клиент видит тайм-аут.

</details>

<details>
<summary>Ответ</summary>

```rust
use super::*;
use std::cell::RefCell;
use std::rc::Rc;
use turmoil::Builder;

#[test]
fn a_timeout_does_not_mean_the_request_failed() -> turmoil::Result {
    let mut sim = Builder::new().rng_seed(1).build();
    sim.host("server", || async {
        serve().await?;
        Ok(())
    });
    sim.client("client", async {
        // Requests reach the server, but no answer comes back.
        turmoil::partition_oneway("server", "client");
        let error = request("server", "SET apples 10").await.unwrap_err();
        assert_eq!(error.kind(), io::ErrorKind::TimedOut);

        turmoil::repair_oneway("server", "client");
        // The write happened anyway.
        assert_eq!(request("server", "GET apples").await?, "10");
        Ok(())
    });
    sim.run()
}
```

Почему это работает: сервер получил и применил `SET`; потерялся только его ответ. Тайм-аут клиента честен («я не получил ответа»), но ничего не говорит о сервере. Поэтому Урок 18.3 настаивает на идемпотентных повторах.

</details>

### Упражнение 2 — Тот же код, то же зерно, та же ошибка (средне)

Без ключа идемпотентности зерно 3 насчитало 21 посещение. Объясните своими словами, почему можно быть **уверенным**, что завтра на другом компьютере зерно 3 снова насчитает 21, тогда как тест того же кода на настоящей сети может пройти сто раз подряд.

<details>
<summary>Подсказка</summary>

Перечислите всё, что определяло время прихода сообщений в зерне 3. Что из этого приходит извне симуляции?

</details>

<details>
<summary>Ответ</summary>

В симуляции задержки сообщений, порядок, в котором выполняются хосты, и каждый таймер берутся из симулированных часов и из одного генератора, запущенного с зерном 3; ничего не берётся с настоящей машины. Поэтому тот же код с тем же зерном делает те же выборы в том же порядке, и медленный ответ, вызвавший двойной подсчёт, повторяется. На настоящей сети время зависит от планировщика операционной системы, загрузки машины и настоящих задержек пакетов; неудачное время может случиться один раз на тысячу запусков и больше никогда, когда вы пытаетесь отладить.

</details>

### Упражнение 3 — Ключи идемпотентности не переживают падение (сложно)

В `src/exercises.rs` напишите тест, который выполняет 20 увеличений **с** идентификаторами (и паузой 100 мс после каждого), роняет сервер на 1-й секунде и перезапускает его на 1,2-й, затем печатает итоговое число. Какое число получается с зерном 1 и почему одни идентификаторы запросов это не исправят?

<details>
<summary>Подсказка</summary>

Скопируйте `count_visits` и цикл шагов из `a_crash_loses_an_acknowledged_write`. В тесте, который ничего не возвращает, используйте `while !sim.step().unwrap()`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[test]
fn a_crash_in_the_middle_of_the_increments() {
    let mut sim = Builder::new().rng_seed(1).build();
    sim.host("server", || async {
        serve().await?;
        Ok(())
    });
    let result = Rc::new(RefCell::new(String::new()));
    let out = result.clone();
    sim.client("client", async move {
        for i in 0..20 {
            let line = format!("INCR visits client-{i}");
            request_with_retry("server", &line, 8).await?;
            tokio::time::sleep(Duration::from_millis(100)).await;
        }
        *out.borrow_mut() = request_with_retry("server", "GET visits", 8).await?;
        Ok(())
    });
    let mut crashed = false;
    let mut restarted = false;
    while !sim.step().unwrap() {
        if !crashed && sim.elapsed() >= Duration::from_secs(1) {
            sim.crash("server");
            crashed = true;
        }
        if !restarted && sim.elapsed() >= Duration::from_millis(1200) {
            sim.bounce("server");
            restarted = true;
        }
    }
    println!("the server counted {}", result.borrow());
}
```

Запустите его через `cargo test crash_in_the_middle -- --nocapture`:

```text
the server counted 13
```

Почему это работает: до падения произошло около 7 увеличений. Перезапущенный сервер начинает с пустым словарём `values` **и** пустым словарём `answered`, поэтому старый счёт пропал, а вместе с ним и память о том, какие идентификаторы уже обработаны. Ключи идемпотентности защищают от повторов, только пока сервер их помнит. Чтобы пережить падения, и данные, и словарь `answered` должны быть в надёжном реплицированном хранилище: в конечном автомате кластера Raft, как в проекте.

</details>

## Вопросы для самопроверки

1. Назовите четыре источника недетерминированности, которые контролирует детерминированная симуляция.

<details>
<summary>Ответ</summary>

Планирование потоков (один поток, порядок симулятора), часы (симулированное время), сеть (симулированная доставка, задержки, разделения) и случайные числа (одно зерно).

</details>

2. Чем `host` отличается от `client` в `turmoil`?

<details>
<summary>Ответ</summary>

Хост выполняет серверную программу; он задаётся замыканием, создающим future, поэтому его можно уронить и перезапустить (`bounce`). Клиент один раз выполняет код теста; симуляция заканчивается, когда все клиенты завершились.

</details>

3. Что случится, если построить `Sim` без `rng_seed`?

<details>
<summary>Ответ</summary>

`turmoil` возьмёт зерно для генератора у операционной системы, поэтому каждый запуск делает другие случайные выборы (например, задержки сообщений). Тест перестаёт быть воспроизводимым.

</details>

4. Почему зерно 3 насчитало 21 посещение и что это исправило?

<details>
<summary>Ответ</summary>

Ответ пришёл после тайм-аута клиента, поэтому клиент повторил увеличение, которое уже произошло. Исправило это то, что каждое увеличение получило ключ идемпотентности, повторно используемый при повторах, а сервер для известного ключа возвращает сохранённый ответ.

</details>

5. Что остаётся от данных сервера в памяти после `sim.crash("server")` и `sim.bounce("server")`?

<details>
<summary>Ответ</summary>

Ничего. `bounce` запускает программу хоста заново с начала, поэтому вся память (здесь `Store`) начинается пустой. Пережить такое могло бы только надёжное хранилище.

</details>

6. Что может найти проверка моделей, что случайная симуляция может пропустить, и чего это стоит?

<details>
<summary>Ответ</summary>

Она перебирает все возможные порядки событий маленькой модели, поэтому в пределах модели гарантированно находит редкие последовательности (вроде рисунка 8 в Raft). Цена в том, что модель маленькая и отделена от настоящего кода, а число состояний растёт очень быстро.

</details>

## Итоги

- Детерминированное тестирование симуляцией запускает распределённую систему в одном потоке, с симулированными часами, симулированной сетью и одним зерном, поэтому каждый сбой можно точно воспроизвести.
- `turmoil` делает это для обычного кода на Tokio: зарегистрируйте хосты и клиентов в `Sim`, используйте `turmoil::net` вместо `tokio::net` и вызывайте `run()` или сами управляйте `step()`.
- Трудности: `partition` / `repair` (в том числе односторонние), диапазоны задержек сообщений, `crash` / `bounce`.
- Всегда задавайте `rng_seed`, печатайте зерно в проверках и прогоняйте свойство по множеству зёрен.
- Множество зёрен нашло настоящую ошибку: повтор неидемпотентного `INCR` после тайм-аута считает дважды. Ключи идемпотентности это исправляют, но только пока сервер их помнит.
- Падение показало, что подтверждённые записи в памяти теряются: нужны надёжное хранение и репликация.
- Симуляция находит вероятные ошибки в настоящем коде; проверка моделей (`stateright`, TLA+) доказывает корректность маленьких моделей.

## Источники и что почитать

- [turmoil на GitHub](https://github.com/tokio-rs/turmoil) — репозиторий крейта с тестами, показывающими TCP, UDP (User Datagram Protocol, протокол пользовательских датаграмм) и разделения, и примерами для `axum` и gRPC (фреймворк удалённого вызова процедур из Урока 16.9).
- [Документация turmoil](https://docs.rs/turmoil) — API: `Builder`, `Sim`, `net` и функции трудностей.
- [Simulation and Testing (FoundationDB)](https://apple.github.io/foundationdb/testing.html) — как команда FoundationDB с первого дня строила базу данных на детерминированной симуляции.
- [We Put a Distributed Database in the Browser (TigerBeetle)](https://tigerbeetle.com/blog/2023-07-11-we-put-a-distributed-database-in-the-browser/) — симулятор TigerBeetle с наглядной демонстрацией падений и разделений.
- [Stateright](https://www.stateright.rs/) — средство проверки моделей для распределённых систем, написанное на Rust, с книгой, в которой проверяются маленькие протоколы.
- [Jepsen](https://jepsen.io/) — тесты настоящих баз данных при разделениях сети; его отчёты показывают, какие ошибки были у настоящих систем.

---

[🏠 Начало](../START-HERE.md) · [← 18.11 Raft: репликация журнала](11-raft-log-replication.md) · [📋 Этап 18](README.md) · [18.13 Распределённые транзакции и саги →](13-distributed-transactions-and-sagas.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
