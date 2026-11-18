<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 15.7 Проектирование протокола: фрейминг и разбор](07-designing-a-protocol.md) · [📋 Этап 15](README.md) · [15.9 HTTP с hyper и reqwest →](09-http-with-hyper-and-reqwest.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/15-networking/08-http-from-scratch.md) <!-- nav -->

# Урок 15.8 — HTTP с нуля

> **Этап 15 — Сети** · Урок 8 из 16 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Точный формат запросов и ответов HTTP/1.1: стартовая строка, заголовки, пустая строка и тело.
- Как сервер узнаёт, где кончается тело (`Content-Length` и кодирование по частям, chunked), и почему ошибка здесь — дыра в безопасности.
- Как работают постоянные соединения (keep-alive) и конвейерная отправка запросов (pipelining) и как `Connection: close` их завершает.
- Как написать строгий парсер HTTP/1.1 с ограничениями в виде чистой функции и сервер на Tokio вокруг него.
- Как тестировать парсер без сети и проверять сервер с помощью `curl -v` и `nc`.

## Перед началом

- Вы прошли [Урок 15.7 — Проектирование протокола: фрейминг и разбор](07-designing-a-protocol.md). Этот урок применяет его три исхода (полный, пока нет, недопустимо) к настоящему протоколу.
- Вы написали первый HTTP-сервер в [Уроке 06.11 — Проект: многопоточный веб-сервер](../06-rust-advanced/11-project-multithreaded-web-server.md): он читал один запрос на соединение одним `read` и отвечал с `Connection: close`. Теперь вы сделаете это как следует.
- В `code/p15-net` есть `tokio` и `bytes` (Уроки 15.6 и 15.7). `curl` входит в macOS.

## Новые слова

| Слово | Значение |
|---|---|
| **строка запроса** (request line) | Первая строка HTTP-запроса: метод, путь (*цель запроса*), версия, например `GET /index.html HTTP/1.1`. |
| **строка состояния** (status line) | Первая строка ответа: версия, код состояния и поясняющая фраза, например `HTTP/1.1 404 Not Found`. |
| **метод** (method) | Что хочет сделать клиент: `GET` (прочитать), `POST` (отправить данные), `PUT`, `DELETE`, `HEAD` и ещё несколько. |
| **поле заголовка** (header field) | Одна строка `Name: value` после стартовой строки. Имена не зависят от регистра. |
| **тело сообщения** (message body) | Байты после пустой строки: отправленная форма, JSON, веб-страница. |
| **код состояния** (status code) | Трёхзначное число: `2xx` — успех, `3xx` — перенаправление, `4xx` — ошибка клиента, `5xx` — ошибка сервера. |
| **постоянное соединение** (persistent connection, keep-alive) | Одно TCP-соединение, по которому идут несколько запросов один за другим. По умолчанию в HTTP/1.1. |
| **конвейерная отправка** (pipelining) | Отправка следующего запроса до того, как пришёл ответ на предыдущий. Ответы должны возвращаться в том же порядке. |
| **кодирование по частям** (chunked transfer coding) | Отправка тела кусками, у каждого из которых впереди его размер; заканчивается куском размера 0. Используется, когда общий размер заранее неизвестен. |
| **контрабанда запросов** (request smuggling) | Атака, при которой две HTTP-программы (например, прокси и сервер) расходятся во мнении, где кончается запрос, и атакующий может спрятать второй запрос внутри первого. |
| **CRLF** (Carriage Return, Line Feed, возврат каретки и перевод строки) | Два байта `\r\n` (13 и 10), которыми заканчивается каждая строка в заголовочной части HTTP-сообщения. |
| **HTTP** (HyperText Transfer Protocol, протокол передачи гипертекста) | Протокол запросов и ответов веба, тема этого урока. |
| **TCP, IP, RTT** (Transmission Control Protocol, Internet Protocol, Round-Trip Time — протокол управления передачей, межсетевой протокол, время приёма-передачи) | Поток байтов под HTTP/1.1, протокол, который переносит его пакеты, и время, за которое сообщение доходит до другой стороны и обратно (Уроки 15.1 и 15.3). |
| **HTTPS, TLS** (HTTP Secure, Transport Layer Security — защищённый HTTP, протокол защиты транспортного уровня) | HTTP, передаваемый внутри зашифрованного соединения TLS (Урок 15.12). |
| **RFC, JSON, URL** (Request for Comments, JavaScript Object Notation, Uniform Resource Locator — «запрос комментариев», нотация объектов JavaScript, единообразный указатель ресурса) | Документы, которые определяют интернет-стандарты; распространённый текстовый формат данных; веб-адрес вроде `http://127.0.0.1:7300/echo`. |
| **КБ, МБ** (KB, MB — килобайт, мегабайт) | Около тысячи и около миллиона байтов. |

