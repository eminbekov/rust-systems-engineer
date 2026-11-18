<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 15.15 Проект: балансировщик нагрузки TCP](15-project-tcp-load-balancer.md) · [📋 Этап 15](README.md) · [Этап 16 — Бэкенд-сервисы →](../16-backend-services/README.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/15-networking/16-checkpoint.md) <!-- nav -->

# Урок 15.16 — Контрольная точка

> **Этап 15 — Сети** · Урок 16 из 16 · ⏱ около 105 минут

## Что вы узнаете

Этот урок не учит ничему новому. Он проверяет, умеете ли вы применять то, чему научил Этап 15, и показывает, куда вернуться, если что-то ещё слабо. Вы:

- Ответите на 18 вопросов об уровнях и адресах, TCP и UDP, сокетах и асинхронной работе с сетью, протоколах и HTTP, DNS, TLS и QUIC, прокси и балансировщиках нагрузки.
- Решите 4 практических задания: три маленькие программы с тестами и одну диагностику настоящих сообщений об ошибках.
- Найдёте уроки, которые стоит повторить, с прямыми ссылками.

## Перед началом

- Вы прошли [Урок 15.15 — Проект: балансировщик нагрузки TCP](15-project-tcp-load-balancer.md), и проекты `p15-dns` и `p15-lb` проходят свои тесты.
- Отвечайте на вопросы **с закрытой книгой**: без уроков, без компилятора, без поиска. Сначала запишите ответ, потом откройте скрытый.
- В практических заданиях можно пользоваться компилятором и уроками, но сначала попробуйте без подсказок. Создайте для них один пакет:

```bash
cd ~/projects/rust
cargo new --lib code/p15-check
cd code/p15-check
cargo add tokio@1 --features full
cargo add bytes@1
mkdir tests
```

Добавьте в `src/lib.rs` строки `pub mod subnet;`, `pub mod frames;` и `pub mod words;`; каждое задание заполняет один из этих модулей.

## Новые слова

| Слово | Значение |
|---|---|
| **контрольная точка** (checkpoint) | Пауза в конце этапа, чтобы проверить себя, прежде чем идти дальше. |
| **с закрытой книгой** (closed book) | Ответ только по памяти, ничего не подсматривая. |
| **диагностика** (diagnosis) | Поиск причины сбоя по его признакам, здесь — по сообщениям об ошибках. |
| **TCP / UDP / DNS / TLS** | Transmission Control Protocol / User Datagram Protocol / Domain Name System / Transport Layer Security (протокол управления передачей / протокол пользовательских датаграмм / система доменных имён / протокол защиты транспортного уровня): протоколы этого этапа. |
| **Другие сокращения этого этапа** | IP (Internet Protocol, межсетевой протокол), MAC (Media Access Control, адрес канального уровня), ОС (OS, операционная система), SYN/RST (флаги TCP *synchronize* и *reset*), HTTP (HyperText Transfer Protocol, протокол передачи гипертекста), TLD (Top-Level Domain, домен верхнего уровня), TTL (Time To Live, время жизни), CNAME (Canonical NAME, каноническое имя), CA (Certificate Authority, удостоверяющий центр), SAN (Subject Alternative Name, альтернативное имя субъекта), SNI (Server Name Indication, указание имени сервера), ALPN (Application-Layer Protocol Negotiation, согласование протокола прикладного уровня), RFC (Request for Comments), L4/L7 (уровень 4/уровень 7), CIDR (Classless Inter-Domain Routing, бесклассовая междоменная маршрутизация: запись `адрес/префикс`), NAT (Network Address Translation, преобразование сетевых адресов), QUIC (транспорт HTTP/3 поверх UDP; не сокращение), API (Application Programming Interface, программный интерфейс). |

## Вопросы для самопроверки

Ставьте себе один балл за каждый **верный и полный** ответ. 15 баллов и больше — идите дальше. Меньше — сначала повторите уроки из таблицы в конце.

