<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 15.8 HTTP с нуля](08-http-from-scratch.md) · [📋 Этап 15](README.md) · [15.10 DNS: как имена превращаются в адреса →](10-dns-how-names-become-addresses.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/15-networking/09-http-with-hyper-and-reqwest.md) <!-- nav -->

# Урок 15.9 — HTTP с hyper и reqwest

> **Этап 15 — Сети** · Урок 9 из 16 · ⏱ около 135 минут

## Что вы узнаете

- Почему настоящие программы используют HTTP-библиотеку, а не самописный парсер, и что делают `hyper` и `reqwest`.
- Общие типы `http`: `Request`, `Response`, `Method`, `StatusCode`, заголовки и тела.
- Как написать HTTP/1.1-сервер на `hyper`: функция-сервис, построитель соединения и адаптер `TokioIo`.
- Как безопасно прочитать тело запроса с ограничением размера.
- Как обращаться к HTTP-серверам через `reqwest` с тайм-аутами и проверкой статуса и как тестировать сервер сквозным тестом.

## Перед началом

- Вы прошли [Урок 15.8 — HTTP с нуля](08-http-from-scratch.md): вы знаете строку запроса, заголовки, пустую строку, `Content-Length` и keep-alive.
- Вы знаете цикл приёма соединений Tokio из [Урока 15.6 — Асинхронная работа с сетью в Tokio](06-async-networking-with-tokio.md) и задачи из [Урока 14.13 — Tokio подробно: задачи и каналы](../14-concurrency-and-async/13-tokio-tasks-and-channels.md).
- Создайте учебный пакет. Этому уроку нужны библиотека и интеграционные тесты, поэтому у него свой пакет рядом с `code/p15-net`: несколько маленьких программ в `src/bin/`, библиотека в `src/lib.rs` и тесты в `tests/`:

```bash
cd ~/projects/rust
cargo new code/p15-http
cd code/p15-http
mkdir -p src/bin tests
rm src/main.rs
cargo add tokio@1 --features full
cargo add hyper@1 --features server,http1
cargo add hyper-util@0.1 --features tokio
cargo add http-body-util@0.1 bytes@1
cargo add reqwest@0.13
```

Первая сборка компилирует около 150 крейтов (в основном `reqwest` и его код TLS; TLS, Transport Layer Security, протокол защиты транспортного уровня, — слой шифрования из Урока 10.11) и занимает несколько минут. Следующие сборки быстрые.

## Новые слова

| Слово | Значение |
|---|---|
| **HTTP** (HyperText Transfer Protocol, протокол передачи гипертекста) | Протокол «запрос — ответ», на котором работает веб. Небольшую его версию вы написали в Уроке 15.8. |
| **`hyper`** | Быстрая низкоуровневая HTTP-библиотека. Разбирает и записывает сообщения HTTP/1.1 и HTTP/2 и управляет соединениями. Маршрутизацию URL она за вас не делает. |
| **`reqwest`** | Высокоуровневый HTTP-*клиент* поверх `hyper`. Добавляет пул соединений, перенаправления, тайм-ауты и HTTPS. |
| **HTTPS** (HTTP Secure, защищённый HTTP) | HTTP внутри соединения, зашифрованного TLS (Урок 10.11). |
| **крейт `http`** (`http` crate) | Небольшой крейт с общими типами `Request`, `Response`, `Method`, `StatusCode`, `HeaderMap`, `Uri`. Его используют `hyper`, `reqwest` и `axum`. |
| **тело** (body) | Байты после заголовков. В `hyper` тело — это *поток* частей (фреймов), потому что оно может быть большим или приходить медленно. |
| **фрейм** (frame, часть тела) | Одна часть тела: кусок данных или трейлеры (заголовки, отправленные после тела). |
| **сервис** (service) | Функция (или объект), которая превращает запрос в future ответа. `hyper` вызывает её один раз на каждый запрос. |
| **I/O** (input/output, ввод-вывод) | Чтение и запись данных: здесь — байтов в сокет и из сокета. |
| **адаптер** (adapter) | Маленький тип-обёртка, который подгоняет тип одной библиотеки под трейт другой. `TokioIo` — такой адаптер. |
| **пул соединений** (connection pool) | Набор открытых соединений, которые клиент держит и переиспользует, чтобы не открывать новое TCP-соединение (Transmission Control Protocol, протокол управления передачей) для каждого запроса. |
| **перенаправление** (redirect) | Ответ со статусом вроде `301` или `308` и заголовком `Location`, который говорит: «спросите вместо этого другой URL». |
| **URL** (Uniform Resource Locator, единый указатель ресурса) | Веб-адрес, например `http://127.0.0.1:3000/health`: схема, хост, порт, путь. |
| **ALPN** (Application-Layer Protocol Negotiation, согласование протокола прикладного уровня) | Поле в рукопожатии TLS (Transport Layer Security, протокол защиты транспортного уровня), в котором клиент и сервер договариваются говорить, например, на HTTP/2. Урок 15.12 показывает это в коде. |
| **сквозной тест** (end-to-end test) | Тест, который запускает настоящий сервер и обращается к нему через настоящий сокет, как это сделал бы пользователь. |
| **API, URI, ГБ** (Application Programming Interface, Uniform Resource Identifier, GB — программный интерфейс, единообразный идентификатор ресурса, гигабайт) | Типы и функции, которые библиотека предлагает вашему коду; общее название для URL или пути вроде `/health`; около миллиарда байтов. |

