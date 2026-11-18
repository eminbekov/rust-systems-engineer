<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 15.12 TLS в серверах на Rust с rustls](12-tls-in-rust-servers.md) · [📋 Этап 15](README.md) · [15.14 Прокси и балансировщики нагрузки →](14-proxies-and-load-balancers.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/15-networking/13-quic-and-http3.md) <!-- nav -->

# Урок 15.13 — QUIC и HTTP/3

> **Этап 15 — Сети** · Урок 13 из 16 · ⏱ около 135 минут

## Что вы узнаете

- Какие проблемы TCP вместе с TLS привели к появлению QUIC: блокировка начала очереди, медленные рукопожатия и протокол, который трудно менять.
- Как работает QUIC: UDP-датаграммы, встроенный TLS 1.3, независимые потоки и идентификаторы соединений.
- Как открыть QUIC-соединение и много потоков с помощью `quinn` и как QUIC выглядит в перехвате пакетов.
- Как HTTP/3 отображает HTTP на QUIC, как клиенты узнают, что сервер его поддерживает, и как сделать HTTP/3-запрос с `h3`.

## Перед началом

- Вы прошли [Урок 15.12 — TLS в серверах на Rust с rustls](12-tls-in-rust-servers.md): сертификаты, `rcgen`, ALPN и криптопровайдер `ring`.
- Вы знаете TCP (Transmission Control Protocol, протокол управления передачей) и UDP (User Datagram Protocol, протокол пользовательских датаграмм) из [Урока 15.3 — TCP и UDP](03-tcp-and-udp.md), а версии HTTP (HyperText Transfer Protocol, протокол передачи гипертекста) — из [Урока 15.9 — HTTP с hyper и reqwest](09-http-with-hyper-and-reqwest.md).
- Создайте учебный пакет:

```bash
cd ~/projects/rust
cargo new code/p15-quic
cd code/p15-quic
mkdir -p src/bin tests && rm src/main.rs
cargo add tokio@1 --features full
cargo add quinn@0.11 rcgen@0.14
cargo add rustls@0.23 --no-default-features --features ring,std
cargo add h3@0.0.8 h3-quinn@0.0.10 http@1 bytes@1 webpki-roots@1
```

`quinn` по умолчанию использует `rustls` с провайдером `ring`, поэтому во всей программе ровно один криптопровайдер, как советует Урок 15.12.

## Новые слова

| Слово | Значение |
|---|---|
| **QUIC** | Транспортный протокол, который работает поверх UDP и включает шифрование и потоки. Название больше не считается аббревиатурой. Определён в RFC 9000 (RFC: Request for Comments, документы, определяющие интернет-протоколы). |
| **HTTP/3** | HTTP поверх QUIC (RFC 9114). Те же методы, заголовки и коды статуса, что в HTTP/1.1 и HTTP/2; другой транспорт. |
| **TLS 1.3** (Transport Layer Security, протокол защиты транспортного уровня) | Протокол шифрования из Урока 10.11. QUIC использует его рукопожатие внутри собственных пакетов. |
| **блокировка начала очереди** (head-of-line blocking) | Ситуация, когда один потерянный или медленный кусок задерживает всё, что стоит за ним в очереди, даже куски других запросов. |
| **поток** (stream, QUIC) | Упорядоченный поток байтов внутри QUIC-соединения. У соединения их могут быть тысячи; потеря в одном не останавливает остальные. |
| **двунаправленный / однонаправленный поток** (bidirectional / unidirectional stream) | Поток, по которому могут отправлять обе стороны / только та, что его открыла. |
| **идентификатор соединения** (connection ID) | Число в каждом пакете QUIC, которое определяет соединение, поэтому оно переживает смену IP-адреса (Internet Protocol, межсетевой протокол) или порта. |
| **миграция соединения** (connection migration) | Сохранение QUIC-соединения, когда клиент перемещается, например с Wi-Fi на мобильную сеть. |
| **RTT** (Round-Trip Time, время кругового пути) | Время, за которое пакет доходит до другой стороны и возвращается ответ. |
| **0-RTT** (zero round-trip time, нулевое время кругового пути) | Отправка данных запроса в самом первом пакете серверу, с которым вы уже общались раньше. Быстро, но злоумышленник может повторить такие данные. |
| **endpoint** (`quinn`) | Один UDP-сокет, который может быть клиентом, сервером или и тем и другим, со всеми своими QUIC-соединениями. |
| **Alt-Svc** (Alternative Services, альтернативные сервисы) | Заголовок HTTP-ответа, который говорит «я доступен ещё и по другому протоколу», например `h3=":443"`. |
| **QPACK** | Сжатие заголовков в HTTP/3 (RFC 9204), устроенное так, чтобы потоки не блокировали друг друга. |
| **датаграмма** (datagram, QUIC) | Ненадёжное сообщение внутри зашифрованного QUIC-соединения: оно может потеряться и никогда не отправляется повторно (RFC 9221). |
| **SYN, ACK, API** | Флаги рукопожатия TCP «synchronise» (синхронизация) и «acknowledge» (подтверждение) из Урока 15.3; Application Programming Interface (программный интерфейс) — типы и функции, которые библиотека предлагает вашему коду. |