## 1. Как выглядит сообщение HTTP/1.1

Форму вы видели в Уроке 06.11. Вот точные правила (RFC 9112). Запрос выглядит так:

```text
request-line    GET /missing HTTP/1.1\r\n            method SP target SP version CRLF
header fields   Host: 127.0.0.1:7300\r\n             name ":" optional spaces value CRLF
                User-Agent: curl/8.7.1\r\n
                Accept: */*\r\n
empty line      \r\n                                 end of the head
message body    (0 or more bytes)                    how many? see section 2
```

Ответ устроен так же, только первой идёт **строка состояния**:

```text
HTTP/1.1 404 Not Found\r\n
Content-Type: text/plain\r\n
Content-Length: 10\r\n
\r\n
not found\n
```

Правила, важные для парсера:

1. Каждая строка заголовочной части заканчивается на **CRLF** (`\r\n`), а **пустая строка** завершает
   её. Значит, заголовочная часть разбивается разделителями, как в Уроке 15.7.
2. Строка запроса состоит ровно из трёх частей, разделённых одиночными пробелами.
3. Имена заголовков **не зависят от регистра**: `Content-Length`, `content-length` и `CONTENT-LENGTH` —
   один и тот же заголовок. Вокруг значения могут быть пробелы, которые в значение не входят.
4. Запросы HTTP/1.1 обязаны содержать заголовок `Host`, потому что один IP-адрес часто обслуживает
   много сайтов.
5. Программы смотрят на код состояния; поясняющая фраза («Not Found») — только для людей.

Заголовочная часть — текст, а тело может состоять из любых байтов. Поэтому следующий вопрос требует
внимания.

## 2. Где кончается тело?

Заголовочная часть кончается на пустой строке. Для тела запрос использует одно из двух правил:

- **`Content-Length: N`**: следом идут ровно `N` байтов. Это фрейминг с префиксом длины, записанным
  текстом.
- **`Transfer-Encoding: chunked`**: тело приходит кусками. Каждый кусок — это его размер в
  *шестнадцатеричном* виде, CRLF, данные, CRLF. Кусок размера 0 завершает тело:

```text
4\r\n          ← 4 bytes follow
Wiki\r\n
5\r\n          ← 5 bytes follow
pedia\r\n
0\r\n          ← the last chunk
\r\n           ← empty line: the body is complete ("Wikipedia")
```

У запроса без обоих заголовков тела нет. (У *ответа* есть третий вариант: с `Connection: close` и без
длины тело кончается, когда сервер закрывает соединение. На это и полагался ваш сервер из
Урока 06.11.)

> 🔒 **Безопасность:** если в запросе есть и `Content-Length`, и `Transfer-Encoding` или два разных
> значения `Content-Length`, две HTTP-программы могут разойтись во мнении, где он кончается. Прокси может
> увидеть один запрос там, где сервер за ним видит два, и спрятанный второй запрос атакующего обходит
> проверки прокси. Это **контрабанда запросов**. RFC 9112 велит серверам считать такие запросы ошибками.
> Наш парсер отклоняет каждый запрос с заголовком `Transfer-Encoding` (он поддерживает только
> `Content-Length`) и каждый запрос с более чем одним `Content-Length`.

## 3. Строгий парсер в виде чистой функции

Как в Уроке 15.7, парсер не трогает сеть. Он получает уже пришедшие байты и отвечает одним из трёх
результатов: полный запрос плюс число использованных байтов, «пока нет» или ошибка. Весь сервер — один
файл `src/bin/http_server.rs`, показанный в трёх частях. Часть 1, парсер:

```rust
use bytes::{Buf, BytesMut};
use std::time::Duration;
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};
use tokio::time::timeout;

const MAX_HEAD: usize = 8 * 1024; // request line + headers
const MAX_BODY: usize = 1024 * 1024;

#[derive(Debug, PartialEq)]
struct Request {
    method: String,
    path: String,
    version: String,
    headers: Vec<(String, String)>, // names in lower case
    body: Vec<u8>,
}

impl Request {
    fn header(&self, name: &str) -> Option<&str> {
        let mut values = self.headers.iter().filter(|(n, _)| n == name);
        values.next().map(|(_, v)| v.as_str())
    }
}

#[derive(Debug, PartialEq)]
enum HttpError {
    BadRequest(&'static str),
    HeadTooLarge,
    BodyTooLarge,
}

fn find(haystack: &[u8], needle: &[u8]) -> Option<usize> {
    haystack.windows(needle.len()).position(|w| w == needle)
}

/// Ok(None): the request is not complete yet.
/// Ok(Some((request, n))): a complete request that used the first `n` bytes of `buf`.
fn parse_request(buf: &[u8]) -> Result<Option<(Request, usize)>, HttpError> {
    let Some(head_end) = find(buf, b"\r\n\r\n") else {
        return if buf.len() > MAX_HEAD {
            Err(HttpError::HeadTooLarge)
        } else {
            Ok(None)
        };
    };
    if head_end > MAX_HEAD {
        return Err(HttpError::HeadTooLarge);
    }
    let head = std::str::from_utf8(&buf[..head_end])
        .map_err(|_| HttpError::BadRequest("head is not UTF-8"))?;
    let mut lines = head.split("\r\n");

    // Request line: METHOD SP PATH SP VERSION
    let mut parts = lines.next().unwrap_or("").split(' ');
    let (Some(method), Some(path), Some(version), None) =
        (parts.next(), parts.next(), parts.next(), parts.next())
    else {
        return Err(HttpError::BadRequest("bad request line"));
    };
    if version != "HTTP/1.1" && version != "HTTP/1.0" {
        return Err(HttpError::BadRequest("unsupported HTTP version"));
    }

    // Headers: NAME ":" VALUE, names are case-insensitive
    let mut headers = Vec::new();
    for line in lines {
        let (name, value) = line
            .split_once(':')
            .ok_or(HttpError::BadRequest("header without ':'"))?;
        if name.is_empty() || name.contains([' ', '\t']) {
            return Err(HttpError::BadRequest("bad header name"));
        }
        headers.push((name.to_ascii_lowercase(), value.trim().to_string()));
    }
    let mut req = Request {
        method: method.to_string(),
        path: path.to_string(),
        version: version.to_string(),
        headers,
        body: Vec::new(),
    };

    // Body: only Content-Length is supported here.
    if req.header("transfer-encoding").is_some() {
        return Err(HttpError::BadRequest("Transfer-Encoding is not supported"));
    }
    let lengths = req
        .headers
        .iter()
        .filter(|(n, _)| n == "content-length")
        .count();
    if lengths > 1 {
        return Err(HttpError::BadRequest("more than one Content-Length"));
    }
    let body_len: usize = match req.header("content-length") {
        None => 0,
        Some(v) => v
            .parse()
            .map_err(|_| HttpError::BadRequest("bad Content-Length"))?,
    };
    if body_len > MAX_BODY {
        return Err(HttpError::BodyTooLarge);
    }
    let body_start = head_end + 4;
    if buf.len() < body_start + body_len {
        return Ok(None); // the body has not fully arrived
    }
    req.body = buf[body_start..body_start + body_len].to_vec();
    Ok(Some((req, body_start + body_len)))
}
```

**Что происходит:**

1. `find` ищет пустую строку `\r\n\r\n`. Пока её нет, заголовочная часть не закончена; но если без неё
   пришло больше 8 КиБ, клиент получает ошибку вместо безграничной буферизации.
2. Строка запроса должна делиться ровно на три части: образец `(Some, Some, Some, None)` в `let ... else`
   отклоняет и две части, и четыре.
3. Имена заголовков хранятся в нижнем регистре, поэтому `header("content-length")` находит их, как бы
   их ни записал клиент. `trim()` убирает необязательные пробелы вокруг значения.
4. Правила для тела идут последними: никакого `Transfer-Encoding`, не больше одного `Content-Length`,
   корректное число, не больше 1 МиБ. Затем, если тело ещё не пришло целиком, результат — «пока нет».
   (Одна поблажка остаётся: `str::parse::<usize>` принимает и ведущий `+`, поэтому
   `Content-Length: +5` проходит. RFC 9110 разрешает только цифры; рабочий парсер дополнительно
   проверяет `v.bytes().all(|b| b.is_ascii_digit())`.)
5. При успехе функция возвращает, сколько байтов она использовала. Всё, что идёт в буфере дальше,
   принадлежит **следующему** запросу (конвейерная отправка), и вызывающий код должен это сохранить.

## 4. Keep-alive: цикл по запросам

HTTP/1.0 закрывал соединение после каждого ответа, и это стоило рукопожатия (один RTT, Урок 15.3) на
каждый запрос. HTTP/1.1 по умолчанию держит соединение открытым: это **постоянное соединение**.
Клиент или сервер завершает его заголовком `Connection: close`. Поэтому задача соединения — это цикл:
разобрать каждый полный запрос в буфере и, только когда их не осталось, читать новые байты. Часть 2
файла:

```rust
fn route(req: &Request) -> (u16, &'static str, String) {
    match (req.method.as_str(), req.path.as_str()) {
        ("GET", "/") => (200, "OK", "hello\n".to_string()),
        ("POST", "/echo") => (200, "OK", String::from_utf8_lossy(&req.body).into_owned()),
        (_, "/" | "/echo") => (
            405,
            "Method Not Allowed",
            "method not allowed\n".to_string(),
        ),
        _ => (404, "Not Found", "not found\n".to_string()),
    }
}

fn response(status: u16, reason: &str, body: &str, close: bool) -> Vec<u8> {
    let connection = if close { "Connection: close\r\n" } else { "" };
    format!(
        "HTTP/1.1 {status} {reason}\r\nContent-Type: text/plain\r\nContent-Length: {}\r\n{connection}\r\n{body}",
        body.len()
    )
    .into_bytes()
}

fn wants_close(req: &Request) -> bool {
    match req.header("connection") {
        Some(v) if v.eq_ignore_ascii_case("close") => true,
        Some(v) if v.eq_ignore_ascii_case("keep-alive") => false,
        _ => req.version == "HTTP/1.0", // HTTP/1.1 keeps connections open by default
    }
}

async fn handle(mut stream: TcpStream) -> std::io::Result<()> {
    let mut buf = BytesMut::with_capacity(4096);
    loop {
        // First serve every complete request that is already in the buffer.
        match parse_request(&buf) {
            Ok(Some((req, used))) => {
                buf.advance(used);
                let close = wants_close(&req);
                let (status, reason, body) = route(&req);
                println!("{} {} -> {status}", req.method, req.path);
                stream
                    .write_all(&response(status, reason, &body, close))
                    .await?;
                if close {
                    return Ok(());
                }
                continue;
            }
            Ok(None) => {} // need more bytes
            Err(e) => {
                let (status, reason) = match e {
                    HttpError::BadRequest(_) => (400, "Bad Request"),
                    HttpError::HeadTooLarge => (431, "Request Header Fields Too Large"),
                    HttpError::BodyTooLarge => (413, "Content Too Large"),
                };
                println!("bad request: {e:?} -> {status}");
                let body = format!("{reason}\n");
                stream
                    .write_all(&response(status, reason, &body, true))
                    .await?;
                return Ok(());
            }
        }
        // Then read more, but do not wait forever for a slow or silent client.
        match timeout(Duration::from_secs(10), stream.read_buf(&mut buf)).await {
            Ok(Ok(0)) => return Ok(()), // the client closed the connection
            Ok(Ok(_)) => {}
            Ok(Err(e)) => return Err(e),
            Err(_) => return Ok(()), // idle for 10 seconds: close
        }
    }
}

#[tokio::main]
async fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7300").await?;
    println!("listening on http://127.0.0.1:7300");
    loop {
        let (stream, _) = listener.accept().await?;
        tokio::spawn(handle(stream));
    }
}
```

**Что происходит:**

1. `route` сопоставляет методу и пути код состояния, поясняющую фразу и тело. Известный путь с неверным
   методом получает `405`, неизвестный путь — `404`.
2. `response` всегда отправляет `Content-Length`, чтобы клиент знал, где кончается наше тело, и мог
   переиспользовать соединение.
3. В `handle` вызов `stream.read_buf(&mut buf)` дописывает пришедшее в конец `BytesMut` (он растёт по
   мере надобности); `buf.advance(used)` убирает обслуженный запрос из начала. Оставшиеся байты
   следующего запроса остаются в буфере и разбираются на следующем витке цикла, *до* нового чтения.
4. На ошибку разбора отвечают подходящим кодом (`400`, `431` или `413`) с `Connection: close`, а затем
   соединение закрывается: после ошибки фрейминга нельзя знать, где начался бы следующий запрос.
5. Тайм-аут в 10 секунд вокруг каждого чтения закрывает простаивающие keep-alive-соединения и медленных
   клиентов.

Запустите сервер командой `cargo run --bin http_server` и во втором терминале запросите два URL одной
командой `curl`. `-v` показывает, что curl отправляет (`>`), получает (`<`) и делает (`*`):

```bash
curl -v http://127.0.0.1:7300/ http://127.0.0.1:7300/missing
```

```text
*   Trying 127.0.0.1:7300...
* Connected to 127.0.0.1 (127.0.0.1) port 7300
> GET / HTTP/1.1
> Host: 127.0.0.1:7300
> User-Agent: curl/8.7.1
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 200 OK
< Content-Type: text/plain
< Content-Length: 6
< 
{ [6 bytes data]
* Connection #0 to host 127.0.0.1 left intact
hello
* Found bundle for host: 0x77bf040000 [serially]
* Can not multiplex, even if we wanted to
* Re-using existing connection with host 127.0.0.1
> GET /missing HTTP/1.1
> Host: 127.0.0.1:7300
> User-Agent: curl/8.7.1
> Accept: */*
> 
* Request completely sent off
< HTTP/1.1 404 Not Found
< Content-Type: text/plain
< Content-Length: 10
< 
{ [10 bytes data]
* Connection #0 to host 127.0.0.1 left intact
not found
```

