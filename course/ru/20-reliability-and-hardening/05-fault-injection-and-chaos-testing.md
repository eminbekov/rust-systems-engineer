<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 20.4 Miri, санитайзеры и loom в CI](04-miri-sanitizers-and-loom-in-ci.md) · [📋 Этап 20](README.md) · [20.6 Основы формальной верификации с Kani →](06-formal-verification-with-kani.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/20-reliability-and-hardening/05-fault-injection-and-chaos-testing.md) <!-- nav -->

# Урок 20.5 — Внедрение сбоев и хаос-тестирование

> **Этап 20 — Надёжность и защита** · Урок 5 из 11 · ⏱ около 135 минут

## Что вы узнаете

- Что такое внедрение сбоев и хаос-тестирование и какой метод за ними стоит: устойчивое состояние, гипотеза, внедрение, наблюдение.
- Как внедрять сбои *внутри* кода с помощью маленьких типов-обёрток и найти классический баг: предположение, что один `read` заполняет весь буфер.
- Как внедрять сетевые сбои (задержку, сброс соединения, пропавшую зависимость, данные крошечными кусками) с помощью toxiproxy в Docker, не меняя программу.
- Как превратить хаос-эксперимент в повторяемый скрипт со «стоп-кнопкой».

## Перед началом

- Вы прошли [Урок 20.4 — Miri, санитайзеры и loom в CI](04-miri-sanitizers-and-loom-in-ci.md).
- Вы знаете трейт `Read` и файлы из [Урока 13.4 — Файлы и файловые дескрипторы](../13-operating-systems/04-files-and-file-descriptors.md), асинхронный TCP (Transmission Control Protocol, протокол управления передачей) — из [Урока 15.6 — Асинхронная работа с сетью в Tokio](../15-networking/06-async-networking-with-tokio.md), а тайм-ауты — из [Урока 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md).
- Вы знаете Docker и `curl` из Этапа 11, а надёжные bash-скрипты — из [Урока 11.9](../11-linux-shell-and-containers/09-robust-bash-scripts.md). Docker Desktop должен быть запущен.
- Создайте пакет для этого урока:

```bash
cd ~/projects/rust
cargo new --lib code/p20-chaos
cd code/p20-chaos
cargo add tokio@1 --features full
mkdir src/bin
```

## Новые слова

| Слово | Значение |
|---|---|
| **внедрение сбоев** (fault injection) | Намеренное создание сбоев (ошибок, задержек, разорванных соединений), чтобы проверить, что программа их обрабатывает. |
| **хаос-тестирование** (chaos testing, chaos engineering) | Эксперименты, которые внедряют сбои в целую работающую систему, чтобы узнать, остаётся ли она здоровой. |
| **устойчивое состояние** (steady state) | Нормальное измеримое поведение системы: «все пинги успешны быстрее 50 мс». Эксперимент сравнивает с ним. |
| **гипотеза** (hypothesis) | Ожидаемый результат эксперимента, записанный до запуска: «с дополнительной задержкой 1 с каждый пинг завершается неудачей за 600 мс». |
| **короткое чтение** (short read) | Вызов `read`, который вернул меньше байтов, чем вмещает буфер. Это нормально для сокетов, каналов и медленных устройств. |
| **прокси** (proxy) | Программа, которая стоит между клиентом и сервером и передаёт трафик дальше. Прокси, внедряющий сбои, может ещё и менять его. |
| **toxiproxy** | TCP-прокси для тестирования от компании Shopify. Им управляют через HTTP (HyperText Transfer Protocol, протокол передачи гипертекста) API и добавляют к его соединениям *токсины* (toxics). |
| **токсин** (toxic) | Один вид сбоя в toxiproxy: `latency`, `reset_peer`, `slicer`, `timeout`, `bandwidth` и другие. |
| **upstream / downstream** | Направления трафика через прокси: upstream идёт от клиента к серверу, downstream — от сервера обратно к клиенту. |
| **API** (Application Programming Interface, программный интерфейс) | Здесь: HTTP-адреса, через которые программа (toxiproxy) позволяет другим программам управлять собой. |
| **день игр** (game day) | Запланированное занятие, на котором команда намеренно внедряет сбои в систему и отрабатывает реакцию. |

## 1. Проверяем отказы намеренно

Урок 20.1 сказал: неисправности будут. Тайм-ауты, повторные попытки и обработка ошибок — код, который выполняется, *только* когда что-то идёт не так, поэтому в обычных тестах он почти не работает. Идея внедрения сбоев проста: **пусть сбой случится сейчас, пока вы смотрите**, а не в три часа ночи.