## 1. Зачем нужна библиотека

В Уроке 15.8 вы сами разбирали `GET /path HTTP/1.1`. Это лучший способ *понять* HTTP, но сервер для настоящих пользователей должен правильно обрабатывать ещё много деталей:

- тела, отправленные с `Transfer-Encoding: chunked`, и тела, которые приходят медленно, по кусочкам;
- keep-alive и конвейерную обработку (pipelining: несколько запросов в одном соединении, ответы в том же порядке);
- странный или враждебный ввод: огромные заголовки, недопустимые байты, одновременно `Content-Length` и `Transfer-Encoding`;
- HTTP/2 — двоичный протокол со множеством потоков в одном соединении;
- правильные заголовки `Date` и `Content-Length` в каждом ответе.

`hyper` делает всё это, и им пользуются тысячи сервисов. Его работа — только **протокол**: он читает байты из соединения, отдаёт вам `Request`, берёт ваш `Response` и записывает байты. Маршрутизация, JSON (JavaScript Object Notation, текстовый формат данных) и middleware (код, который выполняется вокруг каждого запроса, например логирование) — задача фреймворков поверх него, таких как `axum` (Урок 16.3).

```text
 your code:   async fn route(Request) -> Response        (you write this)
                 ▲                      │
 hyper:       parse bytes ──────────────┘ write bytes     (protocol, keep-alive, limits)
                 ▲                      ▼
 tokio:       TcpStream (read)          TcpStream (write) (sockets, from Lesson 15.6)
```

С клиентом то же самое. У `hyper` есть и низкоуровневый клиент, но `reqwest` оборачивает его в удобный API, и большинство программ используют именно его.

## 2. Типы `http`

Все HTTP-библиотеки в Rust используют общие типы крейта `http` (`hyper` реэкспортирует их), поэтому выучить их нужно один раз:

| Тип | Что хранит | Пример |
|---|---|---|
| `Method` | глагол (метод) | `Method::GET`, `Method::POST` |
| `Uri` | цель запроса | `/upper?x=1`; `.path()` даёт `/upper` |
| `StatusCode` | результат | `StatusCode::OK` (200), `StatusCode::NOT_FOUND` (404) |
| `HeaderMap` | все заголовки; имена не зависят от регистра | `headers()["content-type"]` |
| `Request<B>` | метод, URI, версию, заголовки и тело типа `B` | `Request<Incoming>` |
| `Response<B>` | статус, версию, заголовки и тело типа `B` | `Response<Full<Bytes>>` |

Тип тела — это **обобщённый параметр** `B`. Для запроса, пришедшего из сети, `hyper` даёт `hyper::body::Incoming` — поток, который читают фрейм за фреймом. Для ответа, байты которого уже лежат в памяти, используют `http_body_util::Full<Bytes>` — тело ровно из одного фрейма данных. `Bytes` — дёшево клонируемый байтовый буфер из крейта `bytes` (Урок 15.7).

## 3. Первый сервер на hyper

Создайте `src/bin/hello.rs`:

```rust
use std::convert::Infallible;
use std::net::SocketAddr;

use bytes::Bytes;
use http_body_util::Full;
use hyper::body::Incoming;
use hyper::server::conn::http1;
use hyper::service::service_fn;
use hyper::{Request, Response};
use hyper_util::rt::TokioIo;
use tokio::net::TcpListener;

async fn hello(req: Request<Incoming>) -> Result<Response<Full<Bytes>>, Infallible> {
    let text = format!("Hello! You asked for {}\n", req.uri().path());
    Ok(Response::new(Full::new(Bytes::from(text))))
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let addr = SocketAddr::from(([127, 0, 0, 1], 3000));
    let listener = TcpListener::bind(addr).await?;
    println!("listening on http://{addr}");

    loop {
        let (stream, peer) = listener.accept().await?;
        let io = TokioIo::new(stream);
        tokio::spawn(async move {
            let service = service_fn(hello);
            if let Err(err) = http1::Builder::new().serve_connection(io, service).await {
                eprintln!("connection from {peer} failed: {err}");
            }
        });
    }
}
```

**Что происходит:**

1. `hello` — это **сервис**. Он получает один разобранный запрос и возвращает ответ. Его тип ошибки — `Infallible`, тип без значений: он говорит «эта функция никогда не завершается ошибкой». `Response::new` создаёт ответ `200 OK` с данным телом.
2. `main` — цикл приёма соединений из Урока 15.6: bind, accept, по одной задаче на соединение.
3. `TokioIo::new(stream)` — это **адаптер**. `hyper` не зависит от Tokio; у него свои трейты ввода-вывода `hyper::rt::Read` и `hyper::rt::Write`. `TokioIo` из `hyper-util` реализует их для любого потока Tokio.
4. `service_fn(hello)` превращает обычную async-функцию в то, что реализует трейт `Service` из `hyper`.
5. `http1::Builder::new().serve_connection(io, service)` возвращает future, который обслуживает **все** запросы этого соединения (keep-alive) и завершается, когда клиент его закрывает. Ошибки (например, мусорные байты) печатаются в лог, и завершается только это одно соединение.

Запустите его в одном терминале:

```bash
cargo run --bin hello
```

```text
listening on http://127.0.0.1:3000
```

Во втором терминале отправьте запрос. `-i` печатает строку статуса и заголовки:

```bash
curl -i http://127.0.0.1:3000/rust
```

```text
HTTP/1.1 200 OK
content-length: 27
date: Wed, 18 Nov 2026 07:05:07 GMT

Hello! You asked for /rust
```

`content-length` и `date` вы не писали: их добавил `hyper`. Теперь запросите два URL одной командой `curl` и посмотрите на строки о соединении в подробном выводе (`-v`):

```bash
curl -sv http://127.0.0.1:3000/a http://127.0.0.1:3000/b 2>&1 | grep -E '^\* (Connected|Re-using)|^> GET|^Hello'
```

```text
* Connected to 127.0.0.1 (127.0.0.1) port 3000
> GET /a HTTP/1.1
Hello! You asked for /a
* Re-using existing connection with host 127.0.0.1
> GET /b HTTP/1.1
Hello! You asked for /b
```

Оба запроса прошли по **одному** TCP-соединению, поэтому `serve_connection` вызвал `hello` дважды. Keep-alive работает без единой строки вашего кода. Остановите сервер через `Ctrl+C`.

## 4. Маршрутизация и тело с ограничением размера

Настоящий сервер смотрит на метод и путь и решает, что делать. Мы положим эту логику в библиотеку (`src/lib.rs`), чтобы ею могли пользоваться и программа, *и* тесты:

```rust
use std::convert::Infallible;

use bytes::Bytes;
use http_body_util::{BodyExt, Full, LengthLimitError, Limited};
use hyper::body::Incoming;
use hyper::server::conn::http1;
use hyper::service::service_fn;
use hyper::{Method, Request, Response, StatusCode};
use hyper_util::rt::TokioIo;
use tokio::net::TcpListener;

/// The largest request body we accept, in bytes.
pub const MAX_BODY: usize = 1024;

/// Builds a plain-text response with the given status.
fn text(status: StatusCode, body: impl Into<Bytes>) -> Response<Full<Bytes>> {
    Response::builder()
        .status(status)
        .header("content-type", "text/plain; charset=utf-8")
        .body(Full::new(body.into()))
        .expect("status and header are valid")
}

/// Chooses what to do from the method and the path of the request.
pub async fn route(req: Request<Incoming>) -> Result<Response<Full<Bytes>>, Infallible> {
    let response = match (req.method(), req.uri().path()) {
        (&Method::GET, "/health") => text(StatusCode::OK, "ok\n"),
        (&Method::POST, "/upper") => {
            let limited = Limited::new(req.into_body(), MAX_BODY);
            match limited.collect().await {
                Ok(collected) => {
                    let body = collected.to_bytes();
                    text(StatusCode::OK, body.to_ascii_uppercase())
                }
                Err(err) if err.is::<LengthLimitError>() => {
                    text(StatusCode::PAYLOAD_TOO_LARGE, "body is too large\n")
                }
                Err(_) => text(StatusCode::BAD_REQUEST, "could not read the body\n"),
            }
        }
        (_, "/health" | "/upper") => text(StatusCode::METHOD_NOT_ALLOWED, "wrong method\n"),
        _ => text(StatusCode::NOT_FOUND, "not found\n"),
    };
    Ok(response)
}

/// Accepts connections forever and serves each one in its own task.
pub async fn serve(listener: TcpListener) -> std::io::Result<()> {
    loop {
        let (stream, peer) = listener.accept().await?;
        tokio::spawn(async move {
            let io = TokioIo::new(stream);
            if let Err(err) = http1::Builder::new()
                .serve_connection(io, service_fn(route))
                .await
            {
                eprintln!("connection from {peer} failed: {err}");
            }
        });
    }
}
```

**Что происходит:**

1. `text` использует `Response::builder()`, который позволяет задавать статус и заголовки шаг за шагом. `.body(…)` возвращает `Result`, потому что в построитель мог попасть недопустимый заголовок; наши фиксированы и корректны, поэтому `expect` здесь уместен.
2. `route` сопоставляет пару `(method, path)`. `match` по кортежу — простой маршрутизатор: каждая ветка — один эндпоинт (endpoint).
3. Для `POST /upper` вызов `req.into_body()` забирает тело из запроса. `Limited::new(body, MAX_BODY)` оборачивает его: как только придёт больше 1024 байт, чтение завершится ошибкой `LengthLimitError`.
4. `.collect()` (из трейта `BodyExt`) читает все фреймы и склеивает их; `.to_bytes()` даёт один `Bytes`. Мы возвращаем текст в верхнем регистре.
5. `err.is::<LengthLimitError>()` проверяет, *какая* ошибка случилась: слишком большое тело → `413 Payload Too Large`; всё остальное (клиент пропал посередине) → `400 Bad Request`.
6. Известный путь с неправильным методом получает `405 Method Not Allowed`; любой другой путь — `404 Not Found`.
7. `serve` — цикл приёма из раздела 3, теперь внутри библиотеки и с `route` в роли сервиса.

Создайте маленькую программу `src/bin/server.rs`, которая использует библиотеку (пакет называется `p15-http`, поэтому в коде Rust библиотека называется `p15_http`):

```rust
use tokio::net::TcpListener;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:3000").await?;
    println!("listening on http://{}", listener.local_addr()?);
    p15_http::serve(listener).await
}
```

Запустите её через `cargo run --bin server` и попробуйте каждый маршрут из второго терминала. Последняя команда отправляет 2000 байт буквы `a`:

```bash
curl -s http://127.0.0.1:3000/health
curl -s -d 'make me loud' http://127.0.0.1:3000/upper; echo
curl -si -X DELETE http://127.0.0.1:3000/health | head -1
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3000/missing
head -c 2000 /dev/zero | tr '\0' a | curl -s -w '%{http_code}\n' --data-binary @- http://127.0.0.1:3000/upper
```

```text
ok
MAKE ME LOUD
HTTP/1.1 405 Method Not Allowed
404
body is too large
413
```

А теперь отправьте через `nc` (netcat, из Урока 11.11) байты, которые вообще не являются HTTP:

```bash
printf 'GARBAGE\r\n\r\n' | nc 127.0.0.1 3000
```

```text
HTTP/1.1 400 Bad Request
connection: close
content-length: 0
date: Wed, 18 Nov 2026 07:05:18 GMT

```

В терминале сервера появится `connection from 127.0.0.1:51829 failed: invalid HTTP method parsed` (номер порта у вас будет другим). `hyper` ответил `400`, закрыл только это соединение, а сервер продолжает работать.

