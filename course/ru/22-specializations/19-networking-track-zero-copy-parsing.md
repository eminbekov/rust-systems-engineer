<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.18 Трек сетей: высокопроизводительные серверы](18-networking-track-high-performance-servers.md) · [📋 Этап 22](README.md) · [22.20 Трек сетей: прокси для продакшена →](20-networking-track-production-proxies.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/19-networking-track-zero-copy-parsing.md) <!-- nav -->

# Урок 22.19 — Трек сетей: разбор протоколов без копирования

> **Этап 22 — Специализации** · Урок 19 из 30 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Что значит разбор «без копирования» (zero-copy) и почему заимствованные срезы с временами жизни делают его безопасным в Rust.
- Как написать парсер заголовочной части HTTP/1.1-запроса, который возвращает срезы входных данных.
- Как обрабатывать ввод, который приходит по частям: чем «неверно» отличается от «ещё не полностью».
- Как крейт `bytes` (`BytesMut`, `Bytes`) позволяет хранить разобранные части, не копируя их.
- Как тот же парсер выражается крейтом парсер-комбинаторов `nom` и как тестировать парсер свойствами.

## Перед началом

- Вы прошли [Урок 22.18 — Трек сетей: высокопроизводительные серверы](18-networking-track-high-performance-servers.md).
- Вы знаете фрейминг из [Урока 15.7 — Проектирование протокола: фрейминг и разбор](../15-networking/07-designing-a-protocol.md),
  сообщения HTTP (Hypertext Transfer Protocol, протокол передачи гипертекста) из [Урока 15.8 — HTTP с нуля](../15-networking/08-http-from-scratch.md),
  времена жизни из [Урока 06.5 — Времена жизни подробно](../06-rust-advanced/05-lifetimes-in-depth.md) и тесты на основе свойств из
  [Урока 09.4 — Тестирование на основе свойств](../09-testing-debugging-benchmarking/04-property-based-testing.md).
- Добавьте крейты в `code/p22-net` и создайте файл модуля `src/parse.rs`, а в начало `src/lib.rs`
  добавьте строку `pub mod parse;`:

```bash
cd ~/projects/rust/code/p22-net
cargo add bytes@1
cargo add nom@8
cargo add --dev proptest@1
```

## Новые слова

| Слово | Значение |
|---|---|
| **разбор без копирования** (zero-copy parsing) | Разбор, который не копирует вход: результаты — это *представления* (заимствованные срезы) байтов, которые уже лежат в памяти. |
| **заголовочная часть запроса** (request head) | Первая часть HTTP/1.1-запроса: строка запроса и заголовки, в конце — пустая строка. Тело (если оно есть) идёт после неё. |
| **CRLF** (Carriage Return, Line Feed — возврат каретки, перевод строки) | Два байта `\r\n`, которыми заканчивается каждая строка в HTTP/1.1. |
| **неполный ввод** (partial input) | Байты, которые являются правильным *началом* сообщения, но ещё не всем сообщением. Правильная реакция — «прочитать ещё», а не «ошибка». |
| **конвейерная передача** (pipelining) | Клиент отправляет несколько запросов, не дожидаясь ответов, поэтому одно чтение может содержать больше одного запроса. |
| **`BytesMut`** | Растущий буфер байтов из крейта `bytes`, в который читают из сокетов. |
| **`Bytes`** | Дешёвый, разделяемый, только для чтения кусок буфера байтов. Клонирование или взятие среза не копирует байты — только указатель и счётчик ссылок. |
| **счётчик ссылок** (reference count) | Число рядом с общими данными, которое считает их владельцев; данные освобождаются, когда счётчик падает до нуля (как у `Arc`, Урок 05.10). |
| **парсер-комбинатор** (parser combinator) | Маленький парсер, который соединяют с другими в большие парсеры, как детали конструктора. `nom` — библиотека таких парсеров. |

## 1. Сколько стоит копирование и что значит «без копирования»

Прокси читает каждый запрос из сети в буфер. Простой парсер затем строит структуру с полями `String`:
метод, путь, имя и значение каждого заголовка. Каждая `String` — это новое выделение памяти в куче плюс
копирование байтов (Урок 12.13). Для одного запроса это ничто. Для 100 000 запросов в секунду по 15
заголовков это миллионы выделений в секунду, и аллокатор с копированием могут занять больше времени,
чем сама работа.

Парсер **без копирования** не создаёт новых строк. Он возвращает *срезы, указывающие внутрь буфера*:

```text
 buf (bytes from the socket, already in memory)
 ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬────┬────┬───┬───┬───┬───┬───┬
 │ G │ E │ T │   │ / │ h │ e │ l │ l │ o │   │ H  │ T  │ T │ P │ / │ 1 │ . │ ...
 └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴────┴────┴───┴───┴───┴───┴───┴
   ▲           ▲
   │           └─ path:   &str { pointer to byte 4, length 6 }
   └─ method:  &str { pointer to byte 0, length 3 }
```

Срез — это просто указатель и длина (16 байтов на 64-битной машине), каким бы ни был размер данных. У
приёма есть одна опасность: если буфер переиспользуют, пока кто-то держит срез, срез молча
покажет не те байты. В Rust **время жизни** в типе результата привязывает каждый срез к буферу, и
проверка заимствований отвергает любую программу, которая меняет буфер, пока срез жив. Вы получаете
скорость без риска.

## 2. Парсер заголовочной части, написанный вручную

Заголовочная часть HTTP/1.1-запроса выглядит так (каждая строка заканчивается CRLF, пустая строка
завершает заголовочную часть):

```text
GET /hello HTTP/1.1\r\n
Host: example.com\r\n
Accept: */*\r\n
\r\n
```

Парсер должен ответить одно из трёх: «вот полная заголовочная часть, её длина N байтов», «ещё не
полностью, прочитайте ещё» или «это неправильный запрос». Поместите это в `src/parse.rs`:

```rust
//! A zero-copy parser for the head of an HTTP/1.1 request.

/// The largest request head we accept, in bytes.
pub const MAX_HEAD: usize = 8 * 1024;

#[derive(Debug, PartialEq)]
pub struct Request<'a> {
    pub method: &'a str,
    pub path: &'a str,
    pub headers: Vec<(&'a str, &'a [u8])>,
}

#[derive(Debug, PartialEq)]
pub enum Parsed<'a> {
    /// A whole head was found; it is `len` bytes long (with the empty line).
    Complete { request: Request<'a>, len: usize },
    /// Not enough bytes yet: read more and call again.
    Partial,
}

#[derive(Debug, PartialEq)]
pub enum ParseError {
    BadRequestLine,
    BadHeader,
    TooLarge,
}

pub fn parse_request(buf: &[u8]) -> Result<Parsed<'_>, ParseError> {
    let Some(end) = find(buf, b"\r\n\r\n") else {
        return if buf.len() > MAX_HEAD {
            Err(ParseError::TooLarge)
        } else {
            Ok(Parsed::Partial)
        };
    };
    if end + 4 > MAX_HEAD {
        return Err(ParseError::TooLarge);
    }
    let mut lines = buf[..end].split(|&b| b == b'\n');

    let request_line = lines.next().ok_or(ParseError::BadRequestLine)?;
    let request_line = request_line.strip_suffix(b"\r").unwrap_or(request_line);
    let mut parts = request_line.split(|&b| b == b' ');
    let (Some(method), Some(path), Some(b"HTTP/1.1"), None) =
        (parts.next(), parts.next(), parts.next(), parts.next())
    else {
        return Err(ParseError::BadRequestLine);
    };
    let method = text(method, ParseError::BadRequestLine)?;
    let path = text(path, ParseError::BadRequestLine)?;

    let mut headers = Vec::new();
    for line in lines {
        let line = line.strip_suffix(b"\r").unwrap_or(line);
        let colon = line
            .iter()
            .position(|&b| b == b':')
            .ok_or(ParseError::BadHeader)?;
        let name = text(&line[..colon], ParseError::BadHeader)?;
        headers.push((name, line[colon + 1..].trim_ascii()));
    }
    let request = Request {
        method,
        path,
        headers,
    };
    Ok(Parsed::Complete {
        request,
        len: end + 4,
    })
}

/// Position of the first `needle` in `haystack`.
fn find(haystack: &[u8], needle: &[u8]) -> Option<usize> {
    haystack.windows(needle.len()).position(|w| w == needle)
}

/// Borrow `bytes` as `&str` if they are non-empty, visible ASCII characters.
fn text(bytes: &[u8], err: ParseError) -> Result<&str, ParseError> {
    if bytes.is_empty() || !bytes.iter().all(|b| b.is_ascii_graphic()) {
        return Err(err);
    }
    // All bytes are ASCII, so this is valid UTF-8 and cannot fail.
    std::str::from_utf8(bytes).map_err(|_| err)
}
```

**Что происходит:**

1. `Request<'a>` хранит только заимствованные данные: `&'a str` и `&'a [u8]`. Время жизни `'a`
   говорит: «эти срезы живут внутри какого-то буфера, который живёт не меньше `'a`».
   `parse_request(buf: &[u8])` возвращает `Parsed<'_>`, поэтому `'a` — время жизни `buf` (вывод времён
   жизни, lifetime elision, Урок 05.3).