`Connection #0 ... left intact` и `Re-using existing connection` показывают keep-alive в работе: оба
запроса прошли по одному TCP-соединению. («Can not multiplex» означает, что HTTP/1.1 не умеет
отправлять два запроса *одновременно* по одному соединению, в отличие от HTTP/2; Урок 15.13.)

Тело с `POST`:

```bash
curl -i -X POST --data 'hi there' http://127.0.0.1:7300/echo
```

```text
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 8

hi there
```

А теперь **конвейерная отправка** вручную: `nc` отправляет два запроса одной записью, второй — с
`Connection: close`. Оба приходят одним чтением, поэтому сервер отвечает на оба из одного буфера, по
порядку, и закрывает соединение:

```bash
printf 'GET / HTTP/1.1\r\nHost: x\r\n\r\nGET /nope HTTP/1.1\r\nHost: x\r\nConnection: close\r\n\r\n' | nc 127.0.0.1 7300
```

```text
HTTP/1.1 200 OK
Content-Type: text/plain
Content-Length: 6

hello
HTTP/1.1 404 Not Found
Content-Type: text/plain
Content-Length: 10
Connection: close

not found
```

Мусор (`printf 'HELLO\r\n\r\n' | nc 127.0.0.1 7300`) получает `HTTP/1.1 400 Bad Request` с
`Connection: close`, а в терминале сервера появляется `bad request: BadRequest("bad request line") -> 400`.

## 5. Тестирование парсера

Часть 3 файла. Поскольку `parse_request` — чистая функция, её тестам не нужны сокеты:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn parse(text: &str) -> Result<Option<(Request, usize)>, HttpError> {
        parse_request(text.as_bytes())
    }

    #[test]
    fn parses_a_get_request() {
        let (req, used) = parse("GET /a HTTP/1.1\r\nHost: x\r\n\r\n")
            .unwrap()
            .unwrap();
        assert_eq!((req.method.as_str(), req.path.as_str()), ("GET", "/a"));
        assert_eq!(req.header("host"), Some("x"));
        assert_eq!(used, 28);
    }

    #[test]
    fn waits_for_head_and_body() {
        assert_eq!(parse("GET / HTTP/1.1\r\nHost: x\r\n"), Ok(None));
        assert_eq!(
            parse("POST /echo HTTP/1.1\r\nHost: x\r\nContent-Length: 5\r\n\r\nhel"),
            Ok(None)
        );
    }

    #[test]
    fn two_requests_in_one_buffer() {
        let text = "GET /1 HTTP/1.1\r\nHost: x\r\n\r\nGET /2 HTTP/1.1\r\nHost: x\r\n\r\n";
        let (first, used) = parse(text).unwrap().unwrap();
        let (second, _) = parse(&text[used..]).unwrap().unwrap();
        assert_eq!((first.path.as_str(), second.path.as_str()), ("/1", "/2"));
    }

    #[test]
    fn rejects_bad_input() {
        let bad = [
            "GET /\r\n\r\n",
            "GET / HTTP/2\r\n\r\n",
            "GET / HTTP/1.1\r\nHost: x\r\nno colon\r\n\r\n",
            "POST / HTTP/1.1\r\nHost: x\r\nContent-Length: 1\r\nContent-Length: 2\r\n\r\nx",
            "POST / HTTP/1.1\r\nHost: x\r\nTransfer-Encoding: chunked\r\n\r\n",
        ];
        for text in bad {
            assert!(
                matches!(parse(text), Err(HttpError::BadRequest(_))),
                "{text:?}"
            );
        }
        let huge = format!(
            "GET / HTTP/1.1\r\nHost: x\r\nX: {}\r\n\r\n",
            "a".repeat(10_000)
        );
        assert_eq!(parse(&huge).unwrap_err(), HttpError::HeadTooLarge);
    }
}
```

`cargo test --bin http_server` сообщает `test result: ok. 4 passed`.

> 🧪 **Тестирование:** тесты следуют трём исходам из Урока 15.7: полные запросы (и точное число
> использованных байтов, 28), неполные заголовочные части и тела, а также список недопустимых входов.
> Тест с двумя запросами в одном буфере защищает конвейерную отправку, которую легко сломать, выбросив
> оставшиеся байты.

> ⚡ **Производительность:** этот парсер копирует строки и после каждого чтения ищет `\r\n\r\n` с
> начала буфера — для учёбы это нормально. Боевые парсеры, например крейт `httparse` (он работает внутри
> `hyper`, Урок 15.9), разбирают без копирования, прямо из буфера.

## Частые ошибки

> ⚠️ **Частая ошибка: один `read` = один запрос.** Именно так делал сервер из Урока 06.11. Он ломается,
> когда запрос приходит двумя кусками (большие заголовки, медленные сети) и когда два запроса приходят
> одним чтением (конвейерная отправка). Всегда используйте цикл: разбирайте то, что есть, и читайте
> дальше только когда парсер говорит «пока нет».

> ⚠️ **Частая ошибка: нет `Content-Length` в ответах на keep-alive-соединении.** Без него (и без
> кодирования по частям) клиент не может узнать, где кончается ваше тело, и ждёт закрытия соединения,
> которое так и не наступает. Тогда кажется, что `curl` завис.

## Попробуйте сами

1. Выполните `curl -v --http1.0 http://127.0.0.1:7300/`. curl отправляет `HTTP/1.0`, и наш сервер
   закрывает соединение после ответа (найдите `Closing connection` в выводе curl).
