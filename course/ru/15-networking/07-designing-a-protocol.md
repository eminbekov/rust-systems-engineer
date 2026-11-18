<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 15.6 Асинхронная работа с сетью в Tokio](06-async-networking-with-tokio.md) · [📋 Этап 15](README.md) · [15.8 HTTP с нуля →](08-http-from-scratch.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/15-networking/07-designing-a-protocol.md) <!-- nav -->

# Урок 15.7 — Проектирование протокола: фрейминг и разбор

> **Этап 15 — Сети** · Урок 7 из 16 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Три способа разрезать поток байтов на сообщения (разделители, префикс длины, фиксированный размер) и когда какой применять.
- Как написать декодер кадров, который обрабатывает три возможных исхода: полный кадр, байтов пока не хватает, недопустимый кадр.
- Как спроектировать маленький двоичный формат сообщений и разбирать его, никогда не паникуя на плохом вводе.
- Как пользоваться `bytes::BytesMut` и каркасом `tokio_util::codec` (`Framed`, `Decoder`, `LengthDelimitedCodec`).
- Как тестировать протокол: круговые проверки, байты, приходящие по одному, и мусор.

## Перед началом

- Вы прошли [Урок 15.6 — Асинхронная работа с сетью в Tokio](06-async-networking-with-tokio.md).
- Вы видели в [Уроке 15.4](04-tcp-sockets-in-rust.md), что записи и чтения TCP не соответствуют друг другу один к одному.
- Вы знаете порядок байтов из Урока 15.1, перечисления и `match` из Этапа 04, а проверку ввода — из [Урока 10.4 — Проверка ввода: никогда не доверяйте вводу](../10-security-fundamentals/04-input-validation.md).
- Добавьте в `code/p15-net` три крейта (версии с crates.io, проверено 2026-11-17):

```bash
cargo add bytes@1
cargo add tokio-util@0.7 --features codec
cargo add futures@0.3
```

## Новые слова

| Слово | Значение |
|---|---|
| **фрейминг** (framing) | Определение того, где в потоке байтов заканчивается одно сообщение и начинается следующее. |
| **кадр** (frame) | Одно полное сообщение в том виде, в каком оно идёт по сети, вместе с оформлением (например, префиксом длины). Не путайте с кадром Ethernet из Урока 15.1. |
| **разделитель** (delimiter) | Особый байт или последовательность байтов, отмечающая конец сообщения, например `\n`. |
| **префикс длины** (length prefix) | Число перед сообщением, которое говорит, сколько байтов идёт следом. |
| **полезная нагрузка** (payload) | Содержимое кадра без байтов оформления. |
| **формат на проводе** (wire format) | Точные байты, которые отправляет протокол: какое поле идёт первым, какого оно размера, какой порядок байтов. |
| **декодер / кодер** (decoder / encoder) | Код, который превращает байты в сообщения / сообщения в байты. Вместе они — **кодек** (codec). |
| **круговая проверка** (round trip) | Закодировать значение и снова раскодировать его. Правильный кодек возвращает в точности исходное значение. |
| **`BytesMut` / `Bytes`** | Буферы байтов из крейта `bytes`: `BytesMut` может расти и дёшево резаться на части; `Bytes` — неизменяемый кусок, который дёшево разделять. |
| **`Framed`** | Тип из `tokio_util`, который оборачивает сокет и кодек и превращает поток байтов в *поток сообщений* и *приёмник* сообщений. |
| **приёмник** (sink) | Противоположность потока (stream): то, *во что* можно отправлять значения одно за другим. |
| **TCP, HTTP, TLS** (Transmission Control Protocol, HyperText Transfer Protocol, Transport Layer Security — протокол управления передачей, протокол передачи гипертекста, протокол защиты транспортного уровня) | Транспорт потока байтов из Урока 15.3, протокол веба (Урок 15.8) и слой шифрования (Урок 15.12). |
| **SMTP** (Simple Mail Transfer Protocol, простой протокол передачи почты) | Текстовый протокол, которым почтовые серверы передают письма друг другу. |
| **KV** (key-value, «ключ-значение») | Название нашего учебного протокола: он хранит значения под ключами, как `HashMap` на сервере. |
| **UTF-8, ASCII, ГБ** | Кодировка строк Rust (Unicode Transformation Format, 8-bit — формат преобразования Юникода); более старая кодировка из 128 символов, которая составляет её первую часть (American Standard Code for Information Interchange, американский стандартный код для обмена информацией, Урок 12.5); гигабайт (GB), около миллиарда байтов. |