## 1. Зачем новый транспорт

HTTP/2 (Урок 15.9) отправляет много запросов через **одно** TCP-соединение. Это эффективно, но TCP доставляет байты строго по порядку. Если потерян один пакет, TCP задерживает все байты после него, пока потерянный пакет не будет отправлен повторно, — даже байты, относящиеся к совсем другим запросам:

```text
 HTTP/2 over TCP: one ordered byte stream
   packets:   [A1][B1][C1][A2][B2]...        B1 is lost
   delivered: A1 ── wait ── wait ── wait ── (B1 resent) B1 C1 A2 B2
              requests A and C wait for B's lost packet: head-of-line blocking

 HTTP/3 over QUIC: independent streams inside UDP datagrams
   stream A:  A1 A2 ...       delivered at once
   stream B:  B1 (lost) ... B1 resent ... B2      only B waits
   stream C:  C1 ...          delivered at once
```

Есть ещё две проблемы:

1. **Время рукопожатия.** До первого байта HTTP TCP нужен один круг туда-обратно, а TLS 1.3 — ещё один: два RTT. В мобильной сети с RTT 100 мс это 0,2 секунды ожидания, прежде чем что-то произойдёт. QUIC выполняет транспортное и TLS-рукопожатие **вместе**, за один RTT, а при возобновлении прежней сессии может отправить данные даже в первом пакете (0-RTT).
2. **Протокол, который нельзя изменить.** TCP живёт в ядре операционной системы, а многие маршрутизаторы и файрволы на пути заглядывают в TCP-заголовки и отбрасывают то, чего не узнают. Улучшение TCP занимает много лет. QUIC работает поверх обычного UDP, живёт в библиотеке внутри программы (как `quinn`) и шифрует почти все свои заголовки, так что середина сети не может от них зависеть.

Почему UDP, а не новый IP-протокол? Потому что почти все сети на планете уже пропускают UDP (его использует DNS, Domain Name System, система доменных имён). Совершенно новый номер протокола во многих местах заблокировали бы.

## 2. QUIC на одной картинке

```text
 UDP datagram (to port 443)
 └─ QUIC packet: header [connection ID, packet number]  + encrypted payload
                                                           └─ frames:
                                                              STREAM 0: "GET /a ..."
                                                              STREAM 4: "GET /b ..."
                                                              ACK: packets 1-7 received
                                                              CRYPTO: TLS handshake bytes
```

| Свойство | TCP + TLS 1.3 | QUIC |
|---|---|---|
| Работает поверх | IP, в ядре | UDP, в библиотеке |
| Рукопожатие до данных | 2 RTT (1 для TCP, 1 для TLS) | 1 RTT (0-RTT при возобновлении) |
| Шифрование | только полезная нагрузка; заголовки TCP видны | полезная нагрузка и бо́льшая часть заголовка |
| Порядок | один упорядоченный поток байтов | много независимых потоков |
| Потеря одного пакета | блокирует всё после него | блокирует только потоки из этого пакета |
| Смена адреса (Wi-Fi → мобильная сеть) | соединение рвётся | продолжается благодаря идентификатору соединения |
| Сообщения без повторной отправки | невозможно | датаграммы QUIC |

QUIC по-прежнему даёт каждому потоку то, что TCP даёт соединению: повторную отправку потерянных данных, порядок внутри потока, управление потоком (flow control) и управление перегрузкой. Просто он делает это **для каждого потока** внутри одного соединения. TLS не является опцией: каждое QUIC-соединение шифруется TLS 1.3, а ALPN (Application-Layer Protocol Negotiation, согласование протокола прикладного уровня, Урок 15.12) обязателен, поэтому стороны всегда договариваются, на чём говорят.

## 3. QUIC в Rust с quinn

`quinn` — реализация QUIC для Tokio. **Endpoint** — это один UDP-сокет; на нём принимают или открывают соединения, а в соединении открывают потоки. Чтобы пример поместился в один файл, программа запускает сервер и клиент в одном процессе. Создайте `src/bin/quic_echo.rs`:

```rust
use std::net::SocketAddr;
use std::sync::Arc;
use std::time::{Duration, Instant};

use quinn::{ClientConfig, Endpoint, ServerConfig};
use rustls::RootCertStore;
use rustls::pki_types::{CertificateDer, PrivatePkcs8KeyDer};

/// The server: echoes every stream back in upper case. "slow" streams wait 500 ms first.
async fn run_server(endpoint: Endpoint) {
    while let Some(incoming) = endpoint.accept().await {
        tokio::spawn(async move {
            let conn = incoming.await.expect("handshake");
            println!("server: connection from {}", conn.remote_address());
            // Every bidirectional stream the client opens arrives here.
            while let Ok((mut send, mut recv)) = conn.accept_bi().await {
                tokio::spawn(async move {
                    let msg = recv.read_to_end(1024).await.expect("read");
                    if msg.starts_with(b"slow") {
                        tokio::time::sleep(Duration::from_millis(500)).await;
                    }
                    send.write_all(&msg.to_ascii_uppercase())
                        .await
                        .expect("write");
                    send.finish().expect("finish");
                });
            }
        });
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 1. A self-signed certificate for "localhost", in memory.
    let cert = rcgen::generate_simple_self_signed(vec!["localhost".to_string()])?;
    let cert_der = CertificateDer::from(cert.cert);
    let key_der = PrivatePkcs8KeyDer::from(cert.signing_key.serialize_der());

    // 2. The server endpoint: one UDP socket.
    let server_config = ServerConfig::with_single_cert(vec![cert_der.clone()], key_der.into())?;
    let server = Endpoint::server(server_config, "127.0.0.1:0".parse()?)?;
    let server_addr: SocketAddr = server.local_addr()?;
    println!("server: listening on UDP {server_addr}");
    tokio::spawn(run_server(server));

    // 3. The client endpoint trusts exactly this one certificate.
    let mut roots = RootCertStore::empty();
    roots.add(cert_der)?;
    let mut client = Endpoint::client("127.0.0.1:0".parse()?)?;
    client.set_default_client_config(ClientConfig::with_root_certificates(Arc::new(roots))?);

    let start = Instant::now();
    let conn = client.connect(server_addr, "localhost")?.await?;
    println!("client: connected after {:?}", start.elapsed());

    // 4. Three streams on ONE connection, all at the same time.
    let mut tasks = Vec::new();
    for msg in ["slow stream one", "stream two", "stream three"] {
        let conn = conn.clone();
        tasks.push(tokio::spawn(async move {
            let (mut send, mut recv) = conn.open_bi().await?;
            send.write_all(msg.as_bytes()).await?;
            send.finish()?;
            let reply = recv.read_to_end(1024).await?;
            println!(
                "client: {:>7.1?} {:?}",
                start.elapsed(),
                String::from_utf8_lossy(&reply)
            );
            Ok::<_, Box<dyn std::error::Error + Send + Sync>>(())
        }));
    }
    for task in tasks {
        task.await?.map_err(|e| e.to_string())?;
    }

    conn.close(0u32.into(), b"done");
    client.wait_idle().await;
    Ok(())
}
```

**Что происходит:**

1. `rcgen::generate_simple_self_signed` создаёт сертификат и ключ в памяти (в Уроке 15.12 мы делали целый CA, удостоверяющий центр; для одного локального теста хватит самоподписанного сертификата, потому что клиент доверяет ровно этому сертификату).
2. `ServerConfig::with_single_cert` строит TLS-часть сервера для QUIC. `Endpoint::server(config, addr)` привязывает **один UDP-сокет**; никаких `listen`/`accept` на каждое TCP-соединение нет, потому что в UDP нет соединений. QUIC строит их сам.
3. `endpoint.accept()` выдаёт попытку соединения `Incoming`; `.await` на ней выполняет рукопожатие QUIC + TLS. Затем `conn.accept_bi()` выдаёт каждый **поток**, который открывает клиент: пару из половины для отправки и половины для приёма.
4. `read_to_end(1024)` читает, пока другая сторона не **завершит** свою половину потока, с ограничением размера (никогда не читайте неограниченные данные из сети). `send.finish()` говорит «мне больше нечего отправлять в этом потоке», как закрытие пишущей половины TCP-соединения.
5. Endpoint клиента получает `ClientConfig`, который доверяет ровно нашему сертификату. `client.connect(addr, "localhost")` начинает рукопожатие (имя сверяется с сертификатом, как в Уроке 15.12), а `.await` ждёт его завершения.
6. Три задачи открывают три потока в **одном и том же** соединении (`conn.clone()` — дешёвый дескриптор на него). Первый поток сервер заставляет ждать 500 мс.
7. `conn.close(code, reason)` закрывает соединение с прикладным кодом ошибки (0 = всё хорошо), а `wait_idle()` даёт сообщению о закрытии время дойти до другой стороны.

```bash
cargo run -q --bin quic_echo
```

```text
server: listening on UDP 127.0.0.1:56651
client: connected after 4.985083ms
server: connection from 127.0.0.1:52522
client:   5.7ms "STREAM TWO"
client:   5.8ms "STREAM THREE"
client: 507.2ms "SLOW STREAM ONE"
```

Два быстрых потока завершились сразу, хотя медленный поток был открыт первым и делит с ними то же соединение. В одном соединении HTTP/1.1 ответы вернулись бы строго по порядку, и быстрые запросы ждали бы медленный.

