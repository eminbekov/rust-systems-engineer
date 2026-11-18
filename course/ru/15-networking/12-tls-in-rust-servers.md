<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 15.11 Проект: DNS-резолвер](11-project-dns-resolver.md) · [📋 Этап 15](README.md) · [15.13 QUIC и HTTP/3 →](13-quic-and-http3.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/15-networking/12-tls-in-rust-servers.md) <!-- nav -->

# Урок 15.12 — TLS в серверах на Rust с rustls

> **Этап 15 — Сети** · Урок 12 из 16 · ⏱ около 135 минут

## Что вы узнаете

- Что нужно TLS-серверу: цепочка сертификатов, закрытый ключ и конфигурация — и что нужно клиенту, чтобы ему доверять.
- Как создать маленький удостоверяющий центр и сертификаты для локальных тестов с помощью `rcgen`.
- Как принимать TLS-соединения в сервере на Tokio с помощью `tokio-rustls` и как увидеть SNI и ALPN на стороне сервера.
- Как написать клиент на Rust, который доверяет ровно вашему собственному CA, и что означают его ошибки.
- Как работает взаимный TLS: сервер проверяет и сертификат клиента.

## Перед началом

- Вы прошли [Урок 15.11 — Проект: DNS-резолвер](11-project-dns-resolver.md).
- Из [Урока 10.11 — TLS и сертификаты](../10-security-fundamentals/11-tls-and-certificates.md) вы знаете, что защищает TLS (Transport Layer Security, протокол защиты транспортного уровня), что делает рукопожатие, как устроена цепочка доверия и как *клиент* `rustls` проверяет сервер. Этот урок это не повторяет; он строит **серверную** сторону. Держите OpenSSL из Homebrew из того урока первым в `PATH`.
- Вы знаете цикл приёма соединений Tokio из [Урока 15.6 — Асинхронная работа с сетью в Tokio](06-async-networking-with-tokio.md).
- Создайте учебный пакет. Ему нужны файлы сертификатов, которые никогда не должны попасть в Git:

```bash
cd ~/projects/rust
cargo new code/p15-tls
cd code/p15-tls
mkdir -p src/bin && rm src/main.rs
echo "certs/" >> .gitignore
cargo add tokio@1 --features full
cargo add rustls@0.23 --no-default-features --features ring,std,tls12
cargo add tokio-rustls@0.26 --no-default-features --features ring,tls12
cargo add rcgen@0.14 time@0.3
```

Строки `--no-default-features --features ring,…` выбирают для обоих крейтов один криптографический бэкенд, `ring`, как в Уроке 10.11. В «Частых ошибках» показано, что происходит, когда в одной программе оказываются два бэкенда.

## Новые слова

| Слово | Значение |
|---|---|
| **TLS** (Transport Layer Security, протокол защиты транспортного уровня) | Протокол, который шифрует и аутентифицирует соединение (Урок 10.11). |
| **TCP / HTTP / HTTPS** (Transmission Control Protocol / HyperText Transfer Protocol / HTTP Secure) | Надёжный поток байтов под TLS / веб-протокол из Урока 15.9 / HTTP внутри TLS. |
| **CA** (Certificate Authority, удостоверяющий центр) | Тот, кто подписывает сертификаты. Здесь вы сами — свой маленький CA, которому доверяют только ваши тестовые клиенты. |
| **цепочка сертификатов** (certificate chain) | Собственный (конечный, leaf) сертификат сервера и следующие за ним промежуточные сертификаты, которые сервер отправляет во время рукопожатия. |
| **закрытый ключ** (private key) | Секретная половина пары ключей сервера. Он доказывает, что сертификат принадлежит серверу. Он никогда не покидает сервер. |
| **PEM** (Privacy-Enhanced Mail) | Текстовый формат для сертификатов и ключей между строками `-----BEGIN …-----` (Урок 10.11). |
| **SAN** (Subject Alternative Name, альтернативное имя субъекта) | Список имён и IP-адресов (Internet Protocol, межсетевой протокол), для которых действителен сертификат. |
| **SNI** (Server Name Indication, указание имени сервера) | Поле в первом сообщении рукопожатия клиента, где сказано, какое имя хоста ему нужно. Тогда один сервер может выбирать правильный сертификат для многих имён. |
| **ALPN** (Application-Layer Protocol Negotiation, согласование протокола прикладного уровня) | Поле рукопожатия, в котором клиент предлагает протоколы (`h2`, `http/1.1`), а сервер выбирает один. |
| **acceptor / connector** | Объекты `tokio-rustls`, которые превращают `TcpStream` в TLS-поток на стороне сервера / на стороне клиента. |
| **взаимный TLS** (mutual TLS, mTLS) | TLS, в котором клиент тоже предъявляет сертификат, а сервер его проверяет. Распространён между внутренними сервисами. |
| **криптопровайдер** (crypto provider) | Библиотека, которая выполняет саму криптографию для `rustls`: `ring` или `aws-lc-rs`. |
| **CN / DER / PKI** (Common Name / Distinguished Encoding Rules / public key infrastructure) | Главное поле имени в сертификате / двоичная кодировка сертификатов / инфраструктура открытых ключей: всё вокруг сертификатов — CA, ключи и доверие. |
| **ECDSA** (Elliptic Curve Digital Signature Algorithm, алгоритм цифровой подписи на эллиптических кривых) | Алгоритм подписи ключей, которые `rcgen` создаёт по умолчанию. |
| **close_notify** | Сообщение TLS «я закончил отправку». Если сторона закрывает соединение без него, её мог оборвать злоумышленник. |
| **URL** (Uniform Resource Locator, единообразный указатель ресурса) | Веб-адрес вроде `https://localhost:8443/`: схема, хост, порт и путь. |