2. Выполните `curl -i -X DELETE http://127.0.0.1:7300/`. Ответ — `HTTP/1.1 405 Method Not Allowed`.
3. Откройте соединение через `nc 127.0.0.1 7300`, введите `GET / HTTP/1.1`, Enter, `Host: x`, Enter,
   Enter. (`nc` отправляет `\n`, а не `\r\n`, поэтому сервер ждёт.) Подождите 10 секунд: тайм-аут
   простоя закроет соединение.
4. Отправьте запрос `curl -H 'X-Big: '"$(printf 'a%.0s' {1..9000})" http://127.0.0.1:7300/` и получите
   `431 Request Header Fields Too Large`.

## Упражнения

### Упражнение 1 — Покажите заголовки (легко)

Добавьте маршрут `GET /headers`, который отвечает всеми заголовками запроса, по одному `name: value` в
строке, в порядке поступления. Проверьте командой `curl http://127.0.0.1:7300/headers`.

<details>
<summary>Подсказка</summary>

Маршрут получает весь `Request`. Превратите каждую пару `(name, value)` в строку через `format!` и
соедините строки через `concat()`.

</details>

<details>
<summary>Ответ</summary>

Добавьте эту ветку в `route` после ветки `("GET", "/")`:

```rust
        ("GET", "/headers") => {
            let lines: Vec<String> = req
                .headers
                .iter()
                .map(|(n, v)| format!("{n}: {v}\n"))
                .collect();
            (200, "OK", lines.concat())
        }
```

`curl http://127.0.0.1:7300/headers` печатает:

```text
host: 127.0.0.1:7300
user-agent: curl/8.7.1
accept: */*
```

Имена в нижнем регистре, потому что так их хранит парсер. Положение новой ветки здесь не важно, потому
что `/headers` не подходит ни под один другой образец; держать все маршруты `GET` вместе — только ради
читаемости.

</details>

### Упражнение 2 — `Host` обязателен, с тестом (средне)

RFC 9112 говорит, что сервер должен отвечать `400 Bad Request` на запрос HTTP/1.1 без заголовка `Host`.
Запросы HTTP/1.0 могут его не содержать. Добавьте это правило в `parse_request` и тест для обоих
случаев. Проходят ли существующие тесты?

<details>
<summary>Подсказка</summary>

Проверяйте сразу после цикла по заголовкам, до создания `Request`. `headers.iter().any(...)` скажет,
есть ли такое имя.

</details>

<details>
<summary>Ответ</summary>

Вставьте в `parse_request` прямо перед `let mut req = Request {`:

```rust
    if version == "HTTP/1.1" && !headers.iter().any(|(n, _)| n == "host") {
        return Err(HttpError::BadRequest("missing Host header"));
    }
```

и добавьте в `mod tests`:

```rust
    #[test]
    fn host_is_required_in_http_1_1_only() {
        let err = parse("GET / HTTP/1.1\r\n\r\n").unwrap_err();
        assert_eq!(err, HttpError::BadRequest("missing Host header"));
        assert!(parse("GET / HTTP/1.0\r\n\r\n").unwrap().is_some());
    }
```

`cargo test --bin http_server` сообщает `5 passed`. Существующие тесты по-прежнему проходят, потому что
в каждом запросе HTTP/1.1 в них уже есть `Host: x`. Так и задумано: тестовые входы должны быть
правильными во всём, кроме того, что проверяет тест. Иначе новое правило роняет старые тесты или, что
хуже, заставляет их проходить по неверной причине («недопустимый» вход отклоняется из-за отсутствия
`Host`, а не из-за ошибки, ради которой тест был написан).