> 🔒 **Безопасность:** никогда не читайте тело без ограничения. Без `Limited` вызов `collect()` держит всё в памяти, и один клиент, отправивший тело в 10 ГБ, может исчерпать память вашего сервера. Выберите ограничение, подходящее эндпоинту (несколько KiB, кибибайт по 1024 байта, для формы; больше для загрузки файлов), и отвечайте `413`, когда оно превышено.

## 5. Клиент на reqwest

Оставьте сервер работать. Создайте `src/bin/client.rs`:

```rust
use std::time::Duration;

#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    let client = reqwest::Client::builder()
        .timeout(Duration::from_secs(5))
        .user_agent("p15-http-client/0.1")
        .build()?;

    let resp = client.get("http://127.0.0.1:3000/health").send().await?;
    println!("GET  /health -> {} {:?}", resp.status(), resp.version());
    println!("     content-type: {:?}", resp.headers()["content-type"]);
    println!("     body: {:?}", resp.text().await?);

    let resp = client
        .post("http://127.0.0.1:3000/upper")
        .body("hello, hyper")
        .send()
        .await?;
    println!(
        "POST /upper  -> {} body: {:?}",
        resp.status(),
        resp.text().await?
    );

    let resp = client.get("http://127.0.0.1:3000/nope").send().await?;
    println!("GET  /nope   -> {}", resp.status());

    let resp = client.get("https://www.rust-lang.org/").send().await?;
    println!(
        "GET  https://www.rust-lang.org/ -> {} {:?}",
        resp.status(),
        resp.version()
    );
    Ok(())
}
```

**Что происходит:**

1. `Client::builder()` один раз настраивает клиента: **тайм-аут** на весь запрос (подключение, отправка, ожидание, чтение) и заголовок `User-Agent`, который сообщает серверам, кто обращается. `build()` создаёт клиента и его пул соединений.
2. `client.get(url).send().await?` отправляет запрос и ждёт строку статуса и заголовки. Тело ещё не прочитано.
3. `resp.status()`, `resp.version()` и `resp.headers()` смотрят на начало ответа. `resp.text().await?` читает всё тело как `String`. Он забирает `resp` по значению, поэтому идёт последним.
4. `.post(url).body("…")` отправляет тело; `Content-Length` `reqwest` выставит сам.
5. `404` — **не** ошибка: `?` пропускает его, потому что сам HTTP-обмен прошёл успешно. Раздел «Частые ошибки» показывает, как превратить плохие статусы в ошибки.
6. Последний запрос идёт по HTTPS. `reqwest` приносит свой TLS (он использует `rustls`, библиотеку из Урока 10.11), проверяет сертификат, следует перенаправлению с `www.rust-lang.org` на `rust-lang.org` и договаривается об HTTP/2 через ALPN.

```bash
cargo run --bin client
```

```text
GET  /health -> 200 OK HTTP/1.1
     content-type: "text/plain; charset=utf-8"
     body: "ok\n"
POST /upper  -> 200 OK body: "HELLO, HYPER"
GET  /nope   -> 404 Not Found
GET  https://www.rust-lang.org/ -> 200 OK HTTP/2.0
```

Наш сервер говорит только на HTTP/1.1 (мы включили только фичу `http1`), а публичный сайт ответил по HTTP/2.0. Код клиента при этом не менялся: `reqwest` выбирает версию для каждого соединения.

> ⚡ **Производительность:** создайте **один** `reqwest::Client` и переиспользуйте его (клонировать его дёшево, и клоны делят один пул). Пул держит соединения открытыми, поэтому второй запрос к тому же хосту пропускает рукопожатие TCP и рукопожатие TLS. Новый `Client` на каждый запрос теряет всё это.

## 6. Сквозное тестирование сервера

Поскольку `serve` принимает `TcpListener`, тест может привязаться к порту `0`. Тогда операционная система выберет любой свободный порт, и тесты никогда не столкнутся ни с работающим сервером, ни друг с другом. Создайте `tests/api.rs`:

```rust
use tokio::net::TcpListener;

/// Starts the server on a free port and returns its base URL.
async fn start_server() -> String {
    let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
    let addr = listener.local_addr().unwrap();
    tokio::spawn(p15_http::serve(listener));
    format!("http://{addr}")
}

#[tokio::test]
async fn health_says_ok() {
    let base = start_server().await;
    let resp = reqwest::get(format!("{base}/health")).await.unwrap();
    assert_eq!(resp.status(), 200);
    assert_eq!(resp.text().await.unwrap(), "ok\n");
}

#[tokio::test]
async fn upper_rejects_a_big_body() {
    let base = start_server().await;
    let big = "a".repeat(p15_http::MAX_BODY + 1);
    let resp = reqwest::Client::new()
        .post(format!("{base}/upper"))
        .body(big)
        .send()
        .await
        .unwrap();
    assert_eq!(resp.status(), 413);
}
```