## 1. Что нужно TLS-серверу

В Уроке 10.11 проверял клиент: подписан ли сертификат корнем, которому я доверяю, действителен ли он ещё, есть ли моё имя хоста в его SAN? Чтобы эти проверки прошли, **сервер** должен предоставить три вещи:

```text
            server side                                        client side
 ┌───────────────────────────────┐                 ┌─────────────────────────────┐
 │ certificate chain (server.pem)│ ── sent in ───► │ trusted roots (ca.pem)      │
 │ private key       (server.key)│    handshake    │ expected name ("localhost") │
 │ config: versions, ALPN, mTLS  │                 │ config: versions, ALPN      │
 └───────────────────────────────┘                 └─────────────────────────────┘
        the key never leaves              the client checks: signature chain → root,
        the server                         dates, name in SAN
```

В публичном интернете ваш сертификат подписывает публичный CA (например, Let's Encrypt), и каждый клиент уже доверяет этому CA. Для локальной разработки и тестов вы не хотите трогать системные хранилища доверенных сертификатов. Вместо этого вы создаёте **собственный CA**, подписываете им серверный сертификат и отдаёте сертификат CA только своим тестовым клиентам.

> 🔒 **Безопасность:** закрытый ключ — это личность вашего сервера. Тот, у кого он есть, может выдавать себя за ваш сервер. Держите файлы ключей вне Git (`certs/` есть в `.gitignore`), доступными для чтения только пользователю сервиса и вне образов контейнеров; в продакшене они приходят из хранилища секретов (Урок 10.8).

## 2. Локальный CA и сертификаты с rcgen

`rcgen` создаёт сертификаты в коде на Rust. Создайте `src/bin/certgen.rs`:

```rust
use std::fs;

use rcgen::{
    BasicConstraints, CertificateParams, CertifiedIssuer, DnType, IsCa, KeyPair, KeyUsagePurpose,
};
use time::{Duration, OffsetDateTime};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    fs::create_dir_all("certs")?;
    let now = OffsetDateTime::now_utc();

    // 1. Our own certificate authority (CA). It signs the other two certificates.
    let mut ca_params = CertificateParams::new(Vec::<String>::new())?;
    ca_params
        .distinguished_name
        .push(DnType::CommonName, "p15 test CA");
    ca_params.is_ca = IsCa::Ca(BasicConstraints::Unconstrained);
    ca_params.key_usages = vec![KeyUsagePurpose::KeyCertSign];
    ca_params.not_before = now;
    ca_params.not_after = now + Duration::days(30);
    let ca = CertifiedIssuer::self_signed(ca_params, KeyPair::generate()?)?;

    // 2. The server certificate: valid for the name "localhost" and the IP 127.0.0.1.
    let mut server =
        CertificateParams::new(vec!["localhost".to_string(), "127.0.0.1".to_string()])?;
    server
        .distinguished_name
        .push(DnType::CommonName, "localhost");
    server.not_before = now;
    server.not_after = now + Duration::days(7);
    let server_key = KeyPair::generate()?;
    let server_cert = server.signed_by(&server_key, &ca)?;

    // 3. A client certificate, used for mutual TLS in Section 5.
    let mut client = CertificateParams::new(Vec::<String>::new())?;
    client
        .distinguished_name
        .push(DnType::CommonName, "client-1");
    client.not_before = now;
    client.not_after = now + Duration::days(7);
    let client_key = KeyPair::generate()?;
    let client_cert = client.signed_by(&client_key, &ca)?;

    fs::write("certs/ca.pem", ca.pem())?;
    fs::write("certs/server.pem", server_cert.pem())?;
    fs::write("certs/server.key", server_key.serialize_pem())?;
    fs::write("certs/client.pem", client_cert.pem())?;
    fs::write("certs/client.key", client_key.serialize_pem())?;
    println!("wrote 5 files to certs/");
    Ok(())
}
```

**Что происходит:**

1. `CertificateParams` описывает сертификат: имена (SAN), отличительное имя субъекта (здесь только CN), даты действия и является ли он CA. `KeyPair::generate()` создаёт новую пару ключей (по умолчанию ECDSA на кривой P-256).
2. CA **самоподписанный**: `CertifiedIssuer::self_signed` подписывает собственный сертификат CA его же ключом и сохраняет ключ, чтобы подписывать дальше. `IsCa::Ca` и назначение `KeyCertSign` говорят: «этот сертификат может подписывать другие сертификаты».
3. `server.signed_by(&server_key, &ca)` создаёт серверный сертификат для открытого ключа `server_key` и подписывает его ключом CA. `CertificateParams::new` сам превращает `"127.0.0.1"` в IP-SAN, а `"localhost"` — в DNS-SAN (Domain Name System, система доменных имён).
4. Короткие сроки (7 и 30 дней) выбраны намеренно: тестовые сертификаты, которые скоро истекают, мало навредят, если утекут.
5. `.pem()` и `.serialize_pem()` записывают сертификаты и ключи в виде PEM-текста.

```bash
cargo run -q --bin certgen
openssl x509 -in certs/server.pem -noout -subject -issuer -dates -ext subjectAltName
```

```text
wrote 5 files to certs/
subject=CN=localhost
issuer=CN=p15 test CA
notBefore=Nov 18 07:36:26 2026 GMT
notAfter=Nov 25 07:36:26 2026 GMT
X509v3 Subject Alternative Name: 
    DNS:localhost, IP Address:127.0.0.1
```

Субъект и издатель теперь **разные**: это настоящая цепочка из двух звеньев (конечный сертификат → наш CA), а не самоподписанный конечный сертификат из Урока 10.11.

## 3. TLS-сервер на tokio-rustls

Сам `rustls` работает с любым потоком байтов. `tokio-rustls` соединяет его с Tokio: `TlsAcceptor` берёт `TcpStream`, выполняет рукопожатие и возвращает `TlsStream`, из которого читают и в который пишут, как в обычный поток. Создайте `src/bin/tls_server.rs`:

```rust
use std::net::SocketAddr;
use std::sync::Arc;

use rustls::pki_types::pem::PemObject;
use rustls::pki_types::{CertificateDer, PrivateKeyDer};
use rustls::server::WebPkiClientVerifier;
use rustls::{RootCertStore, ServerConfig};
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};
use tokio_rustls::TlsAcceptor;
use tokio_rustls::server::TlsStream;

fn load_config(mutual: bool) -> Result<ServerConfig, Box<dyn std::error::Error>> {
    let certs =
        CertificateDer::pem_file_iter("certs/server.pem")?.collect::<Result<Vec<_>, _>>()?;
    let key = PrivateKeyDer::from_pem_file("certs/server.key")?;
    let builder = ServerConfig::builder();
    let mut config = if mutual {
        let mut roots = RootCertStore::empty();
        roots.add(CertificateDer::from_pem_file("certs/ca.pem")?)?;
        let verifier = WebPkiClientVerifier::builder(Arc::new(roots)).build()?;
        builder
            .with_client_cert_verifier(verifier)
            .with_single_cert(certs, key)?
    } else {
        builder.with_no_client_auth().with_single_cert(certs, key)?
    };
    config.alpn_protocols = vec![b"http/1.1".to_vec()];
    Ok(config)
}

async fn handle(mut tls: TlsStream<TcpStream>, peer: SocketAddr) -> std::io::Result<()> {
    let (_, conn) = tls.get_ref();
    let alpn = conn
        .alpn_protocol()
        .map(|p| String::from_utf8_lossy(p).into_owned());
    let client = conn.peer_certificates().map(|chain| chain.len());
    println!(
        "{peer}: {:?}, SNI {:?}, ALPN {:?}, client certificates: {:?}",
        conn.protocol_version().unwrap(),
        conn.server_name(),
        alpn,
        client
    );

    let mut buf = [0u8; 1024];
    let _ = tls.read(&mut buf).await?; // the request; we answer every request the same way
    let body = "hello over TLS\n";
    let response = format!(
        "HTTP/1.1 200 OK\r\ncontent-length: {}\r\nconnection: close\r\n\r\n{body}",
        body.len()
    );
    tls.write_all(response.as_bytes()).await?;
    tls.shutdown().await
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mutual = std::env::args().any(|a| a == "--mtls");
    let acceptor = TlsAcceptor::from(Arc::new(load_config(mutual)?));
    let listener = TcpListener::bind("127.0.0.1:8443").await?;
    println!("listening on https://localhost:8443 (mutual TLS: {mutual})");

    loop {
        let (tcp, peer) = listener.accept().await?;
        let acceptor = acceptor.clone();
        tokio::spawn(async move {
            match acceptor.accept(tcp).await {
                Ok(tls) => {
                    if let Err(err) = handle(tls, peer).await {
                        eprintln!("{peer}: {err}");
                    }
                }
                Err(err) => eprintln!("{peer}: handshake failed: {err}"),
            }
        });
    }
}
```

**Что происходит:**

1. `load_config` читает PEM-файлы. Трейт `PemObject` (из `rustls::pki_types`) даёт `pem_file_iter`, который выдаёт все сертификаты из файла (в цепочке их может быть несколько), и `from_pem_file` для одного элемента, например ключа.
2. `ServerConfig::builder()` начинает с безопасных значений по умолчанию: TLS 1.3 и 1.2, современные наборы шифров, единственный установленный криптопровайдер. `with_no_client_auth()` означает «подключиться может кто угодно» (нормально для публичного сервера); `with_single_cert` задаёт цепочку и ключ и проверяет, что они подходят друг к другу. Ветка `mutual` — для раздела 5.
3. `alpn_protocols` перечисляет, на чём говорит этот сервер. Если клиент предлагает ALPN и ничего не совпадает, рукопожатие завершается ошибкой — это лучше, чем говорить не на том протоколе.
4. `ServerConfig` обёрнут в `Arc`, потому что его делят все соединения. `TlsAcceptor` дёшево клонировать: каждая задача получает свой дескриптор.
5. `acceptor.accept(tcp).await` выполняет всё рукопожатие. Только если оно удалось, вы получаете `TlsStream`; неудачное рукопожатие записывается в лог, и завершается только это соединение.
6. `tls.get_ref()` возвращает TCP-поток и состояние соединения `rustls`, из которого мы читаем версию TLS, присланный клиентом SNI, согласованный протокол ALPN и сертификаты клиента.
7. `tls.shutdown()` отправляет `close_notify` и затем закрывает соединение. Без этого клиент не может отличить чистое завершение от оборванного соединения (см. «Частые ошибки»).

Запустите его во втором терминале из папки пакета (пути `certs/…` отсчитываются от неё):

```bash
cargo run -q --bin tls_server
```

```text
listening on https://localhost:8443 (mutual TLS: false)
```

Вернитесь в первый терминал и попробуйте `curl` без нашего CA и с ним:

```bash
curl -sS https://localhost:8443/ 2>&1 | head -1
curl -sS --cacert certs/ca.pem https://localhost:8443/
curl -sv --cacert certs/ca.pem https://localhost:8443/ 2>&1 | grep -E 'SSL connection|ALPN|subject:|issuer:|verify ok'
```

```text
curl: (60) SSL certificate problem: unable to get local issuer certificate
hello over TLS
* ALPN: curl offers h2,http/1.1
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256 / [blank] / UNDEF
* ALPN: server accepted http/1.1
*  subject: CN=localhost
*  issuer: CN=p15 test CA
*  SSL certificate verify ok.
```

**Что происходит:** без `--cacert` `curl` использует системное хранилище доверенных сертификатов, где нашего CA нет («unable to get local issuer certificate»). С `--cacert certs/ca.pem` цепочка ведёт к доверенному корню, а имя `localhost` есть в SAN, поэтому проверка успешна. `curl` предложил `h2` и `http/1.1`; наш сервер выбрал `http/1.1`. (`curl` в macOS собран с LibreSSL, которая печатает набор шифров по-своему.)

В терминале сервера появляется по строке на каждое соединение:

```text
127.0.0.1:52161: handshake failed: received fatal alert: UnknownCA
127.0.0.1:52163: TLSv1_3, SNI Some("localhost"), ALPN Some("http/1.1"), client certificates: None
127.0.0.1:52165: TLSv1_3, SNI Some("localhost"), ALPN Some("http/1.1"), client certificates: None
```

Первый `curl` отверг сертификат и перед закрытием сообщил серверу причину **предупреждением** (alert) `UnknownCA`. Оставьте сервер работать.

> 💡 **Совет:** чтобы обслуживать маршрутизатор `hyper` из Урока 15.9 через TLS, сначала примите TLS-поток, а затем передайте `TokioIo::new(tls_stream)` в `serve_connection`. `hyper` всё равно, зашифрованы ли байты внизу. Задайте `alpn_protocols` как `h2` и `http/1.1`, если вы обслуживаете ещё и HTTP/2.

## 4. Клиент на Rust, который доверяет вашему CA

Создайте `src/bin/tls_client.rs`. Это клиент из Урока 10.11, но асинхронный и с **одним** доверенным корнем — нашим CA.

```rust
use std::sync::Arc;

use rustls::pki_types::pem::PemObject;
use rustls::pki_types::{CertificateDer, ServerName};
use rustls::{ClientConfig, RootCertStore};
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::TcpStream;
use tokio_rustls::TlsConnector;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let name = std::env::args()
        .nth(1)
        .unwrap_or_else(|| "localhost".to_string());

    // Trust exactly one root: our own CA.
    let mut roots = RootCertStore::empty();
    roots.add(CertificateDer::from_pem_file("certs/ca.pem")?)?;
    let config = ClientConfig::builder()
        .with_root_certificates(roots)
        .with_no_client_auth();
    let connector = TlsConnector::from(Arc::new(config));

    let tcp = TcpStream::connect("127.0.0.1:8443").await?;
    let server_name = ServerName::try_from(name.clone())?;
    let mut tls = connector.connect(server_name, tcp).await?;

    tls.write_all(format!("GET / HTTP/1.1\r\nHost: {name}\r\n\r\n").as_bytes())
        .await?;
    let mut reply = String::new();
    tls.read_to_string(&mut reply).await?;
    println!("{}", reply.lines().next().unwrap_or(""));
    println!("{}", reply.lines().last().unwrap_or(""));
    Ok(())
}
```

**Что происходит:**

1. `RootCertStore::empty()` плюс один `add` означает: не доверять ничему, кроме нашего CA. Это безопасный способ говорить с внутренним сервисом, у которого частный CA.
2. Программа всегда подключается к `127.0.0.1:8443`, но **имя, которое она проверяет**, берётся из командной строки. `ServerName` — это одновременно имя, которое должно быть в SAN, и SNI, который отправляет клиент.
3. `connector.connect(name, tcp)` выполняет рукопожатие; любая ошибка проверки возвращается здесь, до отправки хотя бы одного байта запроса.

```bash
cargo run -q --bin tls_client
cargo run -q --bin tls_client -- 127.0.0.1
cargo run -q --bin tls_client -- other.test
```

```text
HTTP/1.1 200 OK
hello over TLS
HTTP/1.1 200 OK
hello over TLS
Error: Custom { kind: InvalidData, error: InvalidCertificate(NotValidForNameContext { expected: DnsName("other.test"), presented: ["DnsName(\"localhost\")", "IpAddress(127.0.0.1)"] }) }
```

**Что происходит:** и `localhost`, и `127.0.0.1` есть в SAN, поэтому оба работают. Для `other.test` цепочка в порядке, а имя — нет, и ошибка перечисляет ровно то, что предлагает сертификат. Лог сервера показывает разницу со своей стороны:

```text
127.0.0.1:52166: TLSv1_3, SNI Some("localhost"), ALPN None, client certificates: None
127.0.0.1:52167: TLSv1_3, SNI None, ALPN None, client certificates: None
127.0.0.1:52168: handshake failed: received fatal alert: BadCertificate
```

`SNI None` для `127.0.0.1`: SNI может нести только имена хостов, никогда не IP-адреса. `ALPN None`: наш клиент не предлагал ALPN, и это допустимо. Остановите сервер через `Ctrl+C`.

## 5. Взаимный TLS

Обычно только сервер доказывает, кто он. Во **взаимном TLS** клиент тоже отправляет сертификат, и сервер принимает только тех клиентов, чей сертификат подписан доверенным для него CA. Между внутренними сервисами это заменяет пароли и ключи API (Application Programming Interface, программный интерфейс): сертификат *и есть* личность.

Ветка `mutual` в `load_config` делает именно это: `WebPkiClientVerifier::builder(roots)` строит проверяющий объект, который сверяет сертификаты клиентов с нашим CA, а `with_client_cert_verifier` делает его обязательным. Запустите сервер с `--mtls`:

```bash
cargo run -q --bin tls_server -- --mtls
```

Затем обратитесь к нему без сертификата клиента и с ним:

```bash
curl -sS --cacert certs/ca.pem https://localhost:8443/
curl -sS --cacert certs/ca.pem --cert certs/client.pem --key certs/client.key https://localhost:8443/
cargo run -q --bin tls_client
```

```text
curl: (56) LibreSSL SSL_read: LibreSSL/3.3.6: error:1404C45C:SSL routines:ST_OK:reason(1116), errno 0
hello over TLS
Error: Custom { kind: InvalidData, error: AlertReceived(CertificateRequired) }
```

И сервер:

```text
listening on https://localhost:8443 (mutual TLS: true)
127.0.0.1:52139: handshake failed: peer sent no certificates
127.0.0.1:52141: TLSv1_3, SNI Some("localhost"), ALPN Some("http/1.1"), client certificates: Some(1)
127.0.0.1:52142: handshake failed: peer sent no certificates
```

**Что происходит:** без сертификата клиента сервер завершает рукопожатие предупреждением `CertificateRequired`. Сообщение LibreSSL загадочно (reason 1116 — это и есть то предупреждение); клиент `rustls` называет его прямо. С `--cert` и `--key` `curl` доказывает, что владеет ключом `client-1`, и сервер видит один сертификат клиента. В настоящем сервисе вы бы теперь прочитали имя клиента из этого сертификата и решили, что ему разрешено (авторизация, Урок 10.7).

> ⚡ **Производительность:** полное рукопожатие стоит одного сетевого круга туда-обратно (TLS 1.3) и некоторого объёма вычислений с открытыми ключами на обеих сторонах; после этого шифрование дёшево. Держите соединения открытыми и переиспользуйте их (keep-alive, пулы соединений). `rustls` также поддерживает *возобновление сессии* (session resumption), которое позволяет вернувшемуся клиенту пропустить бо́льшую часть работы рукопожатия.

## Частые ошибки

**1. Два криптопровайдера в одной программе.** Если какая-нибудь зависимость включает у `rustls` второй провайдер (`aws-lc-rs` — провайдер по умолчанию во многих крейтах, например в `reqwest`), `rustls` больше не знает, какой использовать, и `ServerConfig::builder()` паникует во время выполнения:

```text
thread 'main' (5758865) panicked at ~/.cargo/registry/src/…/rustls-0.23.46/src/crypto/mod.rs:249:14:

Could not automatically determine the process-level CryptoProvider from Rustls crate features.
Call CryptoProvider::install_default() before this point to select a provider manually, or make sure exactly one of the 'aws-lc-rs' and 'ring' features is enabled.
```

Исправление: либо оставьте во всём дереве зависимостей ровно одну фичу провайдера (`cargo tree -e features -i rustls` показывает, кто что включает), либо явно выберите провайдер в начале `main`:

```rust
    rustls::crypto::ring::default_provider()
        .install_default()
        .expect("no other provider was installed before");
```

**2. Закрыть соединение без `close_notify`.** Если сервер завершает `handle` после `write_all` без `tls.shutdown().await`, `curl` всё равно печатает тело, а клиент `rustls` отвергает данные:

```text
Error: Custom { kind: UnexpectedEof, error: "peer closed connection without sending TLS close_notify: https://docs.rs/rustls/latest/rustls/manual/_03_howto/index.html#unexpected-eof" }
```

Иначе злоумышленник, способный обрывать TCP-соединения, мог бы выдать обрезанный ответ за полный. Всегда завершайте TLS-поток через shutdown, когда закончили писать.

**3. Говорить простым HTTP с TLS-портом.** `curl http://localhost:8443/` получает `curl: (1) Received HTTP/0.9 when not allowed` (он не смог понять байты TLS), а сервер пишет в лог `handshake failed: received corrupt message of type InvalidContentType`. Увидев любое из этих сообщений, проверьте схему URL.

**4. Запустить сервер не из той папки.** Пути `certs/…` отсчитываются от текущего каталога. Запущенный в другом месте, сервер сразу останавливается с `Error: Io(Os { code: 2, kind: NotFound, message: "No such file or directory" })`. Настоящие сервисы берут пути из конфигурации (Урок 16.13) и печатают, какой файл не удалось прочитать.

## Упражнения

### Упражнение 1 — Ещё одно имя (легко)

Сделайте серверный сертификат действительным и для `api.localhost`, пересоздайте файлы, перезапустите сервер и проверьте через `curl`, что `api.localhost` работает, а `www.localhost` — нет. `curl --resolve api.localhost:8443:127.0.0.1` на одну команду отправляет это имя на `127.0.0.1`.

<details>
<summary>Подсказка</summary>

Имена берутся из `vec![…]`, который передаётся в `CertificateParams::new` для сервера.

</details>

<details>
<summary>Ответ</summary>

```rust
    let mut server = CertificateParams::new(vec![
        "localhost".to_string(),
        "api.localhost".to_string(),
        "127.0.0.1".to_string(),
    ])?;
```

```bash
cargo run -q --bin certgen
cargo run -q --bin tls_server      # in the second terminal, after Ctrl+C
curl -sS --cacert certs/ca.pem --resolve api.localhost:8443:127.0.0.1 https://api.localhost:8443/
curl -sS --cacert certs/ca.pem --resolve www.localhost:8443:127.0.0.1 https://www.localhost:8443/ 2>&1 | head -1
```

```text
wrote 5 files to certs/
hello over TLS
curl: (60) SSL: no alternative certificate subject name matches target host name 'www.localhost'
```

Почему это работает: клиент сверяет своё имя со списком SAN. Пересоздание заодно создало **новый CA**, поэтому каждый клиент должен использовать новый `ca.pem`; если бы сервер продолжал работать со старыми файлами, соединение теперь завершалось бы ошибкой `BadSignature` (то же имя CA, другой ключ).

</details>

### Упражнение 2 — Клиент с сертификатом (средне)

Напишите `src/bin/tls_client_cert.rs`: клиент из раздела 4, который дополнительно предъявляет `certs/client.pem` с `certs/client.key`. Запустите его против `tls_server -- --mtls` и посмотрите в лог сервера.

<details>
<summary>Подсказка</summary>

Замените `.with_no_client_auth()` на `.with_client_auth_cert(chain, key)?`, где `chain` получен из `CertificateDer::pem_file_iter`, а `key` — из `PrivateKeyDer::from_pem_file`.

</details>

<details>
<summary>Ответ</summary>

Скопируйте `src/bin/tls_client.rs` в `src/bin/tls_client_cert.rs`. Добавьте `PrivateKeyDer` в строку `use rustls::pki_types::{…}` и замените создание `config` на:

```rust
    let chain =
        CertificateDer::pem_file_iter("certs/client.pem")?.collect::<Result<Vec<_>, _>>()?;
    let key = PrivateKeyDer::from_pem_file("certs/client.key")?;
    let config = ClientConfig::builder()
        .with_root_certificates(roots)
        .with_client_auth_cert(chain, key)?;
```

Если сервер запущен как `cargo run -q --bin tls_server -- --mtls`, то `cargo run -q --bin tls_client_cert` печатает:

```text
HTTP/1.1 200 OK
hello over TLS
```

Сервер печатает `127.0.0.1:52143: TLSv1_3, SNI Some("localhost"), ALPN None, client certificates: Some(1)`.

Почему это работает: во время рукопожатия сервер запрашивает сертификат; клиент отправляет сертификат `client-1` и подписью доказывает, что у него есть соответствующий закрытый ключ. Проверяющий объект сервера убеждается, что сертификат подписан нашим CA.

</details>

### Упражнение 3 — Тестировать TLS без файлов (сложно)

Напишите `tests/tls.rs` с двумя тестами, которым не нужны ни файлы, ни фиксированный порт: создайте CA и серверный сертификат **в памяти** через `rcgen`, запустите на `127.0.0.1:0` TLS-сервер, который пишет `hello` и завершает соединение, и затем (1) клиент, доверяющий этому CA, получает `hello`; (2) клиент, доверяющий *другому* CA, получает ошибку, в которой упоминается `UnknownIssuer`.

<details>
<summary>Подсказка</summary>

`ca.der()` даёт сертификат CA в DER (двоичной форме) для `RootCertStore::add`; `cert.der().clone()` даёт серверный сертификат. Ключ: `PrivatePkcs8KeyDer::from(key.serialize_der()).into()`. Дайте двум CA разные общие имена (common name); при одинаковых именах `rustls` вместо этого сообщает `BadSignature`.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::net::SocketAddr;
use std::sync::Arc;

use rcgen::{BasicConstraints, CertificateParams, CertifiedIssuer, DnType, IsCa, KeyPair};
use rustls::pki_types::{PrivatePkcs8KeyDer, ServerName};
use rustls::{ClientConfig, RootCertStore, ServerConfig};
use tokio::io::{AsyncReadExt, AsyncWriteExt};
use tokio::net::{TcpListener, TcpStream};
use tokio_rustls::{TlsAcceptor, TlsConnector};

/// A new test CA that lives only in memory.
fn new_ca(name: &str) -> CertifiedIssuer<'static, KeyPair> {
    let mut params = CertificateParams::new(Vec::<String>::new()).unwrap();
    params.distinguished_name.push(DnType::CommonName, name);
    params.is_ca = IsCa::Ca(BasicConstraints::Unconstrained);
    CertifiedIssuer::self_signed(params, KeyPair::generate().unwrap()).unwrap()
}