2. Сначала парсер ищет пустую строку `\r\n\r\n`. Если её нет, заголовочная часть не полная: ответ —
   `Partial`, если только буфер уже не больше `MAX_HEAD`, тогда это `TooLarge`. Полная заголовочная
   часть длиннее `MAX_HEAD` — тоже `TooLarge`, поэтому предел соблюдается, как бы ни приходили байты.
3. `split(|&b| b == b'\n')` режет заголовочную часть на строки без копирования; `strip_suffix(b"\r")`
   убирает `\r` из каждой строки, опять же только делая срез короче.
4. В строке запроса должно быть ровно три части, разделённые одиночными пробелами, и третья должна быть
   `HTTP/1.1`. `let ... else` с образцом-кортежем проверяет всё это в одном месте (Урок 06.3).
5. `text` проверяет, что срез содержит только видимые символы ASCII (American Standard Code for
   Information Interchange, американский стандартный код для обмена информацией), и заимствует его как
   `&str`. `from_utf8` не копирует, а только проверяет.
6. Каждый заголовок делится по первому `:`. Значение — `&[u8]`, а не `&str`, потому что HTTP допускает
   в значениях байты, которые не являются корректным UTF-8 (Unicode Transformation Format, 8-bit, формат преобразования Юникода, 8 бит — обычная кодировка текста, Урок 12.5). `trim_ascii` убирает пробелы
   вокруг.
7. `len: end + 4` — длина всей заголовочной части вместе с финальным `\r\n\r\n`, чтобы вызывающий код
   знал, где начинается следующий запрос (или тело).

Теперь подайте ему байты, которые приходят двумя кусками, как из сокета. Сохраните как
`src/bin/parse_demo.rs`:

```rust
use p22_net::parse::{Parsed, parse_request};

fn main() {
    let mut buf: Vec<u8> = Vec::new();

    // The bytes arrive from the network in two pieces.
    buf.extend_from_slice(b"GET /hello HTTP/1.1\r\nHost: exa");
    println!("after piece 1: {:?}", parse_request(&buf));

    buf.extend_from_slice(b"mple.com\r\nAccept: */*\r\n\r\n");
    match parse_request(&buf) {
        Ok(Parsed::Complete { request, len }) => {
            println!("complete, head is {len} of {} bytes", buf.len());
            println!("method = {}, path = {}", request.method, request.path);
            for (name, value) in &request.headers {
                println!("header {name}: {}", String::from_utf8_lossy(value));
            }
            let offset = request.path.as_ptr() as usize - buf.as_ptr() as usize;
            println!("path lives inside buf at offset {offset}");
        }
        other => println!("unexpected: {other:?}"),
    }
}
```

`cargo run --bin parse_demo`:

```text
after piece 1: Ok(Partial)
complete, head is 55 of 55 bytes
method = GET, path = /hello
header Host: example.com
header Accept: */*
path lives inside buf at offset 4
```

Последняя строка доказывает отсутствие копирования: адрес `request.path` равен адресу `buf` плюс 4,
то есть путь *и есть* байты с 4-го по 9-й буфера, а не их копия.

> 🔒 **Безопасность:** `MAX_HEAD` обязателен. Без него клиент, который бесконечно шлёт
> байты без пустой строки, заставит ваш буфер расти, пока процессу не хватит памяти. В каждом настоящем
> HTTP-сервере есть такой предел (обычно от 8 до 64 КиБ). Никогда не доверяйте размерам из сети.

> ⚡ **Производительность:** `find` ищет с начала буфера при каждом вызове. Если медленный клиент шлёт
> по одному байту, это O(n²) работы (Урок 07.1). Парсеры в продакшене запоминают, где остановились,
> или ищут быстрыми функциями, например из крейта `memchr`. Парсер внутри hyper — это крейт
> `httparse`, его стоит почитать после этого урока.

## 3. Как хранить разобранные части: `BytesMut` и `Bytes`

У заимствованных срезов есть одно ограничение: они не могут пережить буфер. А прокси хочет *хранить*
части запроса: отправить путь задаче логирования, держать заголовочную часть, пока вышестоящий сервер
не ответит, и одновременно переиспользовать буфер для следующего запроса. Копирование вернуло бы ту
самую стоимость, которую мы только что убрали.

Крейт `bytes` решает это двумя типами:

- `BytesMut` — буфер, в который вы читаете. Его метод `split_to(n)` отрезает первые `n` байтов в
  отдельное значение *без копирования*: обе части по-прежнему указывают в ту же память, а исходный
  буфер теперь начинается с байта `n`.
- `freeze()` превращает отрезанный `BytesMut` в `Bytes` — дескриптор только для чтения со счётчиком
  ссылок. `clone()` и `slice(range)` у `Bytes` создают новые дескрипторы той же памяти. Память
  освобождается, когда удаляется последний дескриптор. `Bytes` реализует `Send` и имеет время жизни
  `'static`, поэтому его можно передавать в другие задачи и потоки.

Сохраните как `src/bin/bytes_demo.rs`:

```rust
use bytes::{Bytes, BytesMut};
use p22_net::parse::{Parsed, parse_request};

fn main() {
    let mut buf = BytesMut::with_capacity(4096);
    // Two requests arrived in one read (this is called pipelining).
    buf.extend_from_slice(b"GET /a HTTP/1.1\r\nHost: x\r\n\r\nGET /b HTTP/1.1\r\nHost: y\r\n\r\n");
    let start_of_buffer = buf.as_ptr() as usize;

    let mut heads: Vec<(Bytes, Bytes)> = Vec::new();
    while let Ok(Parsed::Complete { request, len }) = parse_request(&buf) {
        // 1. Remember where the path is. `request` borrows `buf` until here.
        let path_start = request.path.as_ptr() as usize - buf.as_ptr() as usize;
        let path_end = path_start + request.path.len();
        // 2. Cut the head off the front of the buffer. No bytes are copied.
        let head: Bytes = buf.split_to(len).freeze();
        // 3. A Bytes for just the path: shares the same memory as `head`.
        let path = head.slice(path_start..path_end);
        heads.push((head, path));
    }

    for (head, path) in &heads {
        println!(
            "path {:?}: head at offset {}, path at offset {}",
            path,
            head.as_ptr() as usize - start_of_buffer,
            path.as_ptr() as usize - start_of_buffer
        );
    }
    println!("bytes left in buf: {}", buf.len());

    // Bytes is 'static and Send: it can move to another thread.
    let (_, path) = heads.remove(1);
    let handle = std::thread::spawn(move || format!("worker got {path:?}"));
    println!("{}", handle.join().unwrap());
}
```

**Что происходит:**

1. В буфере два запроса, пришедших конвейером. Цикл `while let` разбирает по одной заголовочной части
   за проход.
2. Внутри цикла `request` заимствует `buf`. Мы читаем из него только два числа (где путь начинается и
   заканчивается). После этой строки `request` больше не используется, поэтому заимствование
   заканчивается, и следующая строка может менять `buf` (нелексические времена жизни: заимствование
   длится до последнего использования).
3. `buf.split_to(len)` отрезает первую заголовочную часть. Теперь `buf` начинается со второго запроса.
4. `head.slice(...)` создаёт `Bytes` для пути, указывающий в ту же память, что и `head`.
5. Напечатанные смещения отсчитываются от начала исходного буфера: вторая заголовочная часть
   начинается с байта 28, ровно там же, где была в буфере. Ничего не сдвинулось и не скопировалось.
6. В конце путь перемещается в другой поток. С `&[u8]` так нельзя, потому что поток может прожить
   дольше буфера; `Bytes` владеет своей долей памяти.

`cargo run --bin bytes_demo`:

```text
path b"/a": head at offset 0, path at offset 4
path b"/b": head at offset 28, path at offset 32
bytes left in buf: 0
worker got b"/b"
```

Это стандартный приём сетевого кода на Rust: читать в `BytesMut`, разбирать заимствованными срезами,
затем `split_to(...).freeze()` превращает готовое сообщение в `Bytes`, которое передаётся дальше.
`AsyncReadExt::read_buf` из Tokio читает прямо в `BytesMut` (упражнение 3), а hyper, Pingora и
большинство кодеков Tokio передают друг другу `Bytes`.

## 4. Та же идея с `nom`

Писать парсеры вручную нормально для маленького формата. Для больших форматов помогает библиотека
**парсер-комбинаторов**: вы пишете маленькие парсеры («одна или больше заглавных букв», «ровно один
пробел») и соединяете их. В `nom` парсер — это функция, которая принимает вход и возвращает
`Ok((rest, value))` или ошибку, а значения — срезы входа, так что `nom` тоже работает без копирования.
Сохраните как `src/bin/nom_demo.rs`:

```rust
use nom::bytes::streaming::{tag, take_while1};
use nom::{IResult, Parser};

/// `METHOD SP PATH SP HTTP/1.1 CRLF`, returns (method, path).
fn request_line(input: &[u8]) -> IResult<&[u8], (&[u8], &[u8])> {
    let method = take_while1(|b: u8| b.is_ascii_uppercase());
    let path = take_while1(|b: u8| b.is_ascii_graphic());
    let (rest, (method, _, path, _, _)) =
        (method, tag(" "), path, tag(" HTTP/1.1"), tag("\r\n")).parse(input)?;
    Ok((rest, (method, path)))
}

fn main() {
    for input in [
        &b"GET /index.html HTTP/1.1\r\nHost: x\r\n"[..],
        &b"GET /index.ht"[..],
        &b"get / HTTP/1.1\r\n"[..],
    ] {
        match request_line(input) {
            Ok((rest, (method, path))) => println!(
                "ok: method={:?} path={:?}, {} bytes left",
                String::from_utf8_lossy(method),
                String::from_utf8_lossy(path),
                rest.len()
            ),
            Err(nom::Err::Incomplete(needed)) => println!("incomplete: {needed:?}"),
            Err(e) => println!("error: {e:?}"),
        }
    }
}
```

**Что происходит:**

1. `take_while1(f)` — парсер, который берёт один или больше байтов, для которых `f` истинна.
   `tag(" ")` — парсер, который ожидает ровно этот текст.
2. Кортеж парсеров сам является парсером, который запускает их по порядку; `.parse(input)` (из трейта
   `Parser`, который `nom` 8 требует импортировать) запускает его. Результат — остаток входа и кортеж
   из пяти результатов; мы оставляем два из них.
3. Мы импортируем из `nom::bytes::streaming`. **Потоковые** (streaming) версии возвращают
   `Err::Incomplete`, когда вход заканчивается слишком рано, — это наш `Partial`. Версии `complete`
   вернули бы ошибку, что правильно для целого файла, но неправильно для сокета.
4. `?` передаёт ошибки наверх; `IResult` — тип `Result` из `nom`.

`cargo run --bin nom_demo`:

```text
ok: method="GET" path="/index.html", 9 bytes left
incomplete: Size(1)
error: Error(Error { input: [103, 101, 116, 32, 47, 32, 72, 84, 84, 80, 47, 49, 46, 49, 13, 10], code: TakeWhile1 })
```

Третий вход не разбирается, потому что `get` написан строчными буквами, и ошибка показывает, где
(`input` — остаток байтов, `103` — это `g`) и какой парсер не справился. Какой стиль выбрать? Парсер,
написанный вручную, даёт лучший контроль над скоростью и сообщениями об ошибках; `nom` даёт более
короткий код для больших грамматик. В продакшене используются оба.

> 🧪 **Тестирование:** парсер получает ввод от незнакомцев, поэтому тестируйте его вводом, о котором вы
> не подумали. Для каждого сетевого парсера стоит проверять два свойства: «он никогда не паникует,
> какими бы ни были байты» и «каждый строгий префикс правильного сообщения — `Partial`». Упражнение 2
> пишет оба с `proptest`, а следующий шаг — `cargo fuzz` (Урок 09.8).

## Частые ошибки

**Хранить разобранный запрос, переиспользуя буфер.** Сохраните как `src/bin/dangling.rs`:

```rust
use p22_net::parse::{Parsed, parse_request};

fn main() {
    let mut buf = b"GET / HTTP/1.1\r\n\r\n".to_vec();
    let Ok(Parsed::Complete { request, .. }) = parse_request(&buf) else {
        return;
    };
    buf.clear(); // reuse the buffer for the next request
    println!("{}", request.path);
}
```

```text
error[E0502]: cannot borrow `buf` as mutable because it is also borrowed as immutable
 --> src/bin/dangling.rs:8:5
  |
5 |     let Ok(Parsed::Complete { request, .. }) = parse_request(&buf) else {
  |                                                              ---- immutable borrow occurs here
...
8 |     buf.clear(); // reuse the buffer for the next request
  |     ^^^^^^^^^^^ mutable borrow occurs here
9 |     println!("{}", request.path);
  |                    ------------ immutable borrow later used here
```

Здесь проверка заимствований защищает схему без копирования: `request.path` указывает внутрь `buf`,
поэтому `buf` нельзя очистить, пока путь ещё используется. Исправление: закончите работу с запросом до
того, как менять буфер, или сначала превратите нужные части в `Bytes` (раздел 3).

**Считать неполный ввод ошибкой.** TCP (Transmission Control Protocol, протокол управления передачей) даёт вам байты, а не сообщения (Урок 15.4). Парсер, который
возвращает ошибку для `GET /hel`, будет падать случайно, в зависимости от того, как сеть разрезала
байты. Всегда должно быть три исхода: полный, неполный, неверный.