## 1. Три способа разбить поток на кадры

TCP доставляет байты, а не сообщения (Урок 15.3). Поэтому каждый протокол поверх TCP обязан ответить
на один вопрос: **где заканчивается сообщение?** Распространены три ответа:

```text
 1. delimiter            G E T ␠ k e y \n S E T ␠ k ␠ v \n          "the message ends at \n"
 2. length prefix        00 00 00 05 h e l l o 00 00 00 02 h i      "the next 5 bytes are the message"
 3. fixed size           [16 bytes][16 bytes][16 bytes]             "every message is 16 bytes"
```

| Способ | Хорош для | Осторожно |
|---|---|---|
| **Разделитель** | текстовых протоколов, которые люди набирают и читают (заголовки HTTP, SMTP, команды Redis, наш чат) | разделитель не должен встречаться внутри сообщения или должен экранироваться; чтобы найти его, нужно просмотреть каждый байт |
| **Префикс длины** | двоичных данных, файлов, всего, что может содержать любой байт (записи TLS, кадры HTTP/2, Kafka, PostgreSQL) | длина приходит от другой стороны: ограничивайте её, иначе один кадр может заявить 4 ГБ |
| **Фиксированный размер** | совсем простых протоколов датчиков и оборудования | тратит место; не может нести сообщения разного размера |

Настоящие протоколы часто их сочетают: запрос HTTP/1.1 состоит из заголовка на разделителях (строки,
оканчивающиеся на `\r\n`, и пустая строка в конце), за которым идёт тело, чей размер задан заголовком
длины (`Content-Length`). Именно это вы будете разбирать в Уроке 15.8.

## 2. Декодер кадров вручную

Сначала напишем фрейминг с префиксом длины сами, чтобы никакая библиотека не скрывала идею.
Получатель собирает приходящие байты в буфер. После каждого чтения он спрашивает: «есть ли в начале
полный кадр?» Возможных ответов ровно **три**:

1. **Пока нет**: меньше 4 байтов, или длина говорит 100 байтов, а пришло только 60. Сохраните байты и
   читайте дальше. *Ничего не потребляйте.*
2. **Да**: отрежьте кадр от начала буфера и верните его. В том же буфере могут быть следующие кадры.
3. **Недопустимо**: кадр нарушает правило (здесь: он слишком велик). Верните ошибку и закройте
   соединение.

Крейт `bytes` даёт для этого хороший буфер. `BytesMut` — растущий буфер байтов; `split_to(n)`
отрезает первые `n` байтов в отдельный буфер без копирования, а `freeze()` превращает его в
неизменяемый `Bytes`. Трейты `Buf` (чтение: `advance`, `get_u16`, …) и `BufMut` (запись: `put_u32`,
`put_slice`, …) читают и пишут числа в порядке big-endian. Сохраните как `src/bin/frames.rs`:

```rust
use bytes::{Buf, BufMut, Bytes, BytesMut};

const MAX_FRAME: usize = 64 * 1024;

#[derive(Debug, PartialEq)]
enum FrameError {
    TooLarge(usize),
}

/// Adds a 4-byte big-endian length in front of the payload.
fn encode_frame(payload: &[u8], out: &mut BytesMut) {
    out.put_u32(payload.len() as u32);
    out.put_slice(payload);
}

/// Takes one complete frame from the front of `buf`, if there is one.
fn decode_frame(buf: &mut BytesMut) -> Result<Option<Bytes>, FrameError> {
    if buf.len() < 4 {
        return Ok(None); // нет даже длины
    }
    let len = u32::from_be_bytes([buf[0], buf[1], buf[2], buf[3]]) as usize;
    if len > MAX_FRAME {
        return Err(FrameError::TooLarge(len)); // проверяем ДО ожидания данных
    }
    if buf.len() < 4 + len {
        buf.reserve(4 + len - buf.len()); // освобождаем место для остатка
        return Ok(None);
    }
    buf.advance(4); // отбрасываем длину
    Ok(Some(buf.split_to(len).freeze())) // отрезаем полезную нагрузку спереди
}

fn main() {
    let mut wire = BytesMut::new();
    encode_frame(b"hello", &mut wire);
    encode_frame(b"world!", &mut wire);
    println!("on the wire: {:02x?}", &wire[..]);

    // Получатель получает байты кусками по 3, как их может доставить TCP.
    let mut inbox = BytesMut::new();
    for piece in wire.chunks(3) {
        inbox.extend_from_slice(piece);
        while let Some(frame) = decode_frame(&mut inbox).unwrap() {
            println!("frame: {:?}", String::from_utf8_lossy(&frame));
        }
    }

    let mut evil = BytesMut::from(&[0xff, 0xff, 0xff, 0xff][..]);
    println!("{:?}", decode_frame(&mut evil));
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn waits_for_a_complete_frame() {
        let mut buf = BytesMut::from(&[0, 0, 0, 5, b'h', b'e'][..]);
        assert_eq!(decode_frame(&mut buf), Ok(None));
        assert_eq!(buf.len(), 6, "nothing may be consumed yet");
        buf.extend_from_slice(b"llo");
        assert_eq!(
            decode_frame(&mut buf),
            Ok(Some(Bytes::from_static(b"hello")))
        );
        assert!(buf.is_empty());
    }

    #[test]
    fn two_frames_in_one_read() {
        let mut buf = BytesMut::new();
        encode_frame(b"a", &mut buf);
        encode_frame(b"bc", &mut buf);
        assert_eq!(decode_frame(&mut buf).unwrap().unwrap(), "a");
        assert_eq!(decode_frame(&mut buf).unwrap().unwrap(), "bc");
        assert_eq!(decode_frame(&mut buf), Ok(None));
    }

    #[test]
    fn rejects_huge_lengths() {
        let mut buf = BytesMut::from(&[0x7f, 0, 0, 0][..]);
        assert_eq!(
            decode_frame(&mut buf),
            Err(FrameError::TooLarge(0x7f00_0000))
        );
    }
}
```

`cargo run --bin frames`:

```text
on the wire: [00, 00, 00, 05, 68, 65, 6c, 6c, 6f, 00, 00, 00, 06, 77, 6f, 72, 6c, 64, 21]
frame: "hello"
frame: "world!"
Err(TooLarge(4294967295))
```

**Что происходит:**

1. `encode_frame` записывает длину через `put_u32` (big-endian), а затем полезную нагрузку:
   `00 00 00 05` и `hello`.
2. В `main` получатель получает 19 байтов кусками по 3 — худший случай из Урока 15.4. После каждого
   куска внутренний цикл `while let` достаёт все полные кадры. Большинство кусков не дают ничего; два
   из них дополняют кадр.
3. Порядок проверок в `decode_frame` важен. Предел размера проверяется **сразу, как только известна
   длина**, до ожидания данных. Иначе клиент мог бы отправить `ff ff ff ff` и заставить сервер ждать
   (и накапливать) 4 ГБ.
4. Тесты покрывают три исхода. Первый тест ещё и проверяет, что «пока нет» ничего не потребило:
   декодер, который съедает часть кадра, а потом ждёт, теряет данные.

`cargo test --bin frames` сообщает `3 passed`.

## 3. Проектируем формат сообщений

Фрейминг говорит, где кончается сообщение. Полезной нагрузке всё ещё нужен формат. Спроектируем
крошечный протокол «ключ — значение», «KV». Клиент отправляет запросы `Get` и `Set`; сервер отвечает
`Value`, `NotFound` или `Ok`. Каждая полезная нагрузка начинается с одного **байта типа**; строки
записываются как 2-байтовая длина плюс байты UTF-8:

```text
 frame:    [ length: u32 ][ payload ......................................... ]
 payload:  [ type: u8 ][ fields ... ]

 type 1  Get       [01][key len: u16][key bytes]
 type 2  Set       [02][key len: u16][key bytes][value len: u16][value bytes]
 type 3  Value     [03][value len: u16][value bytes]
 type 4  NotFound  [04]
 type 5  Ok        [05]
```

Вопросы, на которые нужно ответить для каждого протокола, который вы проектируете:

- **Порядок байтов** каждого числа: здесь везде big-endian.
- **Пределы**: кадры не больше 64 КиБ, строки не больше 65 535 байтов (самое большое `u16`).
- **Что считается ошибкой**: неизвестный байт типа, строка короче своей длины, байты не в UTF-8, лишние
  байты после последнего поля. Каждый такой случай должен быть ошибкой, а не паникой.
- **Версии**: как новый клиент будет разговаривать со старым сервером? Простой способ — сначала
  отправлять номер версии и считать неизвестные байты типа ошибкой. (Наш протокол — версия 1, и этот
  шаг мы пропускаем.)