### Уровни, адреса и маршрутизация

1. Назовите четыре уровня модели интернета и для каждого — что он решает.

<details>
<summary>Ответ</summary>

Канальный (link): следующий шаг в локальной сети (MAC-адреса). Межсетевой (internet): какой компьютер (IP-адреса, маршрутизация). Транспортный (transport): какая программа (порты; TCP или UDP). Прикладной (application): что означают байты (HTTP, DNS, TLS…). Каждый уровень оборачивает уровень выше в собственный заголовок ([Урок 15.1](01-how-the-internet-works.md)).

</details>

2. Каковы адрес сети, широковещательный адрес и число пригодных адресов хостов для `192.168.5.130/26`?

<details>
<summary>Ответ</summary>

`/26` оставляет 6 бит под хосты, то есть блоки по 64: `.0`, `.64`, `.128`, `.192`. `130` попадает в блок, начинающийся с `.128`. Сеть `192.168.5.128`, широковещательный адрес `192.168.5.191`, пригодных 2⁶ − 2 = 62 адреса ([Урок 15.2](02-ip-addresses-and-routing.md)).

</details>

3. В таблице маршрутизации есть `0.0.0.0/0 via 10.0.0.1`, `10.0.0.0/8 dev eth0` и `10.1.2.0/24 via 10.0.0.9`. Какой маршрут используется для `10.1.2.3` и почему?

<details>
<summary>Ответ</summary>

`10.1.2.0/24 via 10.0.0.9`. Подходят все три маршрута, и **совпадение по самому длинному префиксу** (longest prefix match) выбирает самый конкретный — `/24` ([Урок 15.2](02-ip-addresses-and-routing.md)).

</details>

### TCP, UDP и сокеты

4. Что означает «connection refused» на уровне TCP и чем это отличается от соединения, которое завершилось по тайм-ауту?

<details>
<summary>Ответ</summary>

«Refused»: хост ответил на SYN пакетом RST, потому что на этом порту никто не слушает; вы узнаёте об этом сразу. Тайм-аут: никто вообще не ответил (хост выключен или файрвол отбрасывает пакеты), поэтому клиент ждёт до своего предела ([Урок 15.3](03-tcp-and-udp.md)).

</details>

5. У сервера тысячи сокетов в состоянии `CLOSE-WAIT`. Чья это ошибка и в чём она?

<details>
<summary>Ответ</summary>

Самого сервера. `CLOSE-WAIT` означает, что другая сторона закрыла соединение, а эта программа ещё не закрыла свой сокет: она забывает удалять или закрывать соединения, когда собеседник уходит ([Урок 15.3](03-tcp-and-udp.md)).

</details>

6. Клиент дважды вызывает `write_all` по 100 байт. Сервер один раз вызывает `read`. Сколько байт он может получить и что из этого следует для любого протокола поверх TCP?

<details>
<summary>Ответ</summary>

Что угодно от 1 до 200 байт: TCP — поток байтов и границ сообщений не хранит. Любому протоколу поверх TCP нужен **фрейминг** (разделитель, префикс длины или фиксированный размер), а читающая сторона должна справляться с частичными и склеенными сообщениями ([Уроки 15.4](04-tcp-sockets-in-rust.md) и [15.7](07-designing-a-protocol.md)).

</details>

7. Что произойдёт с UDP-датаграммой в 2000 байт, если получатель вызывает `recv_from` с буфером в 512 байт?

<details>
<summary>Ответ</summary>

Получатель получит первые 512 байт, а остаток этой датаграммы молча потеряется. UDP сохраняет границы сообщений, поэтому один приём возвращает не больше одной датаграммы, и буфер должен быть достаточно большим ([Урок 15.5](05-udp-sockets-in-rust.md)).

</details>

### Асинхронная работа с сетью, протоколы и HTTP

8. Почему сервер на Tokio может держать открытыми 10 000 простаивающих соединений на нескольких потоках, а сервер «поток на соединение» так легко не может?