**Что происходит:**

1. `#[tokio::test]` похож на `#[test]`, но запускает для теста рантайм Tokio, поэтому тест может быть `async`.
2. `start_server` привязывается к `127.0.0.1:0`, узнаёт настоящий адрес через `local_addr()` и запускает `serve` в фоновой задаче. Сервер останавливается вместе с рантаймом теста.
3. `reqwest::get(url)` — сокращение для «новый клиент, GET, отправить». В тестах это нормально.
4. Второй тест отправляет на один байт больше ограничения и ждёт `413`. Тест на ограничение защищает от будущего изменения, которое случайно его уберёт.

```bash
cargo test --test api
```

```text
running 2 tests
test upper_rejects_a_big_body ... ok
test health_says_ok ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

> 🧪 **Тестирование:** можно вызывать `route` напрямую в модульном тесте, но тогда нужен `Request<Incoming>`, а его может создать только `hyper` из настоящего соединения. Тестировать через настоящий сокет на порту `0` проще, и так проверяется больше: разбор, keep-alive и ваш код вместе.

## Частые ошибки

**1. Фича не включена.** По умолчанию `hyper` почти ничего не компилирует. С `hyper = "1"` без фич:

```text
error[E0433]: cannot find `server` in `hyper`
 --> src/lib.rs:6:12
  |
6 | use hyper::server::conn::http1;
  |            ^^^^^^ could not find `server` in `hyper`
  |
note: found an item that was configured out
  ...
157 |     #![feature = "server"]
    |        ------------------ the item is gated behind the `server` feature