Метод пришёл из науки и состоит из четырёх шагов:

1. **Устойчивое состояние.** Опишите нормальное поведение числом, которое можно измерить: «5 из 5 пингов успешны, каждый быстрее 50 мс».
2. **Гипотеза.** Запишите, что *должно* произойти при сбое: «с задержкой 1 с все пинги неудачны, и каждый сдаётся за 600 мс».
3. **Внедрение.** Вызовите ровно этот сбой и ничего больше.
4. **Наблюдение и сравнение.** Повела ли себя система так, как вы предсказали? Если нет, вы нашли слабое место раньше пользователей.

Внедрять сбои можно на трёх уровнях:

| Уровень | Как | Пример |
|---|---|---|
| Внутри кода | Тип-обёртка или тестовый двойник, который отказывает намеренно | `Read`, который возвращает по одному байту за вызов (раздел 2) |
| На сетевой границе | Прокси между вашей программой и её зависимостью | toxiproxy добавляет задержку 1 с (раздел 3) |
| Вся система | Остановка процессов, машин или сетевых связей | Убить одну копию сервиса в тестовом кластере (раздел 5) |

Начинайте с самого низкого уровня, на котором проблема видна: он самый быстрый и самый точный.

## 2. Внедрение сбоев внутри кода: короткие чтения

Формат сетевого сообщения часто начинается с его длины. Положите это в `src/lib.rs`:

```rust
//! Reading length-prefixed messages, and a reader that injects faults for tests.

use std::io::{self, Read};

/// Reads one message: a 4-byte big-endian length, then that many bytes.
pub fn read_message(source: &mut impl Read) -> io::Result<Vec<u8>> {
    let mut header = [0u8; 4];
    source.read(&mut header)?; // BUG: `read` may fill only part of the buffer
    let len = u32::from_be_bytes(header) as usize;
    let mut body = vec![0u8; len];
    source.read(&mut body)?; // BUG: the same here
    Ok(body)
}

/// A test helper that wraps a reader and returns at most `max_chunk` bytes per call,
/// like a slow network that delivers data in small pieces.
pub struct Choppy<R> {
    pub inner: R,
    pub max_chunk: usize,
}

impl<R: Read> Read for Choppy<R> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        let n = buf.len().min(self.max_chunk);
        self.inner.read(&mut buf[..n])
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    const MESSAGE: &[u8] = &[0, 0, 0, 5, b'h', b'e', b'l', b'l', b'o'];

    #[test]
    fn reads_a_message_from_memory() {
        let mut source = MESSAGE;
        assert_eq!(read_message(&mut source).unwrap(), b"hello");
    }

    #[test]
    fn reads_a_message_that_arrives_in_small_pieces() {
        let mut source = Choppy {
            inner: MESSAGE,
            max_chunk: 2,
        };
        assert_eq!(read_message(&mut source).unwrap(), b"hello");
    }
}
```

**Что происходит:**

1. `read_message` читает 4-байтовый заголовок, превращает его в длину через `u32::from_be_bytes` (порядок байтов big-endian, Урок 12.2) и читает столько байтов.
2. `Read::read(buf)` обещает только прочитать **хотя бы один байт** (или сообщить о конце либо ошибке), но *не* заполнить `buf`. Он возвращает, сколько байтов прочитал, а наш код это число выбрасывает. Для файла или среза в памяти `read` обычно заполняет буфер. Для TCP-сокета он возвращает то, что уже пришло: **короткое чтение**.
3. `Choppy` — обёртка, внедряющая сбои. Она реализует `Read`, передавая каждый вызов внутреннему читателю, но с буфером не больше `max_chunk` байтов. Это несколько строк, и они заставляют любой читатель вести себя как медленная сеть.
4. `&[u8]` сам реализует `Read`, поэтому `MESSAGE` может быть внутренним читателем.

```bash
cargo test -q --lib
```

```text
running 2 tests
. 1/2
tests::reads_a_message_that_arrives_in_small_pieces --- FAILED

failures:

---- tests::reads_a_message_that_arrives_in_small_pieces stdout ----

thread 'tests::reads_a_message_that_arrives_in_small_pieces' (6269709) panicked at src/lib.rs:47:9:
assertion `left == right` failed
  left: []
 right: [104, 101, 108, 108, 111]
...
test result: FAILED. 1 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Тест с памятью проходит; тест с кусками падает. Первый `read` получил только 2 байта, `[0, 0]`, два других байта заголовка остались `0`, поэтому длина оказалась 0, а тело — пустым. С настоящей медленной сетью эта функция время от времени возвращала бы неверные сообщения: тихий отказ.

Clippy знал это с самого начала. Его линт `unused_io_amount` по умолчанию — *ошибка*:

```bash
cargo clippy -q --lib
```

```text
error: read amount is not handled
 --> src/lib.rs:8:5
  |