**Забыть, сколько байтов использовано.** После полной заголовочной части вызывающий код должен убрать
из буфера ровно `len` байтов. Если убрать весь буфер, второй запрос из конвейера потеряется; если
убрать слишком мало, следующий разбор увидит мусор.

**Превращать каждый заголовок в `String` «на всякий случай».** Это работает, но возвращает все
выделения памяти. Храните срезы (или `Bytes`) и превращайте во владеющие значения только те немногие,
которые действительно нужно хранить.

## Попробуйте сами

1. В `parse_demo.rs` разрежьте вход на три куска вместо двух, причём разрез сделайте внутри
   `\r\n\r\n`. Ответ остаётся `Partial` до последнего куска.
2. Отправьте строку запроса с двумя пробелами между методом и путём (`GET  / HTTP/1.1`). Какую ошибку
   вы получите и почему?
3. В `nom_demo.rs` замените в импорте `streaming` на `complete` и запустите снова. Второй вход теперь
   печатает `error`, а не `incomplete`.

## Упражнения

### Упражнение 1 — Найти заголовок (легко)

Добавьте в `Request<'a>` метод `pub fn header(&self, name: &str) -> Option<&'a [u8]>`, который
возвращает значение первого заголовка с таким именем. Имена заголовков в HTTP не зависят от регистра,
поэтому `host` должен находить `Host`. Напишите модульный тест.

<details>
<summary>Подсказка</summary>

`str::eq_ignore_ascii_case` сравнивает две строки без учёта регистра ASCII. Обратите внимание на тип
результата: значение живёт столько же, сколько *буфер* (`'a`), а не только сколько `&self`.

</details>

<details>
<summary>Ответ</summary>

Поместите это в `src/parse.rs`:

```rust
impl<'a> Request<'a> {
    /// The value of the first header called `name`, ignoring ASCII case.
    pub fn header(&self, name: &str) -> Option<&'a [u8]> {
        self.headers
            .iter()
            .find(|(n, _)| n.eq_ignore_ascii_case(name))
            .map(|&(_, value)| value)
    }
}

#[cfg(test)]
mod header_tests {
    use super::*;

    #[test]
    fn header_lookup_ignores_case() {
        let buf = b"GET / HTTP/1.1\r\nHost: a.example\r\nX-Id: 7\r\n\r\n";
        let Ok(Parsed::Complete { request, .. }) = parse_request(buf) else {
            panic!("expected a complete request");
        };
        assert_eq!(request.header("host"), Some(&b"a.example"[..]));
        assert_eq!(request.header("X-ID"), Some(&b"7"[..]));
        assert_eq!(request.header("Accept"), None);
    }
}
```

```text
test parse::header_tests::header_lookup_ignores_case ... ok
```

Почему это работает: `iter()` даёт ссылки на пары `(name, value)`; `.map(|&(_, value)| value)`
копирует наружу срез (ссылка на срез реализует `Copy`), который по-прежнему указывает в буфер, поэтому
его можно вернуть со временем жизни `'a`.

</details>

### Упражнение 2 — Свойства сетевого парсера (средне)

Напишите интеграционный тест `tests/never_panics.rs` с `proptest`, который проверяет два свойства:
`parse_request` никогда не паникует для любого вектора байтов длиной до 2000 байтов, и каждый строгий
префикс правильного запроса — `Partial`. Добавьте обычный тест, что весь правильный запрос — `Complete`
с правильной длиной.

<details>
<summary>Подсказка</summary>

`proptest::collection::vec(any::<u8>(), 0..2000)` генерирует векторы байтов. Для префиксов
генерируйте позицию разреза `cut in 0..VALID.len()` и разбирайте `&VALID[..cut]`. Внутри `proptest!`
используйте `prop_assert_eq!`.

</details>

<details>
<summary>Ответ</summary>

```rust
use p22_net::parse::{Parsed, parse_request};
use proptest::prelude::*;

const VALID: &[u8] = b"POST /api/items HTTP/1.1\r\nHost: shop\r\nContent-Length: 0\r\n\r\n";

proptest! {
    #[test]
    fn never_panics_on_any_bytes(input in proptest::collection::vec(any::<u8>(), 0..2000)) {
        let _ = parse_request(&input);
    }

    #[test]
    fn every_strict_prefix_is_partial(cut in 0..VALID.len()) {
        prop_assert_eq!(parse_request(&VALID[..cut]), Ok(Parsed::Partial));
    }
}

#[test]
fn the_whole_request_is_complete() {
    let parsed = parse_request(VALID).unwrap();
    assert!(matches!(parsed, Parsed::Complete { len, .. } if len == VALID.len()));
}
```