> ⚡ **Производительность:** открытие QUIC-потока не стоит ни одного круга туда-обратно: поток просто начинается со своего первого фрейма. Соединение, напротив, стоит рукопожатия. Поэтому открывайте одно соединение на сервер и много потоков в нём — так же, как вы переиспользуете `reqwest::Client` для HTTP.

## 4. Как QUIC выглядит на проводе

Перехватим начало настоящего QUIC-соединения. Программа из раздела 5 говорит по HTTP/3 с публичным сервером; в перехвате это выглядит так (снято в Linux-контейнере командой `tcpdump -i eth0 -n -c 8 udp port 443` во время работы программы; см. «Попробуйте сами»):

```text
07:48:59.544410 IP 172.17.0.3.57268 > 104.18.26.14.443: UDP, length 1200
07:48:59.636023 IP 104.18.26.14.443 > 172.17.0.3.57268: UDP, length 1200
07:48:59.636044 IP 104.18.26.14.443 > 172.17.0.3.57268: UDP, length 1200
07:48:59.636045 IP 104.18.26.14.443 > 172.17.0.3.57268: UDP, length 1200
07:48:59.636046 IP 104.18.26.14.443 > 172.17.0.3.57268: UDP, length 529
07:48:59.639888 IP 172.17.0.3.57268 > 104.18.26.14.443: UDP, length 1200
07:48:59.639957 IP 172.17.0.3.57268 > 104.18.26.14.443: UDP, length 1275
07:48:59.640077 IP 172.17.0.3.57268 > 104.18.26.14.443: UDP, length 66
```

**Что происходит:**

1. Нет SYN, SYN-ACK, ACK (рукопожатия TCP из Урока 15.3): только UDP-датаграммы на порт 443. Для `tcpdump`, как и для любого маршрутизатора по пути, это «UDP, length 1200» и ничего больше.
2. Первая датаграмма клиента — ровно 1200 байт. QUIC требует дополнять первый пакет хотя бы до 1200 байт, чтобы злоумышленник с поддельным адресом источника не мог крошечным пакетом заставить сервер отправить жертве большой ответ.
3. Сервер отвечает своим сертификатом и данными рукопожатия (около 4 KiB, кибибайт, в четырёх датаграммах) примерно через 90 мс: один RTT до этого сервера. Всего через 4 мс после этого клиент уже отправляет свой запрос; HTTP-запрос ушёл во второй порции данных клиента, без лишнего круга туда-обратно.

С `-X` можно увидеть первые байты QUIC-пакета, сразу после 8-байтового заголовка UDP:

```text
	0x0010:  6812 1a0e b8c6 01bb 04b8 32ff c100 0000  h.........2.....
	0x0020:  0114 b03f cb18 29a6 0e8f 8957 269e f74c  ...?..)....W&..L
```

`c1` — первый байт **длинного заголовка** (long header) QUIC (старший бит 1: пакет рукопожатия; следующий бит всегда 1). `00 00 00 01` — версия QUIC: 1, из RFC 9000. `14` = 20 — длина идентификатора соединения получателя, и следующие 20 байт — сам этот идентификатор. Всё после заголовка зашифровано, даже в самом первом пакете.

> 🔒 **Безопасность:** QUIC шифрует больше, чем TCP + TLS: номера пакетов и подтверждения тоже защищены, поэтому сеть не может их ни увидеть, ни изменить. Цена в том, что инструменты вроде файрволов и балансировщиков нагрузки видят меньше. Некоторые компании по этой причине блокируют UDP-порт 443; клиенты тогда переходят на HTTP/2 поверх TCP.

## 5. HTTP/3 с h3

HTTP/3 сохраняет смысл HTTP и меняет транспорт:

- каждая пара «запрос — ответ» использует **свой двунаправленный QUIC-поток**, поэтому медленный или потерянный ответ не блокирует другие;
- заголовки сжимаются QPACK и отправляются в двоичных фреймах (HEADERS, DATA);
- каждая сторона открывает несколько однонаправленных *управляющих* потоков для настроек.

Клиент заранее не знает, что сервер говорит на HTTP/3. Обычно он сначала подключается по HTTP/2 поверх TCP, а сервер объявляет HTTP/3 заголовком **Alt-Svc**:

```bash
curl -sI https://cloudflare-quic.com/ | grep -i -E '^(HTTP|alt-svc)'
```

```text
HTTP/2 200 
alt-svc: h3=":443"; ma=86400
```

**Что происходит:** `h3=":443"` означает «HTTP/3 доступен на UDP-порту 443 того же хоста»; `ma=86400` («max age») говорит, что клиент может помнить это сутки. Следующий запрос браузера пойдёт через QUIC. (`curl` в macOS сам не умеет HTTP/3, поэтому остаётся на HTTP/2.)

Крейт `h3` для Rust реализует HTTP/3 поверх любой QUIC-библиотеки; `h3-quinn` подключает его к `quinn`. Он ещё молод (версия 0.0.x, поэтому его API меняется между версиями), но работает. Создайте `src/bin/h3_get.rs`:

```rust
use std::sync::Arc;

use bytes::Buf;
use quinn::crypto::rustls::QuicClientConfig;
use rustls::RootCertStore;
use tokio::net::lookup_host;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error + Send + Sync>> {
    let host = "cloudflare-quic.com";

    // 1. TLS for QUIC: Mozilla's roots, and the ALPN name of HTTP/3, "h3".
    let roots = RootCertStore::from_iter(webpki_roots::TLS_SERVER_ROOTS.iter().cloned());
    let mut tls = rustls::ClientConfig::builder()
        .with_root_certificates(roots)
        .with_no_client_auth();
    tls.alpn_protocols = vec![b"h3".to_vec()];
    let quic_tls = QuicClientConfig::try_from(tls)?;

    // 2. A QUIC connection over UDP port 443.
    let addr = lookup_host((host, 443))
        .await?
        .find(|a| a.is_ipv4())
        .ok_or("no IPv4 address")?;
    let mut endpoint = quinn::Endpoint::client("0.0.0.0:0".parse()?)?;
    endpoint.set_default_client_config(quinn::ClientConfig::new(Arc::new(quic_tls)));
    let conn = endpoint.connect(addr, host)?.await?;
    println!("QUIC connection to {addr}, RTT about {:?}", conn.rtt());

    // 3. HTTP/3 on top of the QUIC connection.
    let (mut driver, mut send_request) = h3::client::new(h3_quinn::Connection::new(conn)).await?;
    let drive = tokio::spawn(async move { driver.wait_idle().await });

    let request = http::Request::get(format!("https://{host}/")).body(())?;
    let mut stream = send_request.send_request(request).await?;
    stream.finish().await?; // no request body
    let response = stream.recv_response().await?;
    println!("{:?} {}", response.version(), response.status());
    println!("server: {:?}", response.headers().get("server"));

    let mut size = 0;
    while let Some(chunk) = stream.recv_data().await? {
        size += chunk.remaining();
    }
    println!("body: {size} bytes");

    drop(send_request);
    drive.abort();
    endpoint.wait_idle().await;
    Ok(())
}
```

**Что происходит:**

1. Мы сами строим клиентскую конфигурацию `rustls`, потому что HTTP/3 **требует** значение ALPN `h3`. `QuicClientConfig::try_from` проверяет, что конфигурация пригодна для QUIC (только TLS 1.3), и оборачивает её.
2. `lookup_host` (Урок 15.10) разрешает имя; мы выбираем IPv4-адрес. `endpoint.connect(addr, host)` использует `host` для проверки сертификата и SNI (Server Name Indication, указание имени сервера, Урок 15.12).
3. `h3::client::new` настраивает HTTP/3 на соединении и возвращает две вещи: **драйвер** (driver), который должен продолжать работать, чтобы обслуживать управляющие потоки (мы запускаем его в отдельной задаче), и `send_request` — дескриптор для отправки запросов.
4. `send_request(request)` открывает новый двунаправленный поток и отправляет фрейм HEADERS. `finish()` говорит «тела запроса нет». `recv_response()` ждёт заголовки ответа, а `recv_data()` возвращает тело кусками до `None`.
5. В конце мы удаляем дескриптор запросов, останавливаем драйвер и даём endpoint завершить закрытие.

```bash
cargo run -q --bin h3_get
```

```text
QUIC connection to 104.18.26.14:443, RTT about 92.449917ms
HTTP/3.0 200 OK
server: Some("cloudflare")
body: 125959 bytes
```

Ответ говорит `HTTP/3.0`: тот же тип `http::Response`, что вы знаете по `hyper` и `reqwest`, только доставленный через QUIC. (Адрес, RTT и размер зависят от вашей сети и от дня.)

> 📏 **Соглашение:** большинство сервисов не завершают QUIC сами. Обратный прокси или балансировщик нагрузки на границе сети (Урок 15.14) говорит с интернетом по HTTP/3, а с сервисами позади себя — по HTTP/1.1 или HTTP/2. Используйте QUIC прямо в своей программе, когда вам нужны его возможности: много независимых потоков, миграция соединений или датаграммы.

## Частые ошибки

**1. Нет ALPN.** Уберите строку `tls.alpn_protocols = vec![b"h3".to_vec()];`, и рукопожатие завершится ошибкой:

```text
Error: ConnectionClosed(ConnectionClose { error_code: Code::crypto(78), frame_type: None, reason: b"" })
```

`Code::crypto(78)` — это TLS-предупреждение внутри QUIC: шестнадцатеричное `78` — это 120, предупреждение *no_application_protocol*. QUIC-серверы отказывают клиентам, которые не говорят, на каком протоколе хотят общаться.