<details>
<summary>Ответ</summary>

Каждый поток ОС стоит стека и планирования в ядре. Задача Tokio — маленький конечный автомат; рантайм использует epoll (Linux) или kqueue (macOS), чтобы будить только те задачи, чьи сокеты готовы, поэтому простаивающие соединения почти ничего не стоят ([Урок 15.6](06-async-networking-with-tokio.md)).

</details>

9. У декодера с префиксом длины три возможных исхода. Какие, и что он должен делать с буфером в каждом случае?

<details>
<summary>Ответ</summary>

Полный фрейм: убрать из буфера ровно этот фрейм и вернуть его. Ещё не полный: ничего не потреблять и ждать новых байтов. Некорректный (например, длина больше предела): сразу вернуть ошибку, не дожидаясь и не выделяя столько байт ([Урок 15.7](07-designing-a-protocol.md)).

</details>

10. В запросе есть и `Content-Length`, и `Transfer-Encoding: chunked`. Что должен сделать сервер и почему?

<details>
<summary>Ответ</summary>

Отвергнуть его (ответить `400` и закрыть соединение). Если прокси и сервер расходятся во мнении, где кончается тело, злоумышленник может спрятать второй запрос внутри первого: это контрабанда запросов (request smuggling) ([Урок 15.8](08-http-from-scratch.md)).

</details>

11. Зачем в сервере на `hyper` тело оборачивают в `Limited::new(body, MAX)` перед `collect()` и какой статус должно получить слишком большое тело?

<details>
<summary>Ответ</summary>

Без ограничения `collect()` держит всё тело в памяти, и один клиент может исчерпать память сервера. `Limited` завершается ошибкой, как только приходит больше `MAX` байт; сервер отвечает `413 Payload Too Large` ([Урок 15.9](09-http-with-hyper-and-reqwest.md)).

</details>

### DNS

12. Опишите путь поиска `www.rust-lang.org` от вашей программы до ответа, назвав три вида серверов.

<details>
<summary>Ответ</summary>

Программа спрашивает stub-резолвер ОС (после `/etc/hosts`). Stub спрашивает **рекурсивный резолвер**, который спрашивает **корневой сервер** (реферал к серверам `org`), **TLD-сервер** `org` (реферал к серверам `rust-lang.org`) и наконец **авторитативный** сервер, который отвечает с флагом `aa`: здесь CNAME и записи A. Рекурсивный резолвер кэширует каждый ответ на время его TTL ([Урок 15.10](10-dns-how-names-become-addresses.md)).

</details>

13. В DNS-сообщении поле имени содержит `c0 0c`. Что это значит и что должен проверять парсер, чтобы враждебный пакет не мог его повесить?

<details>
<summary>Ответ</summary>

Указатель сжатия (два старших бита `11`) на смещение 12, где начинается имя вопроса. Парсер должен следовать по указателям только назад и ограничивать число прыжков, проверять каждую позицию на выход за длину сообщения и продолжать чтение сразу после **первого** указателя ([Урок 15.11](11-project-dns-resolver.md)).

</details>

### TLS и QUIC

14. Ваш клиент на Rust сообщает `InvalidCertificate(UnknownIssuer)` для вашего тестового сервера и `NotValidForNameContext` для другого. Что не так в каждом случае?

<details>
<summary>Ответ</summary>

`UnknownIssuer`: цепочка не ведёт к корню, которому доверяет клиент; клиенту нужен ваш тестовый CA в `RootCertStore`. `NotValidForNameContext`: цепочка в порядке, но имени, которое запросил клиент, нет в списке SAN сертификата ([Урок 15.12](12-tls-in-rust-servers.md)).

</details>

15. Почему потерянный пакет замедляет все запросы HTTP/2-соединения, но только один поток HTTP/3-соединения?

<details>
<summary>Ответ</summary>