/// Starts a TLS server for "localhost" signed by `ca`; it says "hello" to every client.
async fn start_server(ca: &CertifiedIssuer<'static, KeyPair>) -> SocketAddr {
    let key = KeyPair::generate().unwrap();
    let params = CertificateParams::new(vec!["localhost".to_string()]).unwrap();
    let cert = params.signed_by(&key, ca).unwrap();
    let key_der = PrivatePkcs8KeyDer::from(key.serialize_der()).into();
    let config = ServerConfig::builder()
        .with_no_client_auth()
        .with_single_cert(vec![cert.der().clone()], key_der)
        .unwrap();
    let acceptor = TlsAcceptor::from(Arc::new(config));

    let listener = TcpListener::bind("127.0.0.1:0").await.unwrap();
    let addr = listener.local_addr().unwrap();
    tokio::spawn(async move {
        loop {
            let (tcp, _) = listener.accept().await.unwrap();
            if let Ok(mut tls) = acceptor.accept(tcp).await {
                tls.write_all(b"hello").await.unwrap();
                tls.shutdown().await.unwrap();
            }
        }
    });
    addr
}

/// Connects to `addr` as "localhost", trusting only `ca`, and returns what the server sent.
async fn fetch(
    addr: SocketAddr,
    ca: &CertifiedIssuer<'static, KeyPair>,
) -> std::io::Result<String> {
    let mut roots = RootCertStore::empty();
    roots.add(ca.der().clone()).unwrap();
    let config = ClientConfig::builder()
        .with_root_certificates(roots)
        .with_no_client_auth();
    let connector = TlsConnector::from(Arc::new(config));
    let tcp = TcpStream::connect(addr).await?;
    let name = ServerName::try_from("localhost").unwrap();
    let mut tls = connector.connect(name, tcp).await?;
    let mut text = String::new();
    tls.read_to_string(&mut text).await?;
    Ok(text)
}