8 |     source.read(&mut header)?; // BUG: `read` may fill only part of the buffer
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^
  |
  = help: use `Read::read_exact` instead, or handle partial reads
...
```

Исправление — `read_exact`: он вызывает `read` в цикле, пока буфер не заполнится, и возвращает ошибку `UnexpectedEof`, если данные кончились слишком рано:

```rust
    source.read_exact(&mut header)?;
```

```rust
    source.read_exact(&mut body)?;
```

После обоих изменений `cargo test -q --lib` сообщает `2 passed`.

> 🔒 **Безопасность:** `vec![0u8; len]` с длиной, взятой из сети, позволяет отправителю четырьмя байтами заставить вас выделить до 4 ГБ (гигабайт). Настоящий читатель сначала сверяет `len` с максимумом (Урок 10.12 и лимиты Урока 20.8).

> 🧪 **Тестирование:** типы-обёртки вроде `Choppy` дёшевы и быстры. Соберите небольшой набор для каждой границы ввода-вывода: одна выдаёт крошечные куски, другая прерывается, третья ломается после N байтов (Упражнение 2 строит вторую). Они работают с каждой функцией, которая принимает `impl Read` или `impl Write`, — ещё одна причина писать функции через трейты, а не через конкретные типы (Урок 09.3).

## 3. Сетевые сбои с toxiproxy

Обёрткам нужны изменения кода. Иногда хочется проверить настоящую, неизменённую программу, включая сокеты операционной системы. Тогда между программой и её зависимостью ставят **прокси** и позволяют прокси вести себя плохо.

```text
 client ──► toxiproxy (port 7171) ──► echo server (port 7070)
                 ▲
                 │ HTTP API (port 8474): "add 1 s of latency"
               curl
```

Сначала две программы. Создайте `src/bin/echo_server.rs` — сервер, который отвечает на каждую строку той же строкой:

```rust
//! A tiny TCP server: it answers every line with the same line.

use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7070").await?;
    println!("echo server listening on 127.0.0.1:7070");
    loop {
        let (socket, peer) = listener.accept().await?;
        tokio::spawn(async move {
            let (reader, mut writer) = socket.into_split();
            let mut lines = BufReader::new(reader).lines();
            while let Ok(Some(line)) = lines.next_line().await {
                if writer.write_all(format!("{line}\n").as_bytes()).await.is_err() {
                    break;
                }
            }
            println!("connection from {peer} closed");
        });
    }
}
```

и `src/bin/client.rs`, который отправляет пять пингов, каждый с лимитом 500 мс, и завершается с кодом 1, если хоть один не удался:

```rust
//! Sends five pings through an address and reports what happened to each one.

use std::time::{Duration, Instant};

use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpStream;
use tokio::time::timeout;

const LIMIT: Duration = Duration::from_millis(500);

async fn ping(address: &str, n: u32) -> Result<String, String> {
    let stream = timeout(LIMIT, TcpStream::connect(address))
        .await
        .map_err(|_| "connect timed out".to_string())?
        .map_err(|e| format!("connect failed: {e}"))?;
    let (reader, mut writer) = stream.into_split();
    writer
        .write_all(format!("ping {n}\n").as_bytes())
        .await
        .map_err(|e| format!("write failed: {e}"))?;
    let mut reply = String::new();
    let mut reader = BufReader::new(reader);
    match timeout(LIMIT, reader.read_line(&mut reply)).await {
        Err(_) => Err("no answer within 500 ms".to_string()),
        Ok(Err(e)) => Err(format!("read failed: {e}")),
        Ok(Ok(0)) => Err("connection closed without an answer".to_string()),
        Ok(Ok(_)) => Ok(reply.trim_end().to_string()),
    }
}