**2. Забыть `finish()`.** Если клиент так и не вызывает `send.finish()`, `read_to_end` на сервере продолжает ждать новых данных, а клиент ждёт ответа: оба ждут вечно. Программа печатает строки о соединении и затем зависает (здесь её остановили через 3 секунды с помощью `timeout 3`, код возврата 124). Поток заканчивается, только когда отправитель его завершает.

**3. Считать, что UDP всегда разрешён.** Некоторые сети блокируют или ограничивают UDP. Настоящие клиенты устраивают гонку QUIC против TCP или переходят на HTTP/2, если QUIC не отвечает быстро. Никогда не делайте публичный сервис доступным **только** по HTTP/3.

**4. Обращаться с датаграммами QUIC как с потоками.** Датаграммы (Упражнение 3) зашифрованы, но **ненадёжны**: они могут потеряться или прийти не по порядку и должны помещаться в один пакет. Используйте их для данных, где новое значение заменяет старое (позиции в игре, текущие измерения), а потоки — для всего, что обязано дойти.

## Попробуйте сами

Перехватите QUIC сами в Linux-контейнере (перехвату нужны Linux и root; никогда не делайте это на самом Mac). Из `code/p15-quic`:

```bash
docker run --rm -it --cap-add NET_RAW --cap-add NET_ADMIN -v "$PWD":/work -w /work rust:1.100-slim bash
```

Внутри контейнера (`CARGO_TARGET_DIR` не даёт Linux-сборке попасть в папку `target/` вашего Mac):

```bash
apt-get update -qq && apt-get install -y -qq --no-install-recommends tcpdump > /dev/null
export CARGO_TARGET_DIR=/tmp/target
cargo build -q --bin h3_get
tcpdump -i eth0 -n -c 8 udp port 443 &
sleep 1
/tmp/target/debug/h3_get
```

Вы увидите вывод программы и восемь UDP-строк, похожих на строки из раздела 4. Запустите ещё раз с `tcpdump … -X -c 1` и найдите в первом пакете `c1`, версию `00000001` и длину идентификатора соединения. Введите `exit`, чтобы выйти и удалить контейнер.

## Упражнения

### Упражнение 1 — Два медленных потока (легко)

Измените сообщения в `quic_echo` на `["slow stream one", "slow stream two", "stream three"]`. Предскажите порядок и время трёх ответов, затем запустите.

<details>
<summary>Подсказка</summary>

Сервер обрабатывает каждый поток в отдельной задаче, и каждый медленный поток ждёт 500 мс сам по себе.

</details>

<details>
<summary>Ответ</summary>

```text
server: listening on UDP 127.0.0.1:57831
client: connected after 2.950584ms
server: connection from 127.0.0.1:64932
client:   3.6ms "STREAM THREE"
client: 505.2ms "SLOW STREAM ONE"
client: 505.2ms "SLOW STREAM TWO"
```

Почему это работает: два медленных потока ждут **одновременно**, а не один за другим, поэтому оба завершаются примерно через 500 мс, а не через 1000 мс. Быстрый поток не задерживается ни одним из них. Потоки независимы и в сети, и в сервере.

</details>

### Упражнение 2 — Протестируйте потоки (средне)

Напишите `tests/quic.rs` с двумя тестами. Помощник запускает эхо-сервер (без медленного пути) на `127.0.0.1:0` и возвращает его адрес и сертификат. Тест 1: пять потоков в одном соединении, каждый получает обратно ровно своё сообщение в верхнем регистре. Тест 2: клиент, который доверяет сертификату *другого* сервера, не может подключиться.

<details>
<summary>Подсказка</summary>

Сделайте так, чтобы сервер в цикле вызывал `accept()`, тогда он сможет обслуживать несколько соединений. Для теста 2 запустите два сервера и подключитесь к первому, доверяя сертификату второго. Сообщение об ошибке содержит `invalid peer certificate`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::net::SocketAddr;
use std::sync::Arc;

use quinn::{ClientConfig, Endpoint, ServerConfig};
use rustls::RootCertStore;
use rustls::pki_types::{CertificateDer, PrivatePkcs8KeyDer};

/// Starts an upper-case echo server; returns its address and its certificate.
fn start_echo_server() -> (SocketAddr, CertificateDer<'static>) {
    let cert = rcgen::generate_simple_self_signed(vec!["localhost".to_string()]).unwrap();
    let cert_der = CertificateDer::from(cert.cert);
    let key_der = PrivatePkcs8KeyDer::from(cert.signing_key.serialize_der());
    let config = ServerConfig::with_single_cert(vec![cert_der.clone()], key_der.into()).unwrap();
    let server = Endpoint::server(config, "127.0.0.1:0".parse().unwrap()).unwrap();
    let addr = server.local_addr().unwrap();
    tokio::spawn(async move {
        while let Some(incoming) = server.accept().await {
            let Ok(conn) = incoming.await else { continue };
            while let Ok((mut send, mut recv)) = conn.accept_bi().await {
                tokio::spawn(async move {
                    let msg = recv.read_to_end(1024).await.unwrap();
                    send.write_all(&msg.to_ascii_uppercase()).await.unwrap();
                    send.finish().unwrap();
                });
            }
        }
    });
    (addr, cert_der)
}