#[tokio::test]
async fn a_client_that_trusts_our_ca_gets_the_greeting() {
    let ca = new_ca("test CA");
    let addr = start_server(&ca).await;
    assert_eq!(fetch(addr, &ca).await.unwrap(), "hello");
}

#[tokio::test]
async fn a_client_that_trusts_another_ca_is_rejected() {
    let ca = new_ca("test CA");
    let addr = start_server(&ca).await;
    let err = fetch(addr, &new_ca("other CA")).await.unwrap_err();
    assert!(
        err.to_string().contains("UnknownIssuer"),
        "unexpected error: {err}"
    );
}
```

```bash
cargo test --test tls
```

```text
running 2 tests
test a_client_that_trusts_another_ca_is_rejected ... ok
test a_client_that_trusts_our_ca_gets_the_greeting ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: каждый тест строит в памяти свою маленькую PKI (CA, сертификаты, доверие), поэтому тесты не зависят от файлов, дат на диске и друг от друга. Отрицательный тест так же важен, как положительный: он доказывает, что клиент действительно проверяет, а не принимает всё подряд.

</details>

## Вопросы для самопроверки

1. Какие три вещи нужны TLS-серверу и какая из них должна оставаться секретной?

<details>
<summary>Ответ</summary>

Цепочка сертификатов, соответствующий закрытый ключ и конфигурация (версии, ALPN, аутентификация клиентов). Секретен только закрытый ключ; цепочка отправляется каждому клиенту.