#[tokio::main]
async fn main() {
    let address = std::env::args()
        .nth(1)
        .unwrap_or_else(|| "127.0.0.1:7171".to_string());
    let mut failures = 0;
    for n in 1..=5 {
        let started = Instant::now();
        let result = ping(&address, n).await;
        let ms = started.elapsed().as_millis();
        match result {
            Ok(reply) => println!("ping {n}: ok   {ms:>4} ms  ({reply})"),
            Err(reason) => {
                failures += 1;
                println!("ping {n}: FAIL {ms:>4} ms  ({reason})");
            }
        }
    }
    println!("{failures} of 5 failed");
    std::process::exit(if failures == 0 { 0 } else { 1 });
}
```

**Что происходит:**

1. И у подключения, и у чтения есть `timeout` (Урок 14.14). Без них мёртвая зависимость заставила бы клиента ждать вечно: зависание.
2. `read_line` у `BufReader` продолжает читать, пока не увидит `\n`, поэтому короткие чтения здесь обработаны за нас.
3. `Ok(Ok(0))` означает «соединение закончилось раньше, чем пришёл хоть один байт».
4. Адрес по умолчанию — порт прокси 7171; аргумент командной строки может его заменить.

Соберите обе программы и запустите сервер во второй вкладке терминала (он продолжает работать):

```bash
cargo build -q
cargo run -q --bin echo_server
```

```text
echo server listening on 127.0.0.1:7070
```

Теперь запустите toxiproxy из образа, опубликованного его авторами, с портом API и одним портом прокси и создайте через `curl` прокси с именем `echo`:

```bash
docker run -d --name course-toxiproxy -p 8474:8474 -p 7171:7171 ghcr.io/shopify/toxiproxy:2.13.0
curl -s localhost:8474/version
curl -s -X POST localhost:8474/proxies -d '{"name":"echo","listen":"0.0.0.0:7171","upstream":"host.docker.internal:7070"}'
```

```text
{"version": "2.13.0"}
{"name":"echo","listen":"[::]:7171","upstream":"host.docker.internal:7070","enabled":true,"Logger":{},"toxics":[]}
```

(`docker run -d` сначала печатает длинный ID (идентификатор) нового контейнера.)

**Что происходит:**

1. Имя контейнера начинается с префикса `course-`, чтобы его потом можно было найти и удалить. `-p 8474:8474` публикует управляющий API, `-p 7171:7171` — порт, на котором будет слушать прокси.
2. `POST /proxies` создаёт прокси: слушать порт 7171 внутри контейнера и пересылать на `host.docker.internal:7070`. `host.docker.internal` — имя, которое Docker Desktop предоставляет внутри контейнеров; оно означает «Mac, на котором работает Docker», где и находится наш эхо-сервер.

Измерьте **устойчивое состояние** через прокси:

```bash
./target/debug/client
```

```text
ping 1: ok      7 ms  (ping 1)
ping 2: ok      4 ms  (ping 2)
ping 3: ok      4 ms  (ping 3)
ping 4: ok      3 ms  (ping 4)
ping 5: ok      5 ms  (ping 5)
0 of 5 failed
```

**Внедрите задержку.** Токсин добавляется через `POST /proxies/<имя>/toxics`. Этот задерживает все данные от сервера к клиенту (`downstream`) на 1000 мс:

```bash
curl -s -X POST localhost:8474/proxies/echo/toxics -d '{"name":"slow","type":"latency","stream":"downstream","attributes":{"latency":1000,"jitter":0}}'
./target/debug/client
```

```text
{"attributes":{"latency":1000,"jitter":0},"name":"slow","type":"latency","stream":"downstream","toxicity":1}
ping 1: FAIL  502 ms  (no answer within 500 ms)
ping 2: FAIL  501 ms  (no answer within 500 ms)
ping 3: FAIL  502 ms  (no answer within 500 ms)
ping 4: FAIL  501 ms  (no answer within 500 ms)
ping 5: FAIL  502 ms  (no answer within 500 ms)
5 of 5 failed
```

Гипотеза подтверждается: каждый пинг неудачен, и каждый сдаётся примерно через 500 мс, а не ждёт. Медленная зависимость стоит этому клиенту не больше полсекунды на запрос. Уберите токсин через `DELETE`:

```bash
curl -s -X DELETE localhost:8474/proxies/echo/toxics/slow
```

**Внедрите частичный сбой.** Настоящие проблемы часто задевают только *некоторые* запросы. `toxicity` — вероятность того, что новое соединение получит токсин:

```bash
curl -s -X POST localhost:8474/proxies/echo/toxics -d '{"name":"some-slow","type":"latency","stream":"downstream","toxicity":0.4,"attributes":{"latency":800,"jitter":0}}'
./target/debug/client
curl -s -X DELETE localhost:8474/proxies/echo/toxics/some-slow
```

```text
{"attributes":{"latency":800,"jitter":0},"name":"some-slow","type":"latency","stream":"downstream","toxicity":0.4}
ping 1: ok     26 ms  (ping 1)
ping 2: FAIL  502 ms  (no answer within 500 ms)
ping 3: ok      5 ms  (ping 3)
ping 4: ok      7 ms  (ping 4)
ping 5: ok      5 ms  (ping 5)
1 of 5 failed
```

(Какие пинги провалятся — случайно; запустите несколько раз.)

**Уберите зависимость.** `POST /proxies/echo` с `{"enabled":false}` закрывает прокси, как будто сервер лежит:

```bash
curl -s -X POST localhost:8474/proxies/echo -d '{"enabled":false}'
./target/debug/client
curl -s -X POST localhost:8474/proxies/echo -d '{"enabled":true}' > /dev/null
```

```text
{"name":"echo","listen":"[::]:7171","upstream":"host.docker.internal:7070","enabled":false,"Logger":{},"toxics":[]}
ping 1: FAIL    0 ms  (read failed: Connection reset by peer (os error 54))
ping 2: FAIL    0 ms  (read failed: Connection reset by peer (os error 54))
ping 3: FAIL    0 ms  (read failed: Connection reset by peer (os error 54))
ping 4: FAIL    0 ms  (read failed: Connection reset by peer (os error 54))
ping 5: FAIL    0 ms  (read failed: Connection reset by peer (os error 54))
5 of 5 failed
```

Можно было ожидать «connection refused» (в соединении отказано). Но Docker Desktop сам держит порт 7171 открытым на Mac и пересылает соединения в контейнер; когда там больше никто не слушает, он принимает соединение, а затем сбрасывает его. Такие эксперименты учат тому, как отказывает ваше *настоящее* окружение, а это часто отличается от учебника.

Самые полезные токсины:

| Токсин | Что делает | Какой сбой имитирует |
|---|---|---|
| `latency` | Задерживает данные на `latency` мс (± `jitter`) | Медленную сеть или перегруженный сервер |
| `timeout` | Останавливает все данные; закрывает через `timeout` мс (0 = никогда) | Зависший сервер |
| `reset_peer` | Сбрасывает соединение через `timeout` мс | Упавший сервер, разорванное соединение |
| `slicer` | Режет данные на маленькие куски с маленькими задержками | Короткие чтения (раздел 2) на настоящем сокете |
| `bandwidth` | Ограничивает скорость до `rate` КБ/с (килобайт в секунду) | Медленную мобильную сеть |
| `limit_data` | Закрывает соединение после `bytes` байтов | Передачу, оборвавшуюся посередине |

## 4. Хаос-эксперимент в виде скрипта

Набирать команды `curl` хорошо для исследования. Для проверки, которую можно повторять (после каждого изменения или в CI, непрерывной интеграции), запишите эксперимент. Создайте `chaos.sh` в папке пакета:

```bash
#!/usr/bin/env bash
# Небольшой хаос-эксперимент: внедряем сбои через toxiproxy и проверяем реакцию клиента.
set -euo pipefail