HTTP/2 работает поверх одного потока байтов TCP, который доставляет строго по порядку, поэтому всё после потерянного пакета ждёт (блокировка начала очереди). HTTP/3 работает поверх QUIC, где внутри UDP-датаграмм независимые потоки; ждут только те потоки, чьи данные были в потерянном пакете ([Урок 15.13](13-quic-and-http3.md)).

</details>

16. Что QUIC требует от TLS такого, без чего может обойтись обычный TLS-сервер поверх TCP, и что происходит, если клиент об этом забывает?

<details>
<summary>Ответ</summary>

ALPN: QUIC-клиент обязан сказать, какой прикладной протокол он хочет (например, `h3`). Без этого сервер завершает рукопожатие предупреждением *no_application_protocol* (`quinn` показывает его как `Code::crypto(78)`) ([Урок 15.13](13-quic-and-http3.md)).

</details>

### Прокси и балансировщики нагрузки

17. Назовите две вещи, которые может прокси L7 и не может прокси L4, и одно преимущество прокси L4.

<details>
<summary>Ответ</summary>

L7 может маршрутизировать каждый запрос по пути или хосту и добавлять или менять заголовки (например, `X-Forwarded-For`), кэшировать или повторять идемпотентные запросы. L4 работает с любым протоколом поверх TCP и почти ничего не стоит на байт, потому что никогда не разбирает трафик ([Урок 15.14](14-proxies-and-load-balancers.md)).

</details>

18. Перечислите шаги корректного завершения балансировщика нагрузки в правильном порядке.

<details>
<summary>Ответ</summary>

Получить SIGTERM или SIGINT; перестать принимать и закрыть слушающий сокет; остановить фоновые задачи, например проверки работоспособности; ждать открытые соединения (`TaskTracker::wait`) с тайм-аутом осушения; выйти, оборвав то, что осталось открытым после срока ([Урок 15.15](15-project-tcp-load-balancer.md)).

</details>

## Практические задания

### Задание 1 — Калькулятор подсетей

В `src/subnet.rs` напишите `pub fn subnet(cidr: &str) -> Option<(Ipv4Addr, Ipv4Addr, u32)>`, которая для ввода вроде `"10.20.30.40/22"` возвращает адрес сети, широковещательный адрес и число пригодных хостов (2ʰ − 2, где h — число бит под хосты). Возвращайте `None` для некорректного ввода и для префиксов больше 30. Напишите тесты для `/22`, для граничных `/30` и `/0` и для плохого ввода.

<details>
<summary>Подсказка</summary>

`Ipv4Addr::to_bits()` даёт адрес как `u32`, `Ipv4Addr::from_bits` превращает обратно. Маска — `u32::MAX << (32 - prefix)`, но сдвигать `u32` на 32 нельзя, поэтому `/0` обработайте отдельно. Число хостов считайте в `u64`: для `/0` 2³² не помещается в `u32`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::net::Ipv4Addr;