> 📏 **Соглашение:** запишите формат на проводе таблицей или схемой, как выше, *до того*, как писать
> код. Это контракт между клиентом и сервером, и другие люди (или другой язык) должны суметь
> реализовать его только по описанию.

## 4. Разбираем, а фрейминг поручаем `tokio_util`

Теперь вы знаете, как работает фрейминг. В настоящем коде его не переписывают: в `tokio_util::codec`
есть готовый кодек с префиксом длины, `LengthDelimitedCodec`, ровно с нашим форматом (4-байтовая
длина в big-endian) и пределом размера. `Framed` оборачивает сокет вместе с кодеком. Получается
**поток** кадров (читайте их через `.next().await` из `futures::StreamExt`) и **приёмник** (отправляйте
кадры через `.send(...).await` из `futures::SinkExt`). Мы пишем только часть с полезной нагрузкой.
Сохраните как `src/bin/kv.rs` (это один файл; здесь он показан в двух частях). Часть 1, сообщения:

```rust
use bytes::{Buf, BufMut, Bytes, BytesMut};
use futures::{SinkExt, StreamExt};
use std::collections::HashMap;
use std::io;
use tokio::net::{TcpListener, TcpStream};
use tokio_util::codec::{Framed, LengthDelimitedCodec};

#[derive(Debug, Clone, PartialEq)]
enum Message {
    Get { key: String },
    Set { key: String, value: String },
    Value(String),
    NotFound,
    Ok,
}

fn bad(why: &str) -> io::Error {
    io::Error::new(io::ErrorKind::InvalidData, why.to_string())
}

fn put_str(out: &mut BytesMut, s: &str) {
    out.put_u16(s.len() as u16);
    out.put_slice(s.as_bytes());
}

fn get_str(p: &mut &[u8]) -> io::Result<String> {
    if p.remaining() < 2 {
        return Err(bad("string length missing"));
    }
    let len = p.get_u16() as usize;
    if p.remaining() < len {
        return Err(bad("string shorter than its length"));
    }
    let text = String::from_utf8(p[..len].to_vec()).map_err(|_| bad("string is not UTF-8"))?;
    p.advance(len);
    Ok(text)
}

impl Message {
    /// Payload layout: one type byte, then the fields as length-prefixed strings.
    fn encode(&self) -> Bytes {
        let mut out = BytesMut::new();
        match self {
            Message::Get { key } => {
                out.put_u8(1);
                put_str(&mut out, key);
            }
            Message::Set { key, value } => {
                out.put_u8(2);
                put_str(&mut out, key);
                put_str(&mut out, value);
            }
            Message::Value(v) => {
                out.put_u8(3);
                put_str(&mut out, v);
            }
            Message::NotFound => out.put_u8(4),
            Message::Ok => out.put_u8(5),
        }
        out.freeze()
    }

    fn parse(mut p: &[u8]) -> io::Result<Message> {
        if p.is_empty() {
            return Err(bad("empty payload"));
        }
        let msg = match p.get_u8() {
            1 => Message::Get {
                key: get_str(&mut p)?,
            },
            2 => Message::Set {
                key: get_str(&mut p)?,
                value: get_str(&mut p)?,
            },
            3 => Message::Value(get_str(&mut p)?),
            4 => Message::NotFound,
            5 => Message::Ok,
            other => return Err(bad(&format!("unknown message type {other}"))),
        };
        if p.has_remaining() {
            return Err(bad("extra bytes after the message"));
        }
        Ok(msg)
    }
}
```

**Что происходит:**

1. `Message` — это протокол в виде перечисления Rust: по варианту на каждый байт типа. `encode` и
   `parse` — единственные места, которые знают формат на проводе.
2. `parse` читает из `p: &[u8]` через трейт `Buf`, который реализован для срезов байтов: `get_u8` и
   `get_u16` читают число **и сдвигают срез вперёд**. Поэтому `get_str` принимает `&mut &[u8]` —
   изменяемую ссылку на срез: она укорачивает срез вызывающего кода.
3. Перед каждым чтением код проверяет `remaining()`. `get_u16` на меньше чем 2 байтах вызвал бы панику,
   поэтому проверка превращает короткую злонамеренную полезную нагрузку в ошибку.
4. `String::from_utf8` отклоняет байты, которые не являются корректным UTF-8 (Урок 12.5). Последняя
   проверка отклоняет лишние байты: строгий парсер оставляет меньше места для расхождений между
   реализациями.

Часть 2 — сервер, клиент и тесты (тот же файл, ниже части 1):