</details>

2. Клиент подключается к `127.0.0.1`, и в логе сервера видно `SNI None`. Почему?

<details>
<summary>Ответ</summary>

SNI может нести только имя хоста, а не IP-адрес, поэтому клиент, подключающийся по IP, SNI не отправляет. Проверка сертификата всё равно работает, потому что в сертификате есть `127.0.0.1` как IP-SAN.

</details>

3. Что отправляет `tls.shutdown().await` и что идёт не так без этого?

<details>
<summary>Ответ</summary>

Он отправляет предупреждение TLS `close_notify` и затем закрывает соединение. Без этого внимательный клиент (как клиент `rustls`) сообщает `UnexpectedEof`, потому что не может отличить конец данных от обрыва соединения злоумышленником.

</details>

4. Что проверяет сервер во взаимном TLS и что он от этого получает?

<details>
<summary>Ответ</summary>

Он требует сертификат клиента, проверяет, что его подписал доверенный CA и что клиент владеет соответствующим ключом. Он получает надёжную личность каждого клиента без паролей, которую может использовать для авторизации.

</details>

5. Ваша программа паникует в `ServerConfig::builder()` с сообщением «Could not automatically determine the process-level CryptoProvider». Что случилось?

<details>
<summary>Ответ</summary>

В дереве зависимостей включены два криптопровайдера (`ring` и `aws-lc-rs`), и `rustls` не может выбрать один. Оставьте ровно одну фичу провайдера или установите провайдер через `CryptoProvider::install_default()` (например, `rustls::crypto::ring::default_provider().install_default()`) в начале `main`.