```

Примечание (note) прямо называет недостающую фичу. Исправление: `cargo add hyper@1 --features server,http1`.

**2. Поток Tokio передан в hyper напрямую.** Если написать `serve_connection(stream, service)` без `TokioIo`:

```text
error[E0277]: the trait bound `tokio::net::TcpStream: hyper::rt::Read` is not satisfied
   --> src/bin/hello.rs:29:70
    |
 29 |             if let Err(err) = http1::Builder::new().serve_connection(io, service).await {
    |                                                     ---------------- ^^ the trait `hyper::rt::Read` is not implemented for `tokio::net::TcpStream`
    |                                                     |
    |                                                     required by a bound introduced by this call
    |
    = help: the following other types implement trait `hyper::rt::Read`:
              &mut T
              Box<T>
              Pin<P>
              TokioIo<T>
              ...
```

В списке help даже назван `TokioIo<T>`. Исправление: оберните поток — `TokioIo::new(stream)`.

**3. Считать, что `404` — это `Err`.** `send().await?` завершается ошибкой, только если не удался сам обмен (нет соединения, тайм-аут, некорректный ответ). `404` или `500` — это *успешный* обмен с плохим статусом, и ваш код продолжает работать, как будто всё хорошо. Если плохой статус должен быть ошибкой, скажите это явно через `error_for_status()` (Упражнение 2).

**4. Нет тайм-аута.** `reqwest::Client` без `.timeout(…)` ждёт вечно сервер, который принял соединение, но так и не отвечает. Одна зависшая зависимость тогда блокирует всю вашу программу. Всегда задавайте тайм-аут, соответствующий тому, чего вы ждёте от сервера.

## Попробуйте сами

1. Запустите `cargo run --bin server`.
2. Откройте «сырое» соединение через `nc 127.0.0.1 3000` и наберите (Enter после каждой строки и дважды Enter в конце):

   ```text
   GET /health HTTP/1.1
   Host: x

   ```

3. Вы увидите `HTTP/1.1 200 OK`, заголовки `content-type`, `content-length` и `date`, пустую строку и `ok`. Соединение остаётся открытым: наберите тот же запрос снова — и получите второй ответ. Это keep-alive, который делает `hyper`. (Клавиша Enter отправляет только `\n`, а не `\r\n`; `hyper` принимает оба варианта.)
4. Наберите `GET /health HTTP/1.1` **без** строки `Host` и дважды нажмите Enter. Стандарт HTTP/1.1 требует `Host`, но `hyper` здесь снисходителен и всё равно отвечает `200`; строгий прокси мог бы ответить `400`. Нажмите `Ctrl+C`, чтобы закрыть `nc`.

## Упражнения

### Упражнение 1 — Эндпоинт версии (легко)

Добавьте `GET /version`, который отвечает `p15-http 0.1.0` и перевод строки, причём версия берётся из `Cargo.toml` (а не набирается вручную). Напишите для него тест в `tests/api.rs`.

<details>
<summary>Подсказка</summary>

Макрос `env!("CARGO_PKG_VERSION")` даёт версию во время компиляции, а `concat!` склеивает строковые литералы во время компиляции.

</details>

<details>
<summary>Ответ</summary>

Добавьте эту ветку в `match` внутри `route`:

```rust
        (&Method::GET, "/version") => text(
            StatusCode::OK,
            concat!("p15-http ", env!("CARGO_PKG_VERSION"), "\n"),
        ),
```

И тест:

```rust
#[tokio::test]
async fn version_route_reports_the_package_version() {
    let base = start_server().await;
    let resp = reqwest::get(format!("{base}/version")).await.unwrap();
    assert_eq!(resp.status(), 200);
    assert_eq!(resp.text().await.unwrap(), "p15-http 0.1.0\n");
}
```

Почему это работает: Cargo задаёт `CARGO_PKG_VERSION`, пока компилирует ваш крейт, поэтому текст всегда совпадает с `Cargo.toml`. `concat!` даёт `&'static str`, и `text` его принимает, потому что `&'static str` преобразуется в `Bytes`.

</details>

### Упражнение 2 — Плохие статусы — это ошибки (средне)

Напишите `src/bin/strict.rs`: для путей `/health` и `/nope` отправьте GET работающему серверу и напечатайте либо тело, либо ошибку. Статус 400 и выше должен становиться `Err`.

<details>
<summary>Подсказка</summary>

`Response::error_for_status(self)` возвращает `Ok(response)` для статусов ниже 400 и `Err(reqwest::Error)` для остальных.

</details>

<details>
<summary>Ответ</summary>

```rust
#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    let client = reqwest::Client::new();
    for path in ["/health", "/nope"] {
        let url = format!("http://127.0.0.1:3000{path}");
        match client.get(&url).send().await?.error_for_status() {
            Ok(resp) => println!("{path}: {}", resp.text().await?.trim_end()),
            Err(err) => println!("{path}: error: {err}"),
        }
    }
    Ok(())
}
```

При работающем сервере `cargo run --bin strict` печатает:

```text
/health: ok
/nope: error: HTTP status client error (404 Not Found) for url (http://127.0.0.1:3000/nope)
```

Почему это работает: `send().await?` обрабатывает ошибки транспорта; затем `error_for_status()` превращает 404 в `reqwest::Error`, в сообщении которого есть статус и URL. Мы используем `match`, а не `?`, чтобы цикл продолжился и напечатал обе строки.

</details>

### Упражнение 3 — Клиент, который сдаётся (сложно)

Добавьте `GET /slow`, который ждёт две секунды перед ответом. Напишите тест, который вызывает его клиентом с тайм-аутом 300 миллисекунд и проверяет, что ошибка — это тайм-аут.

<details>
<summary>Подсказка</summary>

`tokio::time::sleep(Duration::from_secs(2)).await` внутри ветки. В тесте `send().await.unwrap_err()` даёт ошибку, а у `reqwest::Error` есть метод `is_timeout()`.

</details>

<details>
<summary>Ответ</summary>

Добавьте `use std::time::Duration;` в `src/lib.rs` и ветку:

```rust
        (&Method::GET, "/slow") => {
            tokio::time::sleep(Duration::from_secs(2)).await;
            text(StatusCode::OK, "finally\n")
        }
```

Тест:

```rust
#[tokio::test]
async fn client_gives_up_on_a_slow_route() {
    let base = start_server().await;
    let client = reqwest::Client::builder()
        .timeout(std::time::Duration::from_millis(300))
        .build()
        .unwrap();
    let err = client.get(format!("{base}/slow")).send().await.unwrap_err();
    assert!(err.is_timeout(), "expected a timeout, got {err}");
}
```

Все четыре теста (два из раздела 6 и два из упражнений) проходят:

```text
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.32s
```

Почему это работает: таймер клиента срабатывает через 300 мс и отменяет future запроса (отмена, Урок 14.14). Тест завершается примерно за 0,3 секунды, а не за 2, что заодно показывает: медленная задача сервера не блокирует тест.

</details>

## Вопросы для самопроверки

1. В чём работа `hyper` и что в его работу *не* входит?

<details>
<summary>Ответ</summary>

`hyper` реализует протокол HTTP: разбирает запросы, записывает ответы, обрабатывает keep-alive, тела chunked и HTTP/2. Маршрутизацией, JSON, аутентификацией и middleware он не занимается; это добавляют фреймворки поверх него, например `axum`.

</details>

2. Почему `serve_connection` нужен `TokioIo::new(stream)`, а не просто `stream`?

<details>
<summary>Ответ</summary>

`hyper` не зависит от конкретного асинхронного рантайма и определяет собственные трейты ввода-вывода (`hyper::rt::Read`/`Write`). `TcpStream` из Tokio реализует трейты Tokio, а не hyper. `TokioIo` — адаптер из `hyper-util`, который их соединяет.

</details>

3. От чего защищает `Limited::new(body, 1024)` и каким статусом сервер должен ответить, когда ограничение превышено?

<details>
<summary>Ответ</summary>

От клиента, который отправляет огромное тело и заполняет память сервера. Когда приходит больше 1024 байт, чтение завершается ошибкой `LengthLimitError`, и сервер должен ответить `413 Payload Too Large`.

</details>

4. `client.get(url).send().await?` вернул ответ со статусом 500. Вернул ли `?` ошибку? Как сделать, чтобы вернул?

<details>
<summary>Ответ</summary>

Нет. Обмен удался, поэтому `send` вернул `Ok(response)` со статусом 500. Вызовите `.error_for_status()` у ответа, чтобы статусы 4xx и 5xx превращались в `Err`.

</details>

5. Почему программе стоит переиспользовать один `reqwest::Client`?

<details>
<summary>Ответ</summary>

Клиент владеет пулом соединений. Если его переиспользовать, следующие запросы к тому же хосту идут по уже открытым соединениям и пропускают рукопожатия TCP и TLS. Новый клиент на каждый запрос каждый раз открывает новое соединение.

</details>

## Итоги

- `hyper` — движок протокола HTTP; `reqwest` — удобный клиент поверх него; оба используют типы крейта `http`.
- Сервер на `hyper` — это цикл приёма Tokio плюс `http1::Builder::new().serve_connection(TokioIo::new(stream), service_fn(f))` в отдельной задаче на каждое соединение.
- Сервис превращает `Request<Incoming>` в `Response<Full<Bytes>>`; `match` по `(method, path)` — простой маршрутизатор.
- Тела запросов — это потоки; читайте их через `Limited` и `collect()` и отвечайте `413`, когда они слишком велики.
- `reqwest::Client` даёт пул соединений, перенаправления, HTTPS и тайм-ауты; всегда задавайте тайм-аут и решайте, как обращаться с плохими статусами (`error_for_status`).
- Сквозные тесты привязывают сервер к порту `0` и обращаются к нему через `reqwest`.

## Источники и что почитать

- [hyper guide: Getting Started with a Server](https://hyper.rs/guides/1/server/hello-world/) — официальное пошаговое руководство по серверу на hyper 1.
- [Документация hyper](https://docs.rs/hyper/latest/hyper/) — справочник API, включая список фич Cargo.
- [Документация hyper-util](https://docs.rs/hyper-util/latest/hyper_util/) — `TokioIo` и построитель `server::conn::auto`, который обслуживает HTTP/1 и HTTP/2 на одном порту.
- [Документация http-body-util](https://docs.rs/http-body-util/latest/http_body_util/) — `Full`, `Limited`, `BodyExt::collect` и другие помощники для тел.
- [Документация reqwest](https://docs.rs/reqwest/latest/reqwest/) — API клиента, тайм-ауты, перенаправления и дополнительные фичи, например `json`.
- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110) — официальное значение методов, кодов статуса и заголовков.

---

[🏠 Начало](../START-HERE.md) · [← 15.8 HTTP с нуля](08-http-from-scratch.md) · [📋 Этап 15](README.md) · [15.10 DNS: как имена превращаются в адреса →](10-dns-how-names-become-addresses.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