```text
test the_whole_request_is_complete ... ok
test every_strict_prefix_is_partial ... ok
test never_panics_on_any_bytes ... ok
```

Почему это работает: proptest по умолчанию запускает каждое свойство на 256 случайных входах. Паника
(например, выход индекса за границы среза) провалила бы первый тест и была бы сжата до маленького
примера. Второе свойство ловит парсеры, которые слишком рано отвечают «ошибка» или «полный».

</details>

### Упражнение 3 — Чтение заголовочных частей из потока (сложно)

Напишите `src/read.rs` с
`pub async fn read_head<R: AsyncRead + Unpin>(reader: &mut R, buf: &mut BytesMut) -> Result<Option<Bytes>, ReadError>`.
Функция читает, пока в `buf` не окажется полная заголовочная часть, затем отрезает её через `split_to`
и возвращает как `Bytes`. Она возвращает `Ok(None)`, если собеседник закрыл соединение, ничего не
отправив, и ошибку, если он закрыл его посреди заголовочной части. Протестируйте её с
`tokio::io::duplex`, отправляя два запроса конвейером по одному байту.

<details>
<summary>Подсказка</summary>

Цикл: сначала попробуйте разобрать то, что уже лежит в `buf` (предыдущее чтение могло принести два
запроса); только если результат `Partial`, вызывайте `reader.read_buf(buf).await` из `AsyncReadExt`.
`read_buf` возвращает `Ok(0)` в конце файла. `tokio::io::duplex(64)` даёт два соединённых потока в
памяти.

</details>

<details>
<summary>Ответ</summary>

Добавьте `pub mod read;` в `src/lib.rs`, затем:

```rust
use crate::parse::{ParseError, Parsed, parse_request};
use bytes::{Bytes, BytesMut};
use tokio::io::{AsyncRead, AsyncReadExt};

#[derive(Debug)]
pub enum ReadError {
    Io(std::io::Error),
    Parse(ParseError),
    /// The peer closed the connection in the middle of a head.
    UnexpectedEof,
}

/// Reads until `buf` holds one whole request head and returns that head.
/// Returns `Ok(None)` if the peer closed the connection before sending anything.
pub async fn read_head<R: AsyncRead + Unpin>(
    reader: &mut R,
    buf: &mut BytesMut,
) -> Result<Option<Bytes>, ReadError> {
    loop {
        match parse_request(buf).map_err(ReadError::Parse)? {
            Parsed::Complete { len, .. } => return Ok(Some(buf.split_to(len).freeze())),
            Parsed::Partial => {}
        }
        let n = reader.read_buf(buf).await.map_err(ReadError::Io)?;
        if n == 0 {
            return if buf.is_empty() {
                Ok(None)
            } else {
                Err(ReadError::UnexpectedEof)
            };
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;
    use tokio::io::AsyncWriteExt;

    #[tokio::test]
    async fn reads_heads_sent_one_byte_at_a_time() {
        let (mut client, mut server) = tokio::io::duplex(64);
        tokio::spawn(async move {
            for byte in b"GET /1 HTTP/1.1\r\n\r\nGET /2 HTTP/1.1\r\n\r\n" {
                client.write_all(&[*byte]).await.unwrap();
            }
            // dropping `client` closes the connection
        });

        let mut buf = BytesMut::new();
        let first = read_head(&mut server, &mut buf).await.unwrap().unwrap();
        let second = read_head(&mut server, &mut buf).await.unwrap().unwrap();
        assert_eq!(&first[..], b"GET /1 HTTP/1.1\r\n\r\n");
        assert_eq!(&second[..], b"GET /2 HTTP/1.1\r\n\r\n");
        assert!(read_head(&mut server, &mut buf).await.unwrap().is_none());
    }

    #[tokio::test]
    async fn a_cut_head_is_an_error() {
        let mut input: &[u8] = b"GET / HTTP/1.1\r\nHost:";
        let mut buf = BytesMut::new();
        let result = read_head(&mut input, &mut buf).await;
        assert!(matches!(result, Err(ReadError::UnexpectedEof)));
    }
}
```

```text
test read::tests::a_cut_head_is_an_error ... ok
test read::tests::reads_heads_sent_one_byte_at_a_time ... ok
```