```rust
/// Frames: a 4-byte big-endian length, then the payload; at most 64 KiB.
fn framed(stream: TcpStream) -> Framed<TcpStream, LengthDelimitedCodec> {
    let codec = LengthDelimitedCodec::builder()
        .max_frame_length(64 * 1024)
        .new_codec();
    Framed::new(stream, codec)
}

async fn serve_one(stream: TcpStream) -> io::Result<()> {
    let mut store: HashMap<String, String> = HashMap::new();
    let mut frames = framed(stream);
    while let Some(frame) = frames.next().await {
        let answer = match Message::parse(&frame?)? {
            Message::Get { key } => match store.get(&key) {
                Some(v) => Message::Value(v.clone()),
                None => Message::NotFound,
            },
            Message::Set { key, value } => {
                store.insert(key, value);
                Message::Ok
            }
            _ => return Err(bad("clients may only send Get and Set")),
        };
        frames.send(answer.encode()).await?;
    }
    Ok(())
}

#[tokio::main]
async fn main() -> io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:0").await?;
    let addr = listener.local_addr()?;
    tokio::spawn(async move {
        let (stream, _) = listener.accept().await?;
        serve_one(stream).await
    });

    let mut client = framed(TcpStream::connect(addr).await?);
    let lang = || "lang".to_string();
    for request in [
        Message::Get { key: lang() },
        Message::Set {
            key: lang(),
            value: "Rust".into(),
        },
        Message::Get { key: lang() },
    ] {
        println!("-> {request:?}  {:02x?}", &request.encode()[..]);
        client.send(request.encode()).await?;
        let frame = client.next().await.expect("server closed")?;
        println!("<- {:?}", Message::parse(&frame)?);
    }
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn every_message_survives_a_round_trip() {
        let messages = [
            Message::Get { key: "k".into() },
            Message::Set {
                key: "".into(),
                value: "ünïcödé".into(),
            },
            Message::Value("v".into()),
            Message::NotFound,
            Message::Ok,
        ];
        for m in messages {
            assert_eq!(Message::parse(&m.encode()).unwrap(), m);
        }
    }

    #[test]
    fn garbage_is_an_error_not_a_panic() {
        for payload in [&[][..], &[9], &[1, 0], &[1, 0, 5, b'a'], &[5, 0]] {
            assert!(Message::parse(payload).is_err(), "{payload:?} was accepted");
        }
    }
}
```

`cargo run --bin kv` (шестнадцатеричные байты — это полезные нагрузки; кодек добавляет перед ними
4-байтовую длину):

```text
-> Get { key: "lang" }  [01, 00, 04, 6c, 61, 6e, 67]
<- NotFound
-> Set { key: "lang", value: "Rust" }  [02, 00, 04, 6c, 61, 6e, 67, 00, 04, 52, 75, 73, 74]
<- Ok
-> Get { key: "lang" }  [01, 00, 04, 6c, 61, 6e, 67]
<- Value("Rust")
```

**Что происходит:**

1. `framed` создаёт кодек с пределом 64 КиБ. Собеседник, который объявит кадр больше, получит от
   кодека ошибку `frame size too big`, и соединение закончится.
2. В `serve_one` вызов `frames.next().await` возвращает `Some(Ok(frame))` для каждого полного кадра,
   `Some(Err(..))` для ошибки фрейминга и `None` при EOF. Вся буферизация и логика «пока нет» из
   раздела 2 происходят внутри кодека.
3. В `Message::parse(&frame?)?` два `?`: первый для ошибок фрейминга, второй для ошибок полезной
   нагрузки. Оба завершают соединение с ошибкой. При ошибке протокола закрыть соединение безопаснее,
   чем гадать.
4. `send` принимает `Bytes`, добавляет префикс длины и записывает.
5. `cargo test --bin kv` запускает круговую проверку (каждый вариант, пустой ключ, текст не в ASCII) и
   тест мусора (пусто, неизвестный тип, слишком короткая длина, строка короче своей длины, лишние
   байты). Он сообщает `2 passed`.

Если вам нужен собственный кодек вместо `LengthDelimitedCodec`, реализуйте трейт `Decoder` (один
метод, `decode(&mut self, buf: &mut BytesMut) -> Result<Option<Item>, Error>`, ровно с тремя исходами
из раздела 2) и `Encoder`. Упражнение 3 делает это для строк.