</details>

### Упражнение 3 — Раскодируйте тело по частям, с тестами (сложно)

Напишите `src/bin/chunked.rs` с функцией `fn decode_chunked(buf: &[u8]) -> Result<Option<(Vec<u8>,
usize)>, ChunkError>` той же формы с тремя исходами, что и `parse_request`, для формата по частям из
раздела 2. Принимайте шестнадцатеричные размеры в верхнем и нижнем регистре, игнорируйте *расширения
кусков* (`;name=value` после размера), отклоняйте тела больше 1 МиБ, отклоняйте кусок, за которым не
следует CRLF, и не поддерживайте трейлеры (строки заголовков после последнего куска). Протестируйте
пример из раздела 2, каждый его неполный префикс и плохой ввод.

<details>
<summary>Подсказка</summary>

Храните позицию `pos` в буфере. В цикле: найдите следующий CRLF от `pos`, разберите размер через
`usize::from_str_radix(hex, 16)`, затем используйте `buf.get(pos..pos + size + 2)`, который возвращает
`None`, а не паникует, если данные ещё не пришли. Проверяйте предел *до* ожидания данных, как в
Уроке 15.7: `size > MAX_BODY - body.len()` не может переполниться.

</details>

<details>
<summary>Ответ</summary>

```rust
const MAX_BODY: usize = 1024 * 1024;

#[derive(Debug, PartialEq)]
enum ChunkError {
    Bad(&'static str),
    TooLarge,
}

/// Decodes a chunked body at the start of `buf`.
/// Ok(None): not complete yet. Ok(Some((body, n))): done, `n` bytes used.
fn decode_chunked(buf: &[u8]) -> Result<Option<(Vec<u8>, usize)>, ChunkError> {
    let mut body = Vec::new();
    let mut pos = 0;
    loop {
        // The size line: hex digits, maybe ";extension", then CRLF.
        let Some(line_len) = buf[pos..].windows(2).position(|w| w == b"\r\n") else {
            return Ok(None);
        };
        let line = std::str::from_utf8(&buf[pos..pos + line_len])
            .map_err(|_| ChunkError::Bad("size line is not text"))?;
        let hex = line.split(';').next().unwrap_or("").trim();
        let size = usize::from_str_radix(hex, 16).map_err(|_| ChunkError::Bad("bad chunk size"))?;
        pos += line_len + 2;
        if size > MAX_BODY - body.len() {
            return Err(ChunkError::TooLarge);
        }
        if size == 0 {
            // The last chunk is followed by an empty line (we do not support trailers).
            return match buf.get(pos..pos + 2) {
                None => Ok(None),
                Some(b"\r\n") => Ok(Some((body, pos + 2))),
                Some(_) => Err(ChunkError::Bad("trailers are not supported")),
            };
        }
        // The data, followed by CRLF.
        let Some(chunk) = buf.get(pos..pos + size + 2) else {
            return Ok(None);
        };
        if &chunk[size..] != b"\r\n" {
            return Err(ChunkError::Bad("chunk is not followed by CRLF"));
        }
        body.extend_from_slice(&chunk[..size]);
        pos += size + 2;
    }
}

fn main() {
    let wire = b"4\r\nWiki\r\n5\r\npedia\r\n0\r\n\r\nGET /next";
    let (body, used) = decode_chunked(wire).unwrap().unwrap();
    println!("body: {:?}", String::from_utf8_lossy(&body));
    println!("used {used} of {} bytes", wire.len());
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn decodes_the_example() {
        let wire = b"4\r\nWiki\r\n5\r\npedia\r\n0\r\n\r\n";
        assert_eq!(
            decode_chunked(wire),
            Ok(Some((b"Wikipedia".to_vec(), wire.len())))
        );
    }

    #[test]
    fn every_prefix_is_incomplete() {
        let wire = b"4\r\nWiki\r\n5\r\npedia\r\n0\r\n\r\n";
        for end in 0..wire.len() {
            assert_eq!(
                decode_chunked(&wire[..end]),
                Ok(None),
                "prefix of {end} bytes"
            );
        }
    }

    #[test]
    fn rejects_bad_chunks() {
        assert_eq!(
            decode_chunked(b"zz\r\n"),
            Err(ChunkError::Bad("bad chunk size"))
        );
        assert_eq!(
            decode_chunked(b"2\r\nabXY"),
            Err(ChunkError::Bad("chunk is not followed by CRLF"))
        );
        assert_eq!(decode_chunked(b"FFFFFFFF\r\n"), Err(ChunkError::TooLarge));
    }
}
```