Почему это работает: разбор *до* чтения важен для конвейерной передачи: после того как первая
заголовочная часть отрезана, вторая уже может лежать в `buf`, и ожидание чтения привело бы к
зависанию. `read_buf` дописывает в `BytesMut` и увеличивает его по необходимости. `&[u8]` реализует
`AsyncRead`, поэтому вход второго теста умещается в одну строку.

</details>

## Вопросы для самопроверки

1. Что возвращает парсер без копирования вместо значений `String`?

<details>
<summary>Ответ</summary>

Заимствованные срезы (`&str`, `&[u8]`), которые указывают внутрь входного буфера, или дескрипторы
`Bytes`, которые разделяют память буфера.

</details>

2. Почему разбор без копирования не может привести к «срезам, показывающим не те байты» в безопасном
   Rust?

<details>
<summary>Ответ</summary>

У типа результата есть время жизни, привязанное к буферу. Проверка заимствований отвергает любой код,
который меняет или освобождает буфер, пока срез из него ещё используется (ошибка E0502 в частых
ошибках).

</details>

3. Какие три результата возможны при разборе байтов из сокета и что должен сделать вызывающий код в
   каждом случае?

<details>
<summary>Ответ</summary>

Полный (использовать сообщение и убрать его `len` байтов из буфера), неполный (прочитать ещё байты и
попробовать снова), неверный (ответить ошибкой, например `400 Bad Request`, и закрыть соединение).

</details>

4. Что делает `BytesMut::split_to(n)` и копирует ли он?

<details>
<summary>Ответ</summary>

Он отрезает первые `n` байтов в отдельный `BytesMut` и оставляет остаток в исходном. Байты не
копируются; обе части указывают в ту же память.

</details>

5. Почему для сетевого ввода мы импортируем парсеры из `nom::bytes::streaming`, а не из
   `nom::bytes::complete`?

<details>
<summary>Ответ</summary>

Потоковые парсеры сообщают `Incomplete`, когда вход заканчивается рано, что значит «прочитайте ещё».
Парсеры `complete` считают конец входа окончательным и возвращают ошибку, а это неправильно, если ещё
могут прийти байты.

</details>

6. Зачем парсеру предел `MAX_HEAD`?

<details>
<summary>Ответ</summary>

Без предела клиент, который никогда не отправляет пустую строку, заставляет буфер расти бесконечно, и
серверу не хватает памяти. Предел превращает это в аккуратную ошибку `TooLarge`.

</details>

## Итоги

- Разбор без копирования возвращает срезы входа вместо новых строк, что убирает выделения памяти и
  копирование из горячего пути.
- Времена жизни делают это безопасным: проверка заимствований не даёт буферу измениться, пока срез жив.
- У сетевого парсера три исхода: полный (с длиной), неполный, неверный. Неполный — не ошибка.
- Ограничивайте размер того, что вы буферизуете (`MAX_HEAD`); никогда не доверяйте размерам из сети.
- `BytesMut` — буфер для чтения; `split_to(..).freeze()` превращает готовое сообщение в `Bytes`, у
  которого можно брать срезы, клонировать и отправлять в другие задачи без копирования.
- `nom` строит такой же парсер без копирования из маленьких комбинаторов; для сокетов используйте его
  потоковые (`streaming`) парсеры.
- Тестируйте парсеры свойствами («никогда не паникует», «префиксы неполные»), а затем фаззингом.

## Источники и что почитать

- [bytes documentation](https://docs.rs/bytes/1/bytes/) — `Bytes`, `BytesMut`, `split_to`, `freeze` и как работает счётчик ссылок.
- [nom documentation](https://docs.rs/nom/8/nom/) — список комбинаторов и разница между потоковыми и полными (complete) парсерами.
- [httparse documentation](https://docs.rs/httparse/latest/httparse/) — HTTP/1.x-парсер без копирования, который использует hyper; сравните его со своим.
- [RFC 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112) — точная грамматика строки запроса и полей заголовков (RFC, Request for Comments, «запрос комментариев» — официальный документ интернет-стандартов).
- [tokio::io::AsyncReadExt::read_buf](https://docs.rs/tokio/1/tokio/io/trait.AsyncReadExt.html) — чтение из сокета прямо в `BytesMut`.
- [The Rust Programming Language — 10.3 Validating References with Lifetimes](https://doc.rust-lang.org/book/ch10-03-lifetime-syntax.html) — повторение правил времён жизни, которые делают разбор без копирования безопасным.

---

[🏠 Начало](../START-HERE.md) · [← 22.18 Трек сетей: высокопроизводительные серверы](18-networking-track-high-performance-servers.md) · [📋 Этап 22](README.md) · [22.20 Трек сетей: прокси для продакшена →](20-networking-track-production-proxies.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