/// A client endpoint that trusts exactly `cert`.
fn client_trusting(cert: CertificateDer<'static>) -> Endpoint {
    let mut roots = RootCertStore::empty();
    roots.add(cert).unwrap();
    let mut client = Endpoint::client("127.0.0.1:0".parse().unwrap()).unwrap();
    client
        .set_default_client_config(ClientConfig::with_root_certificates(Arc::new(roots)).unwrap());
    client
}

#[tokio::test]
async fn every_stream_gets_its_own_answer() {
    let (addr, cert) = start_echo_server();
    let conn = client_trusting(cert)
        .connect(addr, "localhost")
        .unwrap()
        .await
        .unwrap();
    let mut tasks = Vec::new();
    for i in 0..5 {
        let conn = conn.clone();
        tasks.push(tokio::spawn(async move {
            let (mut send, mut recv) = conn.open_bi().await.unwrap();
            send.write_all(format!("message {i}").as_bytes())
                .await
                .unwrap();
            send.finish().unwrap();
            let reply = recv.read_to_end(1024).await.unwrap();
            assert_eq!(reply, format!("MESSAGE {i}").into_bytes());
        }));
    }
    for task in tasks {
        task.await.unwrap();
    }
}

#[tokio::test]
async fn a_client_that_trusts_another_certificate_is_refused() {
    let (addr, _cert) = start_echo_server();
    let (_, other_cert) = start_echo_server();
    let result = client_trusting(other_cert)
        .connect(addr, "localhost")
        .unwrap()
        .await;
    let err = result.unwrap_err();
    assert!(
        err.to_string().contains("invalid peer certificate"),
        "{err}"
    );
}
```

```bash
cargo test --test quic
```

```text
running 2 tests
test a_client_that_trusts_another_certificate_is_refused ... ok
test every_stream_gets_its_own_answer ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

Почему это работает: каждая задача потока проверяет свой собственный ответ, поэтому путаница между потоками (ответ, пришедший не в тот поток) провалила бы тест. Отрицательный тест доказывает, что QUIC действительно проверяет сертификат: шифрование без проверки приняло бы не тот сервер.

</details>

### Упражнение 3 — Ненадёжные сообщения в надёжном соединении (сложно)

Напишите `src/bin/quic_datagrams.rs`: сервер, который принимает одно соединение и печатает три полученные QUIC-датаграммы, и клиент, который печатает наибольший допустимый размер датаграммы, а затем отправляет датаграммами `position update 1`–`3`.

<details>
<summary>Подсказка</summary>

`conn.send_datagram(bytes)` отправляет одну датаграмму (принимает `bytes::Bytes`; `String` преобразуется через `.into()`), а `conn.read_datagram().await` принимает одну. `conn.max_datagram_size()` говорит, насколько большой может быть датаграмма в этом соединении (`None`, если другая сторона их не поддерживает).

</details>

<details>
<summary>Ответ</summary>

```rust
use std::sync::Arc;

use quinn::{ClientConfig, Endpoint, ServerConfig};
use rustls::RootCertStore;
use rustls::pki_types::{CertificateDer, PrivatePkcs8KeyDer};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let cert = rcgen::generate_simple_self_signed(vec!["localhost".to_string()])?;
    let cert_der = CertificateDer::from(cert.cert);
    let key_der = PrivatePkcs8KeyDer::from(cert.signing_key.serialize_der());

    let server_config = ServerConfig::with_single_cert(vec![cert_der.clone()], key_der.into())?;
    let server = Endpoint::server(server_config, "127.0.0.1:0".parse()?)?;
    let server_addr = server.local_addr()?;
    let receiver = tokio::spawn(async move {
        let conn = server.accept().await.unwrap().await.unwrap();
        for _ in 0..3 {
            let datagram = conn.read_datagram().await.unwrap();
            println!(
                "server got datagram: {:?}",
                String::from_utf8_lossy(&datagram)
            );
        }
    });

    let mut roots = RootCertStore::empty();
    roots.add(cert_der)?;
    let mut client = Endpoint::client("127.0.0.1:0".parse()?)?;
    client.set_default_client_config(ClientConfig::with_root_certificates(Arc::new(roots))?);
    let conn = client.connect(server_addr, "localhost")?.await?;

    println!("largest datagram: {:?} bytes", conn.max_datagram_size());
    for i in 1..=3 {
        conn.send_datagram(format!("position update {i}").into())?;
    }
    receiver.await?;
    conn.close(0u32.into(), b"done");
    client.wait_idle().await;
    Ok(())
}
```

```text
largest datagram: Some(1162) bytes
server got datagram: "position update 1"
server got datagram: "position update 2"
server got datagram: "position update 3"
```