API=http://localhost:8474
CLIENT=./target/debug/client

add_toxic() { curl -sf -X POST "$API/proxies/echo/toxics" -d "$1" > /dev/null; }
remove_toxic() { curl -sf -X DELETE "$API/proxies/echo/toxics/$1" > /dev/null; }

# Стоп-кнопка: что бы ни случилось, убрать наши токсины, когда скрипт завершается.
TOXICS=(slow)
cleanup() {
  for toxic in "${TOXICS[@]}"; do
    curl -s -X DELETE "$API/proxies/echo/toxics/$toxic" > /dev/null || true
  done
}
trap cleanup EXIT

# expect <описание> <ожидаемый код возврата> <максимум секунд>
expect() {
  local description=$1 want=$2 max_seconds=$3 got=0
  local started=$SECONDS
  "$CLIENT" > /dev/null || got=$?
  local took=$((SECONDS - started))
  if [[ $got -ne $want ]]; then
    echo "FAIL $description: exit code $got, expected $want"
    return 1
  fi
  if ((took > max_seconds)); then
    echo "FAIL $description: took ${took}s, expected at most ${max_seconds}s"
    return 1
  fi
  echo "PASS $description (exit $got, ${took}s)"
}

expect "steady state: all pings succeed" 0 2
add_toxic '{"name":"slow","type":"latency","stream":"downstream","attributes":{"latency":1000}}'
expect "1 s of latency: every ping fails fast" 1 4
remove_toxic slow
expect "after the fault: back to steady state" 0 2
```

```bash
chmod +x chaos.sh
./chaos.sh
```

```text
PASS steady state: all pings succeed (exit 0, 0s)
PASS 1 s of latency: every ping fails fast (exit 1, 3s)
PASS after the fault: back to steady state (exit 0, 0s)
```

**Что происходит:**

1. `set -euo pipefail` (Урок 11.9) останавливает скрипт на первой неожиданной ошибке. `curl -sf` заставляет `curl` падать на ошибках HTTP, поэтому опечатка в токсине останавливает эксперимент, а не тихо приводит к тому, что ничего не проверяется.
2. `trap cleanup EXIT` — это **стоп-кнопка**: проходит скрипт, падает или прерывается через Ctrl+C, токсины удаляются. Эксперимент никогда не должен оставлять сбой после себя.
3. `expect` запускает клиента и проверяет две вещи: код возврата и время. `SECONDS` — переменная bash, которая считает секунды с начала скрипта.
4. Три вызова — это устойчивое состояние, гипотеза при сбое и восстановление. Последний важен: система, которая не возвращается в норму после того, как сбой убран, тоже содержит баг.

Ловит ли эксперимент настоящую слабость? Замените `LIMIT` в `client.rs` на `Duration::from_secs(10)`, пересоберите через `cargo build -q` и снова запустите скрипт:

```text
PASS steady state: all pings succeed (exit 0, 0s)
FAIL 1 s of latency: every ping fails fast: exit code 0, expected 1
```

С лимитом в 10 секунд клиент ждал ответа целую секунду и считал это успехом. Для пользователя запрос, который длится секунду, может быть так же плох, как отказ, а для сервера это секунда занятого соединения на каждый запрос (перегрузка из Урока 20.8). Скрипт поймал слишком щедрый тайм-аут. Верните `LIMIT` на 500 мс и пересоберите.

## 5. Хаос-инжиниринг на целых системах

Тот же метод масштабируется до целых систем. Команды убивают случайные копии сервиса, разрывают сеть между двумя дата-центрами или заполняют диск и смотрят на дашборды: заметят ли пользователи. Netflix прославил это инструментом Chaos Monkey; в Kubernetes (Этап 21) то же для подов делают инструменты вроде Chaos Mesh. **День игр** — запланированное занятие, на котором команда делает это вместе и отрабатывает реакцию.

Правила, которые делают это безопасным, те же, что и в вашем скрипте:

- **Начинайте с малого и локально.** Сначала в тестах (разделы 2–4), затем в тестовом окружении и лишь намного позже, осторожно, в продакшене.
- **Ограничивайте радиус поражения.** Одна копия, одна зависимость, малая доля трафика (`toxicity`).
- **Всегда имейте стоп-кнопку** и заранее решите, когда её нажать.
- **Сначала пишите гипотезу.** Эксперимент без ожидаемого результата — это просто поломка ради поломки.

> ⚠️ **Частая ошибка:** внедряйте сбои только в системы, которыми вы владеете и которые запускаете сами. toxiproxy в этом уроке стоит на вашем собственном Mac между двумя вашими собственными программами.

Закончив, остановите эхо-сервер через Ctrl+C и удалите контейнер:

```bash
docker rm -f course-toxiproxy
```

## Частые ошибки

**1. Проверять обработку ошибок только «удачными» читателями.** `&[u8]` или файл обычно заполняют весь буфер, поэтому код, игнорирующий короткие чтения, проходит все тесты. Добавьте обёртку с кусками хотя бы в один тест каждого читателя.

**2. Игнорировать число, которое возвращает `read`.** Используйте `read_exact` (или `read_to_end`, `BufReader::read_line`) и не выключайте ошибку Clippy `unused_io_amount`.

**3. Сетевой вызов без тайм-аута.** Без тайм-аута эксперимент с задержкой не упал бы быстро; он висел бы, пока не уберут токсин. Каждому исходящему вызову нужен крайний срок.

**4. Оставлять сбои после себя.** Токсин, оставшийся после эксперимента, заставляет тесты следующего человека загадочно падать. Используйте очистку через `trap … EXIT`, а при сомнениях проверяйте `curl -s localhost:8474/proxies`.

## Попробуйте сами

1. Создайте `code/p20-chaos`, напишите `read_message` с багом и оба теста. Посмотрите, как падает тест с кусками и как Clippy сообщает `unused_io_amount`. Исправьте через `read_exact`.
2. Напишите эхо-сервер и клиента. Запустите клиента прямо против сервера: `./target/debug/client 127.0.0.1:7070`.
3. Запустите toxiproxy, создайте прокси `echo` и повторите четыре эксперимента из раздела 3: устойчивое состояние, задержка, частичная задержка, выключенный прокси.
4. Напишите `chaos.sh` и запустите его. Затем сделайте лимит клиента 10 с и посмотрите, как скрипт упадёт.
5. Удалите контейнер через `docker rm -f course-toxiproxy`.

## Упражнения

### Упражнение 1 — Предскажите, затем измерьте (легко)

Что напечатает клиент при токсине `latency` на 300 мс в направлении downstream? Запишите предсказание, затем запустите.

<details>
<summary>Подсказка</summary>

Сравните 300 мс с лимитом клиента в 500 мс.

</details>

<details>
<summary>Ответ</summary>

```bash
curl -s -X POST localhost:8474/proxies/echo/toxics -d '{"name":"slow","type":"latency","stream":"downstream","attributes":{"latency":300}}'
./target/debug/client
curl -s -X DELETE localhost:8474/proxies/echo/toxics/slow
```

```text
ping 1: ok    304 ms  (ping 1)
ping 2: ok    304 ms  (ping 2)
ping 3: ok    304 ms  (ping 3)
ping 4: ok    310 ms  (ping 4)
ping 5: ok    309 ms  (ping 5)
0 of 5 failed
```

Все пинги успешны, каждый примерно на 300 мс медленнее. 300 мс меньше лимита в 500 мс, поэтому клиент ждёт и получает ответ. (Первая строка вывода, ответ API в формате JSON (JavaScript Object Notation, нотация объектов JavaScript), здесь опущена.)

</details>

### Упражнение 2 — Прерываемый читатель (средне)

Системные вызовы могут вернуть ошибку `Interrupted`, когда приходит сигнал (Урок 13.7); правильная реакция — просто попробовать снова. Напишите обёртку `Interrupting<R>` с полями `inner: R` и `calls: u32`, которая на каждом нечётном вызове возвращает `Err(io::ErrorKind::Interrupted)`, а на каждом чётном читает нормально. Напишите тест, что `read_message` (исправленная через `read_exact`) по-прежнему возвращает `b"hello"`.

<details>
<summary>Подсказка</summary>

`io::Error::from(io::ErrorKind::Interrupted)` создаёт ошибку. Увеличивайте `calls` в начале `read` и проверяйте `self.calls % 2 == 1`.

</details>

<details>
<summary>Ответ</summary>

```rust
/// A test helper that fails with `Interrupted` on every second call.
pub struct Interrupting<R> {
    pub inner: R,
    pub calls: u32,
}