/// Network address, broadcast address and number of usable hosts of an IPv4 subnet
/// such as "10.20.30.40/22". Prefixes /31 and /32 have no usable hosts in this rule: None.
pub fn subnet(cidr: &str) -> Option<(Ipv4Addr, Ipv4Addr, u32)> {
    let (addr, prefix) = cidr.split_once('/')?;
    let addr: Ipv4Addr = addr.parse().ok()?;
    let prefix: u32 = prefix.parse().ok()?;
    if prefix > 30 {
        return None;
    }
    let mask = if prefix == 0 {
        0
    } else {
        u32::MAX << (32 - prefix)
    };
    let network = addr.to_bits() & mask;
    let broadcast = network | !mask;
    let usable = (1u64 << (32 - prefix)) - 2;
    Some((
        Ipv4Addr::from_bits(network),
        Ipv4Addr::from_bits(broadcast),
        usable as u32,
    ))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_slash_22() {
        let (net, bc, n) = subnet("10.20.30.40/22").unwrap();
        assert_eq!(net, Ipv4Addr::new(10, 20, 28, 0));
        assert_eq!(bc, Ipv4Addr::new(10, 20, 31, 255));
        assert_eq!(n, 1022);
    }

    #[test]
    fn the_borders() {
        assert_eq!(subnet("192.168.1.77/30").unwrap().2, 2);
        let (net, bc, n) = subnet("8.8.8.8/0").unwrap();
        assert_eq!(
            (net, bc, n),
            (Ipv4Addr::new(0, 0, 0, 0), Ipv4Addr::BROADCAST, u32::MAX - 1)
        );
        assert_eq!(subnet("10.0.0.1/31"), None);
        assert_eq!(subnet("10.0.0.1/33"), None);
        assert_eq!(subnet("10.0.0/24"), None);
    }
}
```

Почему это работает: `/22` оставляет 10 бит под хосты; `30 = 0b0001_1110` с обнулёнными двумя последними битами — это `28`, поэтому сеть — `10.20.28.0`, а широковещательный адрес выставляет все 10 бит хоста: `10.20.31.255`. Каждый `?` превращает плохой ввод в `None`, а не в панику.

</details>

### Задание 2 — Декодер фреймов

В `src/frames.rs` напишите `pub fn decode(buf: &mut BytesMut) -> Result<Option<Bytes>, TooLong>` для фреймов из 2-байтовой длины в big-endian и стольких же байт, с максимумом 1024 байта. Тесты: фрейм, пришедший двумя кусками, два фрейма в одном буфере и длина 65 535, которую нужно сразу отвергнуть.

<details>
<summary>Подсказка</summary>

Проверяйте в таком порядке: есть ли хотя бы 2 байта? длина больше предела? весь ли фрейм на месте? Только после этого `buf.advance(2)` и `buf.split_to(len).freeze()`. До этого `buf` не меняйте.

</details>

<details>
<summary>Ответ</summary>

```rust
use bytes::{Buf, Bytes, BytesMut};

pub const MAX_FRAME: usize = 1024;

#[derive(Debug, PartialEq)]
pub struct TooLong(pub usize);

/// Takes one frame (2-byte big-endian length, then that many bytes) off the front of `buf`.
/// Ok(None) means "not complete yet"; nothing is consumed then.
pub fn decode(buf: &mut BytesMut) -> Result<Option<Bytes>, TooLong> {
    if buf.len() < 2 {
        return Ok(None);
    }
    let len = u16::from_be_bytes([buf[0], buf[1]]) as usize;
    if len > MAX_FRAME {
        return Err(TooLong(len));
    }
    if buf.len() < 2 + len {
        return Ok(None);
    }
    buf.advance(2);
    Ok(Some(buf.split_to(len).freeze()))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn waits_for_the_whole_frame() {
        let mut buf = BytesMut::from(&[0, 5, b'h', b'e'][..]);
        assert_eq!(decode(&mut buf), Ok(None));
        assert_eq!(buf.len(), 4, "nothing may be consumed");
        buf.extend_from_slice(b"llo");
        assert_eq!(decode(&mut buf), Ok(Some(Bytes::from_static(b"hello"))));
        assert!(buf.is_empty());
    }

    #[test]
    fn two_frames_in_one_read() {
        let mut buf = BytesMut::from(&[0, 1, b'a', 0, 2, b'b', b'c'][..]);
        assert_eq!(decode(&mut buf).unwrap().unwrap(), "a");
        assert_eq!(decode(&mut buf).unwrap().unwrap(), "bc");
        assert_eq!(decode(&mut buf), Ok(None));
    }

    #[test]
    fn rejects_a_huge_length_before_reading_it() {
        let mut buf = BytesMut::from(&[0xff, 0xff][..]);
        assert_eq!(decode(&mut buf), Err(TooLong(65535)));
    }
}
```

Почему это работает: предел проверяется, как только длина известна, ещё до ожидания тела, поэтому враждебная длина ничего не стоит. Поскольку потребляются только полные фреймы, оставшиеся байты следующего фрейма остаются в буфере до следующего вызова.

</details>

### Задание 3 — Асинхронный счётчик слов с тестом

В `src/words.rs` напишите `pub async fn serve(listener: TcpListener) -> std::io::Result<()>`: на каждую строку, которую присылает клиент, сервер отвечает числом слов в этой строке и переводом строки. Одновременно должно обслуживаться много клиентов. В `tests/words.rs` запустите сервер на порту 0 и проверьте двух клиентов **одновременно**, с тайм-аутом, чтобы зависший сервер проваливал тест, а не блокировал его.

<details>
<summary>Подсказка</summary>

Одна задача на соединение; `stream.into_split()` и `BufReader::new(reader).lines()`; `split_whitespace().count()`. В тесте запустите двух клиентов через `tokio::join!` внутри `tokio::time::timeout`.

</details>

<details>
<summary>Ответ</summary>

`src/words.rs`:

```rust
use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::TcpListener;