Почему это работает: датаграммы идут внутри того же зашифрованного соединения, что и потоки, но QUIC никогда не отправляет их повторно. Датаграмма должна помещаться в один пакет: здесь 1162 байта — размер пакета минус заголовки QUIC. На `127.0.0.1` ничего не теряется; в настоящей сети сервер не должен ждать ровно трёх, потому что некоторые могут так и не прийти.

</details>

## Вопросы для самопроверки

1. Что такое блокировка начала очереди в HTTP/2 поверх TCP и как QUIC её избегает?

<details>
<summary>Ответ</summary>

TCP доставляет байты строго по порядку, поэтому один потерянный пакет задерживает все последующие байты соединения, включая байты других запросов HTTP/2. В QUIC независимые потоки: потерянный пакет задерживает только те потоки, чьи данные он нёс.

</details>

2. Почему QUIC построен поверх UDP, а не сделан новым протоколом рядом с TCP?

<details>
<summary>Ответ</summary>

Почти любая сеть пропускает UDP, а новый IP-протокол заблокировали бы многие маршрутизаторы и файрволы. Кроме того, UDP позволяет QUIC жить в библиотеке, а не в ядре, поэтому его можно быстро улучшать.

</details>

3. Сколько кругов туда-обратно клиент ждёт, прежде чем сможет отправить HTTP-запрос по TCP + TLS 1.3 и по QUIC?

<details>
<summary>Ответ</summary>

Два по TCP + TLS 1.3 (один для TCP, один для TLS); один по QUIC, который выполняет оба рукопожатия вместе. При возобновлении прежней сессии QUIC может отправить данные даже в первом пакете (0-RTT).

</details>

4. Что сообщает браузеру `alt-svc: h3=":443"; ma=86400`?

<details>
<summary>Ответ</summary>

Что тот же сайт доступен ещё и по HTTP/3 на UDP-порту 443 и что браузер может помнить это 86 400 секунд (сутки). Для следующих запросов браузер попробует QUIC.

</details>

5. Что произойдёт, если отправитель QUIC-потока так и не вызовет `finish()`, а получатель использует `read_to_end`?

<details>
<summary>Ответ</summary>

Получатель продолжит ждать конца потока, который никогда не наступит. Если отправитель в свою очередь ждёт ответа, оба ждут вечно.

</details>

6. Зачем QUIC-клиент дополняет свой первый пакет до 1200 байт?

<details>
<summary>Ответ</summary>

Чтобы предотвратить атаки усиления (amplification): иначе злоумышленник с поддельным адресом источника мог бы отправить крошечный пакет и заставить сервер отправить жертве гораздо больший ответ. С дополнением первый ответ сервера ненамного больше того, что он получил.

</details>

## Итоги

- Единственный упорядоченный поток TCP вызывает блокировку начала очереди для HTTP/2, а TCP + TLS требует двух кругов туда-обратно до данных; QUIC исправляет и то и другое.
- QUIC работает поверх UDP, шифрует TLS 1.3 (ALPN обязателен), несёт много независимых потоков и определяет соединения по идентификаторам соединений.
- В `quinn` `Endpoint` — это один UDP-сокет; `connect`/`accept` дают соединения; `open_bi`/`accept_bi` дают потоки; `finish()` завершает поток.
- На проводе QUIC — просто UDP на порт 443; первый пакет дополнен до 1200 байт, и почти всё зашифровано.
- HTTP/3 отображает каждый запрос на свой QUIC-поток; серверы объявляют его через `Alt-Svc`; `h3` вместе с `h3-quinn` — реализация на Rust.
- Датаграммы QUIC зашифрованы, но ненадёжны; используйте их только для данных, которые можно потерять.

## Источники и что почитать

- [RFC 9000 — QUIC: A UDP-Based Multiplexed and Secure Transport](https://www.rfc-editor.org/rfc/rfc9000) — сам протокол QUIC; раздел 14.1 объясняет правило 1200 байт.
- [RFC 9114 — HTTP/3](https://www.rfc-editor.org/rfc/rfc9114) — как HTTP отображается на потоки QUIC.
- [Документация quinn](https://docs.rs/quinn/latest/quinn/) — `Endpoint`, `Connection`, потоки и датаграммы.
- [h3 на GitHub](https://github.com/hyperium/h3) — крейт HTTP/3 с примерами клиента и сервера.
- [RFC 9221 — An Unreliable Datagram Extension to QUIC](https://www.rfc-editor.org/rfc/rfc9221) — датаграммы из Упражнения 3.
- [High Performance Browser Networking](https://hpbn.co/) — бесплатная книга о задержках, TCP и HTTP/2: фон, объясняющий, зачем появился QUIC.

---

[🏠 Начало](../START-HERE.md) · [← 15.12 TLS в серверах на Rust с rustls](12-tls-in-rust-servers.md) · [📋 Этап 15](README.md) · [15.14 Прокси и балансировщики нагрузки →](14-proxies-and-load-balancers.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