> 🔒 **Безопасность:** парсер — первый код, который касается байтов атакующего. Сверяйте каждую длину с
> оставшимся вводом *и* с фиксированным пределом, никогда не обращайтесь по индексу без проверки и
> считайте всё неожиданное ошибкой, которая закрывает соединение. Фаззинг (Урок 09.8) — лучший
> инструмент, чтобы найти забытые случаи: `Message::parse(&[u8])` — идеальная цель для фаззинга.

> 🧪 **Тестирование:** круговая проверка естественно ложится на тестирование на основе свойств
> (Урок 09.4): «для любого `Message` m верно `parse(encode(m)) == m`». Добавьте ещё один тест, который
> подаёт декодеру правильный кадр по одному байту; он ловит декодеры, которые потребляют байты слишком
> рано.

## Частые ошибки

> ⚠️ **Частая ошибка: проверять длину после чтения данных.** Декодер, который сначала ждёт `len`
> байтов, а потом проверяет `len <= MAX`, позволяет клиенту заставить его накопить гигабайты.
> Проверяйте предел в тот момент, когда длина стала известна.

> ⚠️ **Частая ошибка: приведения `as`, которые обрезают числа.** `s.len() as u16` молча оставляет только
> младшие 16 бит: строка в 70 000 байтов объявляется как 4464 байта, и получатель неверно читает всё,
> что идёт после неё. Здесь её по счастливой случайности ловит предел кадра в 64 КиБ. Лучше используйте
> `u16::try_from(s.len())` и возвращайте ошибку.

> ⚠️ **Частая ошибка: обращаться по индексу до проверки.** `p[0]` или `get_u16()` на пустом срезе
> вызывает панику, а паника в задаче соединения — это удалённый способ её убить (а с
> `panic = "abort"` — и весь сервер). Каждому чтению ввода нужна предварительная проверка длины.

## Попробуйте сами

1. В `frames.rs` замените `wire.chunks(3)` на `wire.chunks(1)`, а затем на `wire.chunks(100)`.
   Напечатанные кадры не меняются; меняется только то, как часто вызывается декодер.
2. Уберите проверку `if len > MAX_FRAME` и запустите тесты: `rejects_huge_lengths` падает. Верните её.
3. В `kv.rs` отправьте с клиента `Message::Ok`. Сервер отвечает закрытием соединения, а `next()` у
   клиента возвращает `None`, поэтому он паникует с `server closed`.

## Упражнения

### Упражнение 1 — Закодируйте вручную (легко)

Запишите байт за байтом в шестнадцатеричном виде полный кадр (вместе с префиксом длины), который
клиент отправляет для `Message::Set { key: "a".into(), value: "hi".into() }`.

<details>
<summary>Подсказка</summary>

Сначала запишите полезную нагрузку: байт типа, затем для каждой строки её длину двумя байтами и сами
байты (`a` — это `61`, `h` — `68`, `i` — `69`). Посчитайте байты полезной нагрузки; кадр начинается с
этого числа в 4 байтах.

</details>

<details>
<summary>Ответ</summary>

```text
00 00 00 08   02   00 01 61   00 02 68 69
 length=8    Set   key "a"    value "hi"
```

Полезная нагрузка — 1 + (2 + 1) + (2 + 2) = 8 байтов. Если закодировать сообщение через `encode()` и
`LengthDelimitedCodec` и напечатать байты, получится ровно
`[00, 00, 00, 08, 02, 00, 01, 61, 00, 02, 68, 69]`.

</details>

### Упражнение 2 — Сообщение `Delete` с тестами (средне)

Расширьте протокол сообщением `Delete { key }` (байт типа 6). Сервер удаляет ключ и отвечает `Ok` или
`NotFound`, если ключа не было. Обновите `encode`, `parse` и `serve_one` и напишите тест: круговую
проверку для `Delete` плюс проверку, что байт типа 6 без ключа — это ошибка. Затем измените `main`,
чтобы он отправлял `Set`, `Delete`, `Delete` и `Get`, и проверьте, что ответы — `Ok`, `Ok`,
`NotFound`, `NotFound`.

<details>
<summary>Подсказка</summary>

Компилятор вам поможет: после добавления варианта `encode` не скомпилируется, пока вы его не
обработаете (`match` должен быть исчерпывающим). В `parse` и `serve_one` есть ветки `_`, так что там
придётся помнить самим. `HashMap::remove` возвращает `Option<V>`.

</details>

<details>
<summary>Ответ</summary>

Добавьте в `Message` вариант `Delete { key: String },`. В `encode` добавьте ветку:

```rust
            Message::Delete { key } => {
                out.put_u8(6);
                put_str(&mut out, key);
            }
```

В `parse` добавьте ветку `6 => Message::Delete { key: get_str(&mut p)? },` (rustfmt растянет её на три
строки). В `serve_one` добавьте перед веткой `_`:

```rust
            Message::Delete { key } => match store.remove(&key) {
                Some(_) => Message::Ok,
                None => Message::NotFound,
            },
```

Добавьте этот тест в `mod tests`:

```rust
    #[test]
    fn delete_round_trip_and_bad_delete() {
        let msg = Message::Delete { key: "lang".into() };
        assert_eq!(Message::parse(&msg.encode()).unwrap(), msg);
        assert!(Message::parse(&[6]).is_err(), "Delete without a key");
    }
```

`cargo test --bin kv` сообщает `3 passed`, а изменённый `main` печатает четыре ожидаемых ответа.

Почему это работает: протокол живёт в одном перечислении с одним `encode` и одним `parse`, поэтому
добавление сообщения — локальное изменение, а круговая проверка сразу убеждается, что обе стороны
согласны насчёт новых байтов.

</details>

### Упражнение 3 — Кодек строк с пределом (сложно)

Чат-сервер из Урока 15.6 принимает строки любой длины. Напишите `src/bin/lines_codec.rs` со структурой
`struct MaxLineCodec { max: usize }`, которая реализует `tokio_util::codec::Decoder` с
`Item = String`. Она возвращает строки без `\n`, ждёт, если `\n` ещё нет, и возвращает ошибку, как
только строка оказывается длиннее `max` байтов, **ещё до прихода её перевода строки**. Протестируйте
её без всякой сети, вызывая `decode` прямо на `BytesMut`.

<details>
<summary>Подсказка</summary>

`buf.iter().position(|&b| b == b'\n')` находит перевод строки. Если его нет и `buf.len() > max`, строка
может стать только длиннее: это и есть ранняя ошибка. `split_to(i)` забирает строку, `advance(1)`
отбрасывает `\n`.

</details>

<details>
<summary>Ответ</summary>

```rust
use bytes::{Buf, BytesMut};
use std::io;
use tokio_util::codec::Decoder;

/// Splits a byte stream into lines, and refuses lines longer than `max` bytes.
struct MaxLineCodec {
    max: usize,
}

impl Decoder for MaxLineCodec {
    type Item = String;
    type Error = io::Error;

    fn decode(&mut self, buf: &mut BytesMut) -> io::Result<Option<String>> {
        match buf.iter().position(|&b| b == b'\n') {
            Some(i) if i > self.max => Err(too_long(self.max)),
            Some(i) => {
                let line = buf.split_to(i); // строка без '\n'
                buf.advance(1); // отбрасываем '\n'
                String::from_utf8(line.to_vec())
                    .map(Some)
                    .map_err(|_| io::Error::new(io::ErrorKind::InvalidData, "not UTF-8"))
            }
            None if buf.len() > self.max => Err(too_long(self.max)),
            None => Ok(None), // перевода строки ещё нет: ждём байтов
        }
    }
}

fn too_long(max: usize) -> io::Error {
    io::Error::new(
        io::ErrorKind::InvalidData,
        format!("line longer than {max} bytes"),
    )
}

fn main() {
    let mut codec = MaxLineCodec { max: 10 };
    let mut buf = BytesMut::from("hello\nworld\nthis line is far too long\n");
    loop {
        match codec.decode(&mut buf) {
            Ok(Some(line)) => println!("line: {line:?}"),
            Ok(None) => break,
            Err(e) => {
                println!("error: {e}");
                break;
            }
        }
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn lines_split_across_reads() {
        let mut codec = MaxLineCodec { max: 100 };
        let mut buf = BytesMut::from("he");
        assert_eq!(codec.decode(&mut buf).unwrap(), None);
        buf.extend_from_slice(b"llo\nwor");
        assert_eq!(codec.decode(&mut buf).unwrap().as_deref(), Some("hello"));
        assert_eq!(codec.decode(&mut buf).unwrap(), None);
        assert_eq!(&buf[..], b"wor");
    }

    #[test]
    fn a_long_line_is_refused_before_its_newline_arrives() {
        let mut codec = MaxLineCodec { max: 4 };
        let mut buf = BytesMut::from("12345"); // '\n' ещё нет, но строка уже слишком длинная
        assert!(codec.decode(&mut buf).is_err());
    }

    #[test]
    fn exactly_max_is_fine() {
        let mut codec = MaxLineCodec { max: 4 };
        let mut buf = BytesMut::from("1234\n");
        assert_eq!(codec.decode(&mut buf).unwrap().as_deref(), Some("1234"));
    }
}
```