/// Serves forever: for every line a client sends, answers with the number of words in it.
pub async fn serve(listener: TcpListener) -> std::io::Result<()> {
    loop {
        let (stream, _) = listener.accept().await?;
        tokio::spawn(async move {
            let (reader, mut writer) = stream.into_split();
            let mut lines = BufReader::new(reader).lines();
            while let Ok(Some(line)) = lines.next_line().await {
                let answer = format!("{}\n", line.split_whitespace().count());
                if writer.write_all(answer.as_bytes()).await.is_err() {
                    break;
                }
            }
        });
    }
}
```

`tests/words.rs`:

```rust
use std::time::Duration;

use tokio::io::{AsyncBufReadExt, AsyncWriteExt, BufReader};
use tokio::net::{TcpListener, TcpStream};
use tokio::time::timeout;

async fn ask(addr: std::net::SocketAddr, lines: &[&str]) -> Vec<String> {
    let stream = TcpStream::connect(addr).await.unwrap();
    let (reader, mut writer) = stream.into_split();
    let mut replies = BufReader::new(reader).lines();
    let mut out = Vec::new();
    for line in lines {
        writer
            .write_all(format!("{line}\n").as_bytes())
            .await
            .unwrap();
        out.push(replies.next_line().await.unwrap().unwrap());
    }
    out
}