</details>

## Итоги

- TLS-серверу нужны цепочка сертификатов, закрытый ключ, который никогда его не покидает, и `ServerConfig`; клиенту — правильный доверенный корень и ожидаемое имя.
- Для разработки сделайте собственный CA через `rcgen`, подпишите короткоживущие серверные и клиентские сертификаты и отдавайте `ca.pem` только своим клиентам.
- `TlsAcceptor::accept(tcp)` превращает `TcpStream` в `TlsStream` после полного рукопожатия; SNI, ALPN и сертификаты клиента читаются из `get_ref()`.
- Клиент с `RootCertStore::empty()` плюс ваш CA доверяет ровно одному корню; ошибки имени перечисляют, что предлагает сертификат.
- Взаимный TLS (`WebPkiClientVerifier`) заставляет клиентов доказывать свою личность сертификатами.
- Используйте один криптопровайдер, всегда делайте `shutdown()` для TLS-потока и никогда не отключайте проверку.

## Источники и что почитать

- [Документация rustls](https://docs.rs/rustls/latest/rustls/) — `ServerConfig`, `ClientConfig`, криптопровайдеры и руководство с практическими рецептами.
- [Документация tokio-rustls](https://docs.rs/tokio-rustls/latest/tokio_rustls/) — `TlsAcceptor` и `TlsConnector`.
- [Документация rcgen](https://docs.rs/rcgen/latest/rcgen/) — `CertificateParams`, `CertifiedIssuer` и генерация ключей.
- [rustls-pki-types: PemObject](https://docs.rs/rustls-pki-types/latest/rustls_pki_types/pem/trait.PemObject.html) — чтение сертификатов и ключей из PEM-файлов.
- [RFC 8446 — The Transport Layer Security (TLS) Protocol Version 1.3](https://www.rfc-editor.org/rfc/rfc8446) — сам протокол; в разделе 4.2 перечислены расширения, такие как SNI и ALPN.
- [Let's Encrypt — How It Works](https://letsencrypt.org/how-it-works/) — как публичные серверы автоматически получают бесплатные сертификаты.

---

[🏠 Начало](../START-HERE.md) · [← 15.11 Проект: DNS-резолвер](11-project-dns-resolver.md) · [📋 Этап 15](README.md) · [15.13 QUIC и HTTP/3 →](13-quic-and-http3.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