`cargo run --bin chunked` печатает `body: "Wikipedia"` и `used 24 of 33 bytes`, а
`cargo test --bin chunked` сообщает `3 passed`. Почему это работает: каждое обращение к срезу идёт через
`get`, поэтому короткий буфер означает «пока нет», а не панику; тест по всем префиксам доказывает это
для каждой возможной точки разреза. 9 неиспользованных байтов (`GET /next`) принадлежат следующему
запросу, точно как в `parse_request`.

</details>

## Вопросы для самопроверки

1. Как сервер узнаёт, где кончается заголовочная часть запроса HTTP/1.1 и где кончается его тело?

<details>
<summary>Ответ</summary>

Заголовочная часть кончается на первой пустой строке (`\r\n\r\n`). Тело имеет длину ровно
`Content-Length` байтов или передаётся по частям (`Transfer-Encoding: chunked`) и кончается куском
размера 0; без обоих заголовков тела нет.

</details>

2. Чем опасен запрос, в котором есть и `Content-Length`, и `Transfer-Encoding`?

<details>
<summary>Ответ</summary>

Две программы на пути могут искать конец по разным заголовкам и разойтись во мнении, где кончается
запрос. Тогда атакующий может спрятать второй запрос (контрабанда запросов). Серверы обязаны такой
запрос отклонять.

</details>

3. Что экономит keep-alive и как завершается постоянное соединение?

<details>
<summary>Ответ</summary>

Он экономит рукопожатие TCP (а с HTTPS ещё и рукопожатие TLS) для каждого запроса после первого. Любая
сторона завершает его, отправив `Connection: close` (или просто закрыв соединение, например после
тайм-аута простоя).

</details>

4. Клиент отправляет два запроса конвейером. Почему сервер никогда не должен выбрасывать байты,
   оставшиеся в буфере после разбора первого?

<details>
<summary>Ответ</summary>

Это начало (или весь) второго запроса. Выбросив их, сервер потеряет этот запрос, и клиент будет вечно
ждать ответа на него.

</details>

5. Какие коды состояния отправляет наш сервер на испорченный запрос, на заголовок в 20 КБ и на тело в
   5 МБ?

<details>
<summary>Ответ</summary>

`400 Bad Request`, `431 Request Header Fields Too Large` и `413 Content Too Large`.

</details>

## Итоги

- Сообщение HTTP/1.1 — это стартовая строка, строки заголовков, пустая строка и необязательное тело;
  строки заголовочной части кончаются на CRLF, имена заголовков не зависят от регистра, а запросам
  HTTP/1.1 нужен `Host`.
- Тело кончается через `Content-Length` байтов или на куске нулевого размера при передаче по частям.
  Противоречащие друг другу заголовки длины отклоняются, чтобы предотвратить контрабанду запросов.
- Соединения HTTP/1.1 по умолчанию постоянные; `Connection: close` их завершает. Запросы, отправленные
  конвейером, делят один буфер, поэтому сервер разбирает их в цикле и сохраняет оставшиеся байты.
- Хороший парсер — чистая функция с тремя исходами и жёсткими пределами на размер заголовочной части и
  тела; цикл соединения добавляет тайм-ауты и отвечает на ошибки кодами `400`, `413` или `431`, прежде
  чем закрыть соединение.
- `curl -v` показывает, что на самом деле идёт по сети, включая переиспользование соединения; `nc`
  позволяет отправить любые байты.

## Источники и что почитать

- [RFC 9112 — HTTP/1.1](https://datatracker.ietf.org/doc/html/rfc9112) — синтаксис сообщений: разделы 2–7 (формат, строка запроса, длина тела, кодирование по частям) и 9 (соединения, keep-alive, конвейерная отправка).
- [RFC 9110 — HTTP Semantics](https://datatracker.ietf.org/doc/html/rfc9110) — методы, коды состояния и смысл заголовков, общие для всех версий HTTP.
- [MDN — An overview of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Overview) — дружелюбное введение со схемами.
- [PortSwigger — HTTP request smuggling](https://portswigger.net/web-security/request-smuggling) — как эта атака работает на практике.
- [Крейт `httparse`](https://docs.rs/httparse/latest/httparse/) — быстрый парсер HTTP/1 без копирования, который использует `hyper`.

---

[🏠 Начало](../START-HERE.md) · [← 15.7 Проектирование протокола: фрейминг и разбор](07-designing-a-protocol.md) · [📋 Этап 15](README.md) · [15.9 HTTP с hyper и reqwest →](09-http-with-hyper-and-reqwest.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