```text
line: "hello"
line: "world"
error: line longer than 10 bytes
```

`cargo test --bin lines_codec` сообщает `3 passed`. С `Framed::new(stream, MaxLineCodec { max: 1024 })`
чат-сервер был бы защищён. (В `tokio_util` есть и готовый `LinesCodec::new_with_max_length`; теперь вы
знаете, что он делает внутри.)

Почему это работает: декодер оставляет случай «пока нет» без побочных эффектов и сдаётся, как только
правило нарушено, поэтому память на одно соединение ограничена величиной `max` плюс одно чтение.

</details>

## Вопросы для самопроверки

1. Назовите три способа разбить поток на кадры и по одному протоколу для каждого.

<details>
<summary>Ответ</summary>

Разделитель (строки заголовков HTTP и команды Redis оканчиваются на `\r\n`), префикс длины (записи
TLS, кадры HTTP/2, наш протокол KV), фиксированный размер (простые аппаратные протоколы).

</details>

2. Какие три результата может дать вызов `decode` и что декодер должен сделать с буфером в каждом
   случае?

<details>
<summary>Ответ</summary>

Полный кадр: убрать из буфера ровно этот кадр и вернуть его. Байтов пока не хватает: ничего не менять
(разве что зарезервировать место) и вернуть `Ok(None)`. Недопустимо: вернуть ошибку.

</details>

3. Почему предел размера кадра нужно проверять сразу, как только пришёл префикс длины?

<details>
<summary>Ответ</summary>

Иначе декодер ждёт и накапливает столько байтов, сколько заявил собеседник, — до 4 ГБ для длины
`u32`, и один клиент может исчерпать память сервера.

</details>

4. Почему `get_str` принимает `p: &mut &[u8]`, а не `p: &[u8]`?

<details>
<summary>Ответ</summary>

Она должна сдвинуть срез вызывающего кода за прочитанную строку, чтобы следующее поле читалось с
правильного места. `&mut` на срез позволяет заменить срез вызывающего кода более коротким.

</details>

## Итоги

- TCP несёт байты; **фрейминг** решает, где кончаются сообщения: разделитель, префикс длины или
  фиксированный размер.
- У декодера три исхода: полный кадр (потребить его), пока нет (ничего не потреблять), недопустимо
  (ошибка).
- Проверяйте пределы размера в момент, когда длина стала известна; сверяйте каждую длину с оставшимся
  вводом.
- Сначала запишите **формат на проводе**: порядок полей, размеры, порядок байтов, пределы, ошибки,
  версии.
- `bytes::BytesMut` с `Buf`/`BufMut` читает и пишет поля в big-endian и дёшево делит буферы;
  `tokio_util::codec` даёт `Framed`, `Decoder`/`Encoder`, `LengthDelimitedCodec` и `LinesCodec`.
- Тестируйте круговыми проверками, побайтовым вводом и мусором; парсеры — идеальные цели для
  фаззинга.

## Источники и что почитать

- [Tokio tutorial — Framing](https://tokio.rs/tokio/tutorial/framing) — официальное руководство о буферах и кадрах на примере протокола в стиле Redis.
- [`tokio_util::codec`](https://docs.rs/tokio-util/latest/tokio_util/codec/index.html) — `Framed`, `Decoder`, `Encoder` и готовые кодеки.
- [`LengthDelimitedCodec`](https://docs.rs/tokio-util/latest/tokio_util/codec/length_delimited/index.html) — все параметры кодека с префиксом длины (размер префикса, порядок байтов, пределы).
- [Крейт `bytes`](https://docs.rs/bytes/latest/bytes/) — `Bytes`, `BytesMut`, `Buf` и `BufMut`.
- [Redis serialization protocol (RESP)](https://redis.io/docs/latest/develop/reference/protocol-spec/) — настоящий, хорошо описанный протокол, который сочетает разделители и префиксы длины.

---

[🏠 Начало](../START-HERE.md) · [← 15.6 Асинхронная работа с сетью в Tokio](06-async-networking-with-tokio.md) · [📋 Этап 15](README.md) · [15.8 HTTP с нуля →](08-http-from-scratch.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