impl<R: Read> Read for Interrupting<R> {
    fn read(&mut self, buf: &mut [u8]) -> io::Result<usize> {
        self.calls += 1;
        if self.calls % 2 == 1 {
            return Err(io::Error::from(io::ErrorKind::Interrupted));
        }
        self.inner.read(buf)
    }
}

#[cfg(test)]
mod fault_tests {
    use super::*;

    const MESSAGE: &[u8] = &[0, 0, 0, 5, b'h', b'e', b'l', b'l', b'o'];

    #[test]
    fn interrupted_reads_are_retried() {
        let mut source = Interrupting {
            inner: MESSAGE,
            calls: 0,
        };
        assert_eq!(read_message(&mut source).unwrap(), b"hello");
    }
}
```

Почему это работает: `read_exact` считает `Interrupted` сигналом «попробуй снова» и продолжает цикл; любая другая ошибка возвращается вызывающему. Ваши собственные циклы чтения должны поступать так же.

</details>

### Упражнение 3 — Данные крошечными кусками (сложно)

Добавьте в `chaos.sh` четвёртый шаг: токсин `slicer`, который отправляет ответ сервера по одному байту (`"average_size":1,"size_variation":0,"delay":10000`, где `delay` — в микросекундах). Сначала предскажите: пройдёт ли клиент? Убедитесь, что стоп-кнопка убирает и этот токсин.

<details>
<summary>Подсказка</summary>

Добавьте имя токсина в массив `TOXICS`. Подумайте, как `read_line` у `BufReader` собирает байты.

</details>

<details>
<summary>Ответ</summary>

Измените массив на `TOXICS=(slow pieces)` и добавьте в конец:

```bash
add_toxic '{"name":"pieces","type":"slicer","stream":"downstream","attributes":{"average_size":1,"size_variation":0,"delay":10000}}'
expect "answers arrive one byte at a time: still correct" 0 2
remove_toxic pieces
```

```text
PASS steady state: all pings succeed (exit 0, 0s)
PASS 1 s of latency: every ping fails fast (exit 1, 3s)
PASS after the fault: back to steady state (exit 0, 0s)
PASS answers arrive one byte at a time: still correct (exit 0, 0s)
```

Почему это работает: `read_line` читает, пока не найдёт `\n`, какими бы маленькими ни были куски, поэтому клиент правильно обрабатывает короткие чтения. Каждый ответ занимает около 70 мс (7 байтов по 10 мс), с большим запасом до лимита. Это тест `Choppy` из раздела 2, выполненный на настоящем сокете.

</details>

## Вопросы для самопроверки

1. Каковы четыре шага хаос-эксперимента?

<details>
<summary>Ответ</summary>

Определить устойчивое состояние, записать гипотезу, внедрить один сбой, наблюдать и сравнить с гипотезой (а затем проверить, что система возвращается в устойчивое состояние).

</details>

2. Почему код, который игнорирует возвращаемое значение `read`, может проходить все тесты, читающие из `&[u8]`?

<details>
<summary>Ответ</summary>

Чтение из памяти обычно заполняет весь буфер за один вызов, поэтому баг не проявляется. На сокетах и каналах `read` часто возвращает меньше байтов: короткое чтение.

</details>

3. Что делает `"toxicity": 0.4`?

<details>
<summary>Ответ</summary>

Каждое новое соединение получает токсин с вероятностью 40 %, поэтому задеты только некоторые запросы, как при многих настоящих сбоях.

</details>

4. Зачем `chaos.sh` использует `trap cleanup EXIT`?

<details>
<summary>Ответ</summary>

Это стоп-кнопка: токсины удаляются при любом завершении скрипта — успешном, неудачном или прерванном. Эксперимент не должен оставлять сбои после себя.

</details>

5. Эксперимент с лимитом клиента в 10 секунд «провалился», хотя все пинги были успешны. Почему это правильный результат?

<details>
<summary>Ответ</summary>

Гипотеза была «при медленной зависимости клиент быстро сдаётся». С 10 с он ждал каждого ответа целую секунду. Это слабость: медленные ответы держат ресурсы занятыми и могут перегрузить сервис. Эксперимент нашёл слишком щедрый тайм-аут.

</details>

## Итоги

- Внедрение сбоев заставляет редкие сбои случиться сейчас, под наблюдением. Метод: устойчивое состояние, гипотеза, внедрение, наблюдение, затем проверка восстановления.
- Внутри кода маленькие обёртки, реализующие `Read` или `Write`, внедряют короткие чтения, прерывания и разорванные соединения. Они нашли классический баг с игнорированием возвращаемого значения `read`; `read_exact` и линт Clippy `unused_io_amount` предотвращают его.
- На сетевой границе toxiproxy добавляет задержки, сбросы, зависания и нарезку к настоящим TCP-соединениям, не меняя вашу программу. `toxicity` задевает только часть соединений.
- Хаос-эксперимент становится повторяемым в виде скрипта с ясными ожиданиями (код возврата и время) и стоп-кнопкой (`trap … EXIT`).
- На целых системах действуют те же правила: начинать с малого, ограничивать радиус поражения, держать стоп-кнопку и проверять только системы, которыми вы владеете.

## Источники и что почитать

- [Toxiproxy](https://github.com/Shopify/toxiproxy) — README перечисляет все токсины, их атрибуты и весь HTTP API.
- [std::io::Read](https://doc.rust-lang.org/std/io/trait.Read.html) — точные обещания `read`, `read_exact` и то, как обрабатывается `Interrupted`.
- [Clippy — unused_io_amount](https://rust-lang.github.io/rust-clippy/master/index.html) — найдите `unused_io_amount`, линт, который ловит проигнорированные объёмы чтения и записи.
- [Principles of Chaos Engineering](https://principlesofchaos.org/) — короткое исходное описание метода: устойчивое состояние, гипотезы, малый радиус поражения.
- [Docker Desktop networking](https://docs.docker.com/desktop/features/networking/) — как работают `host.docker.internal` и опубликованные порты на Mac.

---

[🏠 Начало](../START-HERE.md) · [← 20.4 Miri, санитайзеры и loom в CI](04-miri-sanitizers-and-loom-in-ci.md) · [📋 Этап 20](README.md) · [20.6 Основы формальной верификации с Kani →](06-formal-verification-with-kani.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