#[tokio::test]
async fn two_clients_at_the_same_time() {
    let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
    let addr = listener.local_addr().unwrap();
    tokio::spawn(p15_check::words::serve(listener));

    let one = ask(addr, &["hello world", "", "a b c"]);
    let two = ask(addr, &["  spaces   everywhere  "]);
    let (one, two) = timeout(Duration::from_secs(2), async { tokio::join!(one, two) })
        .await
        .expect("the server must answer within 2 s");
    assert_eq!(one, ["2", "0", "3"]);
    assert_eq!(two, ["2"]);
}
```

Когда сделаны все три задания, `cargo test` печатает:

```text
test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
...
test two_clients_at_the_same_time ... ok
```

Почему это работает: у каждого клиента своя задача, поэтому два вызова `ask` выполняются одновременно и не ждут друг друга. Тест проверяет граничные случаи (пустая строка, лишние пробелы), а тайм-аут превращает «сервер завис» в понятный провал.

</details>

### Задание 4 — Диагностика ошибок (без кода)

Для каждого сообщения скажите, на каком уровне проблема, какова самая вероятная причина и какую команду вы запустите первой, чтобы это проверить.

```text
a) curl: (6) Could not resolve host: no-such-name.rust-lang.org
b) curl: (7) Failed to connect to 127.0.0.1 port 8099 after 0 ms: Couldn't connect to server
c) curl: (28) Failed to connect to 192.0.2.1 port 80 after 3003 ms: Timeout was reached
d) curl: (60) SSL certificate problem: unable to get local issuer certificate
e) curl: (56) Recv failure: Connection reset by peer   (through your load balancer, every second request)
```

<details>
<summary>Подсказка</summary>

Подумайте, как далеко зашло каждое соединение: имя, TCP-рукопожатие, TLS-рукопожатие или данные после него. «After 0 ms» и «after 3003 ms» говорят, ответил ли кто-нибудь.

</details>

<details>
<summary>Ответ</summary>

- **a)** DNS. Имени не существует (NXDOMAIN) или оно написано с ошибкой, либо недоступен ни один резолвер. Проверьте через `dig +noall +comments <name> | grep status` и загляните в `/etc/hosts` ([Урок 15.10](10-dns-how-names-become-addresses.md)).
- **b)** TCP. В соединении сразу отказали (RST): на порту 8099 никто не слушает. Проверьте через `ss -tlnp` в Linux (или `lsof -iTCP:8099 -sTCP:LISTEN` в macOS), запущен ли сервер и на каком адресе ([Уроки 15.3](03-tcp-and-udp.md) и [15.4](04-tcp-sockets-in-rust.md)).
- **c)** TCP или IP. На SYN за 3 секунды никто не ответил: хост выключен, маршрут неверен или файрвол отбрасывает пакеты. Проверьте маршрут через `ip route get 192.0.2.1` (Урок 15.2) и сделайте перехват `tcpdump`, чтобы увидеть, уходят ли SYN и не возвращается ли ничего.
- **d)** TLS. Рукопожатие дошло до проверки сертификата: цепочка не ведёт к доверенному корню (частный или самоподписанный CA). Посмотрите цепочку через `openssl s_client -connect host:443 -showcerts` и доверьтесь правильному CA через `--cacert`, а не отключайте проверку ([Урок 15.12](12-tls-in-rust-servers.md)).
- **e)** Балансировщик нагрузки. Он принял клиента, но не смог достучаться до одного бэкенда и закрыл соединение: один бэкенд не работает, а проверок работоспособности и переключения при отказе нет. Поищите в логе прокси `cannot reach` ([Уроки 15.14](14-proxies-and-load-balancers.md) и [15.15](15-project-tcp-load-balancer.md)).

Все пять сообщений настоящие; бо́льшую часть из них вы видели на этом этапе. Читать ошибку как ответ на вопрос «как далеко зашло соединение?» — самый быстрый путь к причине.

</details>

## Если что-то не получилось

| Тема | Урок для повторения |
|---|---|
| Уровни, пакеты, инкапсуляция, `tcpdump` | [Урок 15.1 — Как работает интернет: уровни и пакеты](01-how-the-internet-works.md) |
| IPv4/IPv6, CIDR, подсети, NAT, маршрутизация | [Урок 15.2 — IP-адреса, подсети и маршрутизация](02-ip-addresses-and-routing.md) |
| Рукопожатие, состояния, отказ и тайм-аут, TCP и UDP | [Урок 15.3 — TCP и UDP](03-tcp-and-udp.md) |
| `TcpListener`, `TcpStream`, частичное чтение, тайм-ауты | [Урок 15.4 — TCP-сокеты в Rust](04-tcp-sockets-in-rust.md) |
| `UdpSocket`, размеры датаграмм, повторы | [Урок 15.5 — UDP-сокеты в Rust](05-udp-sockets-in-rust.md) |
| Серверы на Tokio, разделённые потоки, `select!`, ограничения | [Урок 15.6 — Асинхронная работа с сетью в Tokio](06-async-networking-with-tokio.md) |
| Фрейминг, декодеры, `BytesMut`, кодеки | [Урок 15.7 — Проектирование протокола: фрейминг и разбор](07-designing-a-protocol.md) |
| Сообщения HTTP/1.1, длина тела, keep-alive, контрабанда запросов | [Урок 15.8 — HTTP с нуля](08-http-from-scratch.md) |
| Серверы на `hyper`, ограничения тела, `reqwest`, сквозные тесты | [Урок 15.9 — HTTP с hyper и reqwest](09-http-with-hyper-and-reqwest.md) |
| Резолверы, типы записей, TTL, формат на проводе | [Урок 15.10 — DNS: как имена превращаются в адреса](10-dns-how-names-become-addresses.md) |
| Сборка и разбор DNS-сообщений, сжатие | [Урок 15.11 — Проект: DNS-резолвер](11-project-dns-resolver.md) |
| Сертификаты, серверы на `rustls`, SNI, ALPN, взаимный TLS | [Урок 15.12 — TLS в серверах на Rust с rustls](12-tls-in-rust-servers.md) |
| Потоки QUIC, HTTP/3, ALPN в QUIC | [Урок 15.13 — QUIC и HTTP/3](13-quic-and-http3.md) |
| Прямые и обратные прокси, L4/L7, алгоритмы | [Урок 15.14 — Прокси и балансировщики нагрузки](14-proxies-and-load-balancers.md) |
| Проверки работоспособности, ограничения, корректное завершение, тестирование | [Урок 15.15 — Проект: балансировщик нагрузки TCP](15-project-tcp-load-balancer.md) |
| Задачи, `select!`, семафоры, отмена | [Урок 14.13 — Tokio подробно: задачи и каналы](../14-concurrency-and-async/13-tokio-tasks-and-channels.md) и [Урок 14.14 — Отмена, тайм-ауты и корректное завершение](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md) |
| Основы TLS, цепочки доверия, `curl -v` | [Урок 10.11 — TLS и сертификаты](../10-security-fundamentals/11-tls-and-certificates.md) |

Повторяя урок, не ограничивайтесь перечитыванием: заново сделайте его упражнения и пишите тесты первыми.

## Что дальше

Вы прошли Этап 15. Теперь вы можете проследить запрос от имени до байтов на проводе, писать серверы и клиенты для TCP, UDP, HTTP, TLS и QUIC, безопасно разбирать двоичные протоколы, а также построить и протестировать балансировщик нагрузки. Прежде чем идти дальше:

1. Сделайте коммит пакета контрольной точки и отправьте последнее состояние своих проектов в свой репозиторий.
2. Запишите в журнал обучения несколько строк: какой уровень всё ещё кажется чёрным ящиком и какую дополнительную цель `p15-dns` или `p15-lb` вы хотите попробовать.
3. Сделайте перерыв.

На **Этапе 16 — Бэкенд-сервисы** вы используете эту основу, чтобы строить настоящие сервисы: HTTP API на `axum` (который построен на `hyper`), базы данных через `sqlx`, аутентификацию, gRPC, кэширование, обмен сообщениями, а также логи, метрики и трассировки. Начните с [Урока 16.1 — Что такое бэкенд-сервис](../16-backend-services/01-what-a-backend-service-is.md).

## Источники и что почитать

- [Computer Networks: A Systems Approach](https://book.systemsapproach.org/) — бесплатный учебник, который подробнее раскрывает каждый уровень этого этапа.
- [High Performance Browser Networking](https://hpbn.co/) — бесплатная книга о задержках, TCP, TLS, HTTP/2 и многом другом.
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/) — классическое руководство по API сокетов, которое лежит под `std::net` и Tokio.
- [The Tokio tutorial](https://tokio.rs/tokio/tutorial) — официальное руководство по асинхронной работе с сетью в Tokio.
- [RFC Editor](https://www.rfc-editor.org/) — официальный дом всех RFC, использованных на этом этапе (TCP 9293, HTTP 9110/9112, DNS 1035, TLS 8446, QUIC 9000, HTTP/3 9114).

---

[🏠 Начало](../START-HERE.md) · [← 15.15 Проект: балансировщик нагрузки TCP](15-project-tcp-load-balancer.md) · [📋 Этап 15](README.md) · [Этап 16 — Бэкенд-сервисы →](../16-backend-services/README.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
