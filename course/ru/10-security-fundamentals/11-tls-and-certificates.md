<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 10.10 Криптография II: шифрование и подписи](10-cryptography-encryption-and-signatures.md) · [📋 Этап 10](README.md) · [10.12 Безопасность памяти и проверка небезопасного кода →](12-memory-safety-and-reviewing-unsafe.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/10-security-fundamentals/11-tls-and-certificates.md) <!-- nav -->

# Урок 10.11 — TLS и сертификаты

> **Этап 10 — Основы безопасности** · Урок 11 из 15 · ⏱ около 120 минут

## Что вы узнаете

- Что защищает TLS (приватность, целостность, личность) и как рукопожатие использует инструменты из Уроков 10.9 и 10.10.
- Что содержит сертификат и как цепочка доверия ведёт от сайта к корневому удостоверяющему центру.
- Как посмотреть настоящее TLS-соединение с помощью `curl -v` и `openssl s_client`.
- Как создать локальный самоподписанный сертификат, поднять с ним сервер и увидеть, почему клиенты его отвергают.
- Как клиент на Rust с `rustls` проверяет сервер и почему проверку нельзя отключать.

## Перед началом

- Вы прошли [Урок 10.10 — Криптография II: шифрование и подписи](10-cryptography-encryption-and-signatures.md).
- Вы знаете, что такое TCP-соединение (Transmission Control Protocol, протокол управления передачей): надёжный поток байтов между двумя программами. TLS работает поверх него.
- Инструменты: `curl` (уже есть в macOS) и OpenSSL 3. `openssl`, который идёт с macOS, — это более старый форк (LibreSSL), поэтому урок использует OpenSSL из Homebrew; если его нет, установите командой `brew install openssl`. Ниже команды пишутся как `openssl` после того, как вы поставили его первым в `PATH`.

```bash
export PATH="/opt/homebrew/opt/openssl/bin:$PATH"
openssl version
```

```text
OpenSSL 3.6.2 7 Apr 2026 (Library: OpenSSL 3.6.2 7 Apr 2026)
```

- Создайте учебный пакет и папку для локальных сертификатов, которую Git должен игнорировать:

```bash
cd ~/projects/rust
cargo new code/p10-tls
cd code/p10-tls
mkdir certs
echo "certs/" >> .gitignore
cargo add rustls@0.23 --no-default-features --features ring,std,tls12
cargo add webpki-roots@1
```

> 🔒 **Безопасность и этика:** вы подключаетесь только к публичному сайту (читать публичную страницу можно) и к своему собственному серверу на `localhost`. Вы **не** будете добавлять тестовый сертификат в связку ключей macOS или в какое-либо хранилище доверия: он используется только через параметры командной строки.

## Новые слова

| Слово | Значение |
|---|---|
| **TLS** (Transport Layer Security, защита транспортного уровня) | Протокол, который делает соединение приватным, защищённым от подмены и аутентифицированным. Текущая версия — 1.3; версия 1.2 всё ещё широко используется. Он заменил более старый SSL (Secure Sockets Layer, уровень защищённых сокетов). |
| **HTTPS** (HTTP Secure, защищённый HTTP) | HTTP (HyperText Transfer Protocol, протокол передачи гипертекста — веб-протокол), переданный внутри TLS. |
| **рукопожатие** (handshake) | Первые сообщения TLS-соединения, в которых обе стороны договариваются об алгоритмах и ключах. |
| **сертификат** (certificate) | Подписанный документ, который утверждает: «этот открытый ключ принадлежит этому имени». Его формат называется X.509. |
| **CA** (Certificate Authority, удостоверяющий центр) | Организация, которая проверяет, кто вы, и подписывает сертификаты. |
| **корневой сертификат** (root certificate) | Сертификат CA, которому ваша система уже доверяет. Операционные системы и браузеры поставляются со списком таких сертификатов — *хранилищем доверия* (trust store). |
| **промежуточный сертификат** (intermediate certificate) | Сертификат, подписанный корневым, который, в свою очередь, подписывает обычные сертификаты сайтов. |
| **цепочка доверия** (chain of trust) | Последовательность лист → промежуточный → корневой. Каждый сертификат подписан следующим. |
| **SAN** (Subject Alternative Name, альтернативное имя субъекта) | Поле сертификата со списком имён хостов (имён DNS, Domain Name System, система доменных имён), для которых он действителен. |
| **самоподписанный сертификат** (self-signed certificate) | Сертификат, подписанный собственным ключом. Никто за него не ручается, поэтому клиенты его отвергают, если вы явно ему не доверились. |
| **прямая секретность** (forward secrecy) | Свойство обмена ключами: если долговременный ключ сервера позже утечёт, старый записанный трафик всё равно нельзя расшифровать. |
| **человек посередине** (man-in-the-middle, MITM) | Злоумышленник, который находится между клиентом и сервером, читает или меняет трафик и притворяется каждой из сторон. |
| **PEM** (Privacy-Enhanced Mail) | Текстовый формат ключей и сертификатов между строками вроде `-----BEGIN CERTIFICATE-----`. |

## 1. Что даёт TLS

Обычный HTTP поверх TCP — это открытка: любой на пути (Wi-Fi-роутер, провайдер, взломанная машина) может её прочитать и изменить,
а вы не можете знать, кто на самом деле на другом конце. TLS решает сразу все три проблемы. Две из них — цели триады CIA из Урока 10.1 (конфиденциальность и целостность; для доступности TLS ничего не делает), а третья — аутентификация:

| Обещание | Как |
|---|---|
| **Конфиденциальность** (никто не прочитает) | Данные шифруются AEAD-шифром (AEAD, Authenticated Encryption with Associated Data, аутентифицированное шифрование с присоединёнными данными; Урок 10.10) ключом, который знают только две стороны. |
| **Целостность** (никто не изменит) | Метка AEAD обнаруживает любое изменение. |
| **Аутентификация** (вы говорите с нужным сервером) | Сервер доказывает свою личность сертификатом и подписью. |

TLS **не** скрывает, что вы обращаетесь к серверу (IP, Internet Protocol, адрес виден, а обычно и имя хоста, потому что клиент отправляет его в рукопожатии; см. SNI в разделе 4), и не делает сам сервер заслуживающим доверия, а его код — свободным от ошибок.

## 2. Рукопожатие простыми словами

```text
 Client                                                   Server
   │ ── ClientHello: my TLS versions, my ciphers,           │
   │    a fresh public key share (key exchange) ──────────► │
   │                                                        │  picks a version and cipher,
   │ ◄── ServerHello: my key share ─────────────────────────│  computes the shared secret
   │                                                        │
   │   ═══ from here on everything is encrypted ═══         │
   │ ◄── Certificate: my certificate chain ─────────────────│
   │ ◄── CertificateVerify: a signature made with my        │
   │     private key over the whole handshake so far ───────│
   │ ◄── Finished (a MAC over the handshake) ───────────────│
   │ ── Finished ─────────────────────────────────────────► │
   │ ◄═══════════ application data (HTTP), AEAD ═══════════►│
```

Каждый шаг использует знакомый вам инструмент:

1. **Обмен ключами.** Каждая сторона отправляет новый временный открытый ключ (например, X25519 — обмен ключами на эллиптической кривой Curve25519 с 32-байтными ключами). Из своего закрытого ключа и открытого ключа другой стороны обе вычисляют **один и тот же общий секрет**, который никогда не передаётся по сети. Ключи временные, что даёт **прямую секретность**. Современные серверы подмешивают и постквантовый алгоритм (вы увидите `X25519MLKEM768` в выводе ниже), поэтому записанный трафик остаётся в безопасности даже перед будущими квантовыми компьютерами.
2. **Формирование ключей.** Общий секрет пропускают через функции на основе хешей (HKDF, HMAC-based Key Derivation Function, функция формирования ключей на основе HMAC — кода аутентификации сообщения на основе хеша из Урока 10.9), чтобы получить ключи шифрования.
3. **Аутентификация.** Сервер отправляет свой сертификат и подписывает рукопожатие парным закрытым ключом (Урок 10.10). Клиент проверяет сертификат (раздел 3) и подпись. Такую подпись может создать только настоящий владелец закрытого ключа.
4. **Данные.** Каждая запись шифруется и аутентифицируется AEAD (`AES-GCM` или `ChaCha20-Poly1305`).

## 3. Сертификаты и цепочка доверия

Сертификат — небольшой документ, подписанный CA, который содержит:

| Поле | Значение |
|---|---|
| Subject | Кто это (например, `CN=rust-lang.org`; CN — Common Name, общее имя). |
| SAN | Имена хостов, для которых сертификат действителен. **Клиенты проверяют именно его**, а не subject. |
| Открытый ключ | Ключ, парный закрытому ключу на сервере. |
| Срок действия | Даты `notBefore` и `notAfter`. Вне них сертификат недействителен. |
| Issuer | Кто подписал. |
| Basic constraints | `CA:TRUE` для CA, `CA:FALSE` для обычных сертификатов. |
| Подпись | Подпись издателя над всем перечисленным. |

Заранее знать ключ каждого сайта невозможно, поэтому доверие передаётся по цепочке:

```text
   Root CA certificate      ← already in your system's trust store (you trust it)
        │ signs
        ▼
   Intermediate certificate ← sent by the server
        │ signs
        ▼
   Leaf certificate         ← the website's own certificate (CA:FALSE)
```

Клиент принимает сервер, только если **все** условия выполнены:

1. У каждого сертификата в цепочке верная подпись следующего выше, а цепочка заканчивается корневым сертификатом из хранилища доверия.
2. Каждый сертификат находится в пределах срока действия.
3. SAN листового сертификата содержит имя хоста, которое запросил клиент.
4. Листовой сертификат может использоваться как серверный (он не CA).

Если любая проверка не пройдена, клиент обязан отказаться. Человек посередине не может заставить CA подписать сертификат для имени хоста, которым он не владеет, и не может подписать его сам, потому что клиент ему не доверяет.

> 🔒 **Безопасность:** сертификаты намеренно недолговечны. Тот, что вы увидите ниже, действует около 90 дней. Короткий срок ограничивает ущерб от украденного ключа, а такие инструменты, как ACME (Automatic Certificate Management Environment, автоматическое управление сертификатами; используется в Let's Encrypt), продлевают их автоматически. Просроченный сертификат — очень частая причина сбоев.

## 4. Смотрим на настоящее соединение

Попросите `curl` показать TLS-часть запроса к публичному сайту. Оставим только интересные строки:

```bash
curl -sv -o /dev/null https://www.rust-lang.org 2>&1 | grep -E "TLS handshake|SSL connection|subject|issuer|expire date|start date|verify ok"
```

```text
* (304) (OUT), TLS handshake, Client hello (1):
* (304) (IN), TLS handshake, Server hello (2):
* (304) (IN), TLS handshake, Unknown (8):
* (304) (IN), TLS handshake, Certificate (11):
* (304) (IN), TLS handshake, CERT verify (15):
* (304) (IN), TLS handshake, Finished (20):
* (304) (OUT), TLS handshake, Finished (20):
* SSL connection using TLSv1.3 / AEAD-CHACHA20-POLY1305-SHA256 / [blank] / UNDEF
*  subject: CN=rust-lang.org
*  start date: Sep 21 20:02:52 2026 GMT
*  expire date: Dec 20 20:02:51 2026 GMT
*  subjectAltName: host "www.rust-lang.org" matched cert's "www.rust-lang.org"
*  issuer: C=US; O=Let's Encrypt; CN=YR1
*  SSL certificate verify ok.
```

**Что происходит:**

1. `-s` прячет индикатор прогресса, `-v` показывает подробности, `-o /dev/null` выбрасывает страницу, а `2>&1` направляет подробности (которые `curl` печатает в поток ошибок) в конвейер для `grep`.
2. Строки «Client hello», «Server hello», «Certificate», «CERT verify» и «Finished» — это рукопожатие из раздела 2. Шифр — AEAD (`CHACHA20-POLY1305`).
3. `subjectAltName … matched` — проверка 3 из раздела 3, а `SSL certificate verify ok` означает, что все проверки пройдены.

У вас будут другие даты (а возможно, и другой шифр): сертификат регулярно продлевают.

`openssl s_client` — более подробный инструмент. Запустите его, не подавая ввода (`</dev/null`), и отфильтруйте вывод:

```bash
openssl s_client -connect www.rust-lang.org:443 -servername www.rust-lang.org </dev/null 2>/dev/null | grep -E "^ *[0-9] s:|^ +i:|Verification|Protocol|^New|group"
```

```text
 0 s:CN=rust-lang.org
   i:C=US, O=Let's Encrypt, CN=YR1
 1 s:C=US, O=Let's Encrypt, CN=YR1
   i:C=US, O=ISRG, CN=Root YR
 2 s:C=US, O=ISRG, CN=Root YR
   i:C=US, O=Internet Security Research Group, CN=ISRG Root X1
Negotiated TLS1.3 group: X25519MLKEM768
Verification: OK
New, TLSv1.3, Cipher is TLS_AES_128_GCM_SHA256
Protocol: TLSv1.3
    Protocol  : TLSv1.3
```

**Что происходит:**

1. `-connect host:port` открывает соединение, а `-servername` отправляет имя хоста в рукопожатии (SNI, Server Name Indication, указание имени сервера), чтобы сервер, на котором много сайтов, знал, какой сертификат показать.
2. `s:` — это *subject* (субъект), а `i:` — *issuer* (издатель) каждого сертификата. Сертификат 0 (листовой, `rust-lang.org`) выдан `YR1`; `YR1` выдан `Root YR`; `Root YR` выдан `ISRG Root X1` — корневым сертификатом, который есть в хранилище доверия. Это цепочка из раздела 3.
3. `Negotiated TLS1.3 group` — обмен ключами из раздела 2: X25519 в сочетании с постквантовым алгоритмом ML-KEM (Module-Lattice Key Encapsulation Mechanism, механизм инкапсуляции ключа на модульных решётках).
4. `Verification: OK` — результат проверок по хранилищу доверия OpenSSL. (Последняя строка `Protocol` относится к билету сессии, который сервер отправляет после рукопожатия.)

Чтобы подробно прочитать один сертификат, передайте его в `openssl x509`:

```bash
openssl s_client -connect www.rust-lang.org:443 -servername www.rust-lang.org </dev/null 2>/dev/null | openssl x509 -noout -subject -issuer -dates -ext subjectAltName,basicConstraints
```

```text
subject=CN=rust-lang.org
issuer=C=US, O=Let's Encrypt, CN=YR1
notBefore=Sep 21 20:02:52 2026 GMT
notAfter=Dec 20 20:02:51 2026 GMT
X509v3 Basic Constraints: critical
    CA:FALSE
X509v3 Subject Alternative Name: 
    DNS:rust-lang.org, DNS:www.rust-lang.org
```

Срок действия — около 90 дней, сертификат не является CA, а два имени, которые он покрывает, перечислены в SAN.

## 5. Локальный самоподписанный сертификат

Для собственных экспериментов сертификат можно сделать самому. Он **самоподписанный**: его издатель — он сам, поэтому ни один CA за него не ручается. Создайте его в папке `certs`:

```bash
cd ~/projects/rust/code/p10-tls/certs
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
  -keyout key.pem -out cert.pem -days 7 -subj "/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1" \
  -addext "basicConstraints=critical,CA:FALSE"
openssl x509 -in cert.pem -noout -subject -issuer -dates -ext subjectAltName,basicConstraints
```

```text
subject=CN=localhost
issuer=CN=localhost
notBefore=Nov 18 06:39:48 2026 GMT
notAfter=Nov 25 06:39:48 2026 GMT
X509v3 Subject Alternative Name: 
    DNS:localhost, IP Address:127.0.0.1
X509v3 Basic Constraints: critical
    CA:FALSE
```

**Что происходит:**

1. `req -x509` сразу создаёт самоподписанный сертификат. `-newkey ec -pkeyopt ec_paramgen_curve:prime256v1` создаёт новую пару ключей на эллиптической кривой. `-nodes` значит «no DES» (DES, Data Encryption Standard, стандарт шифрования данных, — старый шифр; опция значит «не шифровать файл закрытого ключа». Это допустимо для одноразового тестового ключа, но никогда для настоящего. OpenSSL 3 понимает и новое имя `-noenc`).
2. `-days 7` ограничивает срок действия неделей. `-subj "/CN=localhost"` задаёт subject.
3. Два параметра `-addext` добавляют SAN (его проверяют клиенты) и сообщают, что это не CA. `key.pem` — это **закрытый** ключ; `cert.pem` — сертификат. Subject и issuer совпадают: это и значит «самоподписанный».

Откройте **второй терминал** и запустите небольшой тестовый TLS-сервер, который отдаёт собственную страницу состояния. Он работает, пока вы не нажмёте `Ctrl+C`:

```bash
cd ~/projects/rust/code/p10-tls/certs
openssl s_server -accept 8443 -cert cert.pem -key key.pem -www -quiet
```

В первом терминале подключитесь с хранилищем доверия по умолчанию:

```bash
curl https://localhost:8443/
```

```text
curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.se/docs/sslcerts.html
```

(`curl` печатает ещё и более длинное объяснение. Важна первая строка.) Соединение отвергнуто: сертификат корректен, но его не подписал никто, кому доверяет система. Теперь скажите `curl`, что именно этот сертификат — доверенный корневой **для этой одной команды**:

```bash
curl -sS --cacert cert.pem -o /dev/null -w "HTTP %{http_code}\n" https://localhost:8443/
```

```text
HTTP 200
```

Проверка имени хоста по-прежнему действует. Запросите имя, которого нет в SAN, направив его на тот же сервер (`--resolve` сопоставляет имя с адресом только для этой команды):

```bash
curl -sS --cacert cert.pem --resolve other.test:8443:127.0.0.1 -o /dev/null https://other.test:8443/
```

```text
curl: (60) SSL: no alternative certificate subject name matches target host name 'other.test'
More details here: https://curl.se/docs/sslcerts.html
```

То же самое с `openssl s_client`. Первая команда (без `-CAfile`) сообщает об ошибке проверки; вторая (с `-CAfile`) выводит `Verification: OK`:

```bash
openssl s_client -connect localhost:8443 -servername localhost -brief </dev/null 2>&1 | sed -n 1,4p
openssl s_client -connect localhost:8443 -servername localhost -CAfile cert.pem -verify_hostname localhost -brief </dev/null 2>&1 | grep -E "Verif"
```

```text
Connecting to ::1
depth=0 CN=localhost
verify error:num=18:self-signed certificate
CONNECTION ESTABLISHED
Verification: OK
Verified peername: localhost
```

**Что происходит:** ошибка `num=18` означает «self-signed certificate» (самоподписанный сертификат). С `-CAfile cert.pem` мы велели OpenSSL доверять именно этому сертификату, а `-verify_hostname localhost` просит проверить ещё и имя.

> 🔒 **Безопасность:** у каждого инструмента есть переключатель, отключающий проверки (`curl -k`, `danger_accept_invalid_certs` в HTTP-библиотеках; а `openssl s_client` по умолчанию только *сообщает* об ошибке проверки и оставляет соединение открытым, как вы видели выше, если не добавить `-verify_return_error`). Без проверки соединение зашифровано, но вы не знаете, с кем
> говорите, поэтому человек посередине прочитает всё. «Работает с `-k`» — то же самое, что «сломано». Вместо этого доверяйте конкретному сертификату (`--cacert`), как выше.
> И никогда не импортируйте тестовый сертификат в связку ключей macOS: тогда ему начнёт доверять весь ваш компьютер.

Остановите сервер через `Ctrl+C` во втором терминале.

## 6. TLS-клиент на Rust с rustls

`rustls` — библиотека TLS, написанная на Rust. Она выполняет всё рукопожатие и все проверки из раздела 3. Ваша программа передаёт **список доверенных корней** (`webpki-roots` — копия списка Mozilla) и **имя хоста**. Замените `src/main.rs` в `code/p10-tls`:

```rust
use std::io::{Read, Write};
use std::net::TcpStream;
use std::sync::Arc;

use rustls::pki_types::ServerName;
use rustls::{ClientConfig, ClientConnection, RootCertStore, Stream};

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let host = std::env::args().nth(1).unwrap_or_else(|| "www.rust-lang.org".to_string());
    let port = std::env::args().nth(2).unwrap_or_else(|| "443".to_string());

    // 1. Список удостоверяющих центров, которым мы доверяем (копия списка Mozilla).
    let roots = RootCertStore::from_iter(webpki_roots::TLS_SERVER_ROOTS.iter().cloned());
    let config = ClientConfig::builder().with_root_certificates(roots).with_no_client_auth();

    // 2. Объект TLS-соединения, который ожидает сертификат для `host`.
    let server_name = ServerName::try_from(host.clone())?;
    let mut tls = ClientConnection::new(Arc::new(config), server_name)?;
    let mut tcp = TcpStream::connect(format!("{host}:{port}"))?;
    let mut stream = Stream::new(&mut tls, &mut tcp);

    // 3. Отправляем обычный HTTP-запрос. Рукопожатие происходит автоматически при первой записи.
    write!(stream, "GET / HTTP/1.1\r\nHost: {host}\r\nConnection: close\r\n\r\n")?;
    let mut first = [0u8; 40];
    stream.read_exact(&mut first)?;
    println!("version : {:?}", stream.conn.protocol_version().unwrap());
    println!("cipher  : {:?}", stream.conn.negotiated_cipher_suite().unwrap().suite());
    println!("reply   : {}", String::from_utf8_lossy(&first).lines().next().unwrap_or(""));
    Ok(())
}
```

**Что происходит:**

1. Хост и порт берутся из командной строки, со значениями по умолчанию. `RootCertStore` хранит доверенные корни. `ClientConfig::builder()` начинает настройку: ей нужны корни, и мы сообщаем, что клиентский сертификат не предъявляем.
2. `ServerName` — имя, которое должно быть в SAN сертификата. `ClientConnection` — конечный автомат TLS; `TcpStream` — нижележащее соединение. `Stream` связывает их, чтобы обычные вызовы `Read` и `Write` шифровали и расшифровывали за вас.
3. Первый `write!` запускает рукопожатие. Если сертификат неверный, `write!` или `read_exact` вернёт ошибку, а `?` завершит программу до того, как какие-либо данные будут отправлены открытым текстом.
4. Мы печатаем согласованную версию TLS и набор шифров, а также первую строку HTTP-ответа (для неё хватает 40 байт).

Запустите `cargo run` (при первом запуске компилируется `rustls`, это занимает минуту):

```text
version : TLSv1_3
cipher  : TLS13_AES_128_GCM_SHA256
reply   : HTTP/1.1 301 Moved Permanently
```

(`301` — это перенаправление с `www.rust-lang.org` на `rust-lang.org`: запрос прошёл, а это всё, что мы хотели увидеть.) Теперь снова запустите локальный сервер (второй терминал, та же команда, что в разделе 5) и направьте клиента на него:

```bash
cargo run --quiet -- localhost 8443
```

```text
Error: Custom { kind: InvalidData, error: InvalidCertificate(UnknownIssuer) }
```

**Что происходит:** локальный сертификат не подписан ни одним корнем из списка, поэтому `rustls` отказывает (`UnknownIssuer`), точно как и `curl`. Ошибка возникает во время рукопожатия, до того как программа отправит запрос. Остановите сервер через `Ctrl+C`.

> 📏 **Соглашение:** используйте библиотеку TLS, но никогда не пишите свой TLS. В настоящих сервисах часть TLS обычно выполняет обратный прокси или фреймворк, использующий `rustls`. Вы настраиваете *файлы сертификата и ключа* и *каким корням доверять*. Протокол вы не пишете.

## Частые ошибки

**1. Отключение проверки, «чтобы заработало».** `curl -k`, `danger_accept_invalid_certs(true)` или собственный верификатор, принимающий всё. Исправление — довериться нужному сертификату (`--cacert` или добавить его в `RootCertStore`) либо устранить настоящую проблему (неверное имя, истёкший срок, отсутствующий промежуточный сертификат).

**2. Сертификат без SAN.** Современные клиенты, такие как браузеры и `rustls`, игнорируют `CN` и смотрят только на SAN, поэтому сертификат с `-subj "/CN=localhost"` и без `subjectAltName` они отвергнут (некоторые старые инструменты всё ещё откатываются к `CN`, что скрывает ошибку). Всегда добавляйте SAN.

**3. Подключение по другому имени, чем в сертификате** (IP-адрес, когда в SAN только имена `DNS:`, или `127.0.0.1`, когда указан только `localhost`). Добавьте имя или запись `IP:` либо подключайтесь по правильному имени.

**4. Забыть, что сертификаты истекают.** Автоматизируйте продление и следите за датой окончания. `openssl x509 -noout -enddate` показывает её.

**5. Коммит `key.pem` в Git.** Закрытый ключ в репозитории становится публичным навсегда (Этап 01, Урок 10.8). Поэтому `certs/` занесена в `.gitignore`.

## Попробуйте сами

1. Выполните `openssl s_client -connect www.rust-lang.org:443 -servername www.rust-lang.org -tls1_2 -brief </dev/null`. Какие протокол и шифр используются теперь?
2. Выведите весь сертификат командой `openssl x509 -in cert.pem -noout -text` и найдите издателя, даты срока действия, открытый ключ и алгоритм подписи.
3. Запустите клиент на Rust для `www.rust-lang.org` и для `localhost` (сервер включён) и сравните два результата.

## Упражнения

### Упражнение 1 — Когда истекает? (легко)

Напишите одну команду, которая печатает только дату окончания сертификата `www.rust-lang.org`.

<details>
<summary>Подсказка</summary>

Передайте вывод `openssl s_client` (с `</dev/null 2>/dev/null`) в `openssl x509 -noout -enddate`.

</details>

<details>
<summary>Ответ</summary>

```bash
openssl s_client -connect www.rust-lang.org:443 -servername www.rust-lang.org </dev/null 2>/dev/null | openssl x509 -noout -enddate
```

```text
notAfter=Dec 20 20:02:51 2026 GMT
```

Почему это работает: `s_client` печатает сертификат в формате PEM; `x509` читает его из конвейера, а `-enddate` печатает только поле `notAfter`. Такую команду можно использовать в скрипте мониторинга.

</details>

### Упражнение 2 — Сертификат для собственного имени (средне)

Создайте самоподписанный сертификат для имени `myapp.test` (домен `.test` зарезервирован для тестов), поднимите сервер с ним на порту 8444 и сделайте три вызова `curl`:
успешный, один без `--cacert` и один с именем `other.test`. Используйте `--resolve`, чтобы сопоставить имена с `127.0.0.1`.

<details>
<summary>Подсказка</summary>

Та же команда `openssl req`, что в разделе 5, с `/CN=myapp.test` и `subjectAltName=DNS:myapp.test`. Параметр `curl` — `--resolve myapp.test:8444:127.0.0.1`.

</details>

<details>
<summary>Ответ</summary>

```bash
cd ~/projects/rust/code/p10-tls/certs
openssl req -x509 -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes \
  -keyout key2.pem -out cert2.pem -days 7 -subj "/CN=myapp.test" \
  -addext "subjectAltName=DNS:myapp.test" -addext "basicConstraints=critical,CA:FALSE"
# second terminal: openssl s_server -accept 8444 -cert cert2.pem -key key2.pem -www -quiet
curl -sS --resolve myapp.test:8444:127.0.0.1 --cacert cert2.pem -o /dev/null -w "HTTP %{http_code}\n" https://myapp.test:8444/
curl -sS --resolve myapp.test:8444:127.0.0.1 -o /dev/null https://myapp.test:8444/
curl -sS --resolve other.test:8444:127.0.0.1 --cacert cert2.pem -o /dev/null https://other.test:8444/
```

```text
HTTP 200
curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.se/docs/sslcerts.html
curl: (60) SSL: no alternative certificate subject name matches target host name 'other.test'
More details here: https://curl.se/docs/sslcerts.html
```

Почему это работает: вызов 1 доверяет именно этому сертификату, и имя совпадает с SAN. В вызове 2 нет доверия к самоподписанному сертификату. В вызове 3 сертификату доверяют, но имени нет в SAN. Остановите сервер через `Ctrl+C`.

</details>

### Упражнение 3 — Сопоставление имён хостов с тестами (сложно)

Сертификаты могут содержать подстановочный знак, например `*.example.com`. Напишите `pub fn host_matches(pattern: &str, host: &str) -> bool` для нового библиотечного пакета по таким правилам:
имена сравниваются без учёта регистра; подстановочный знак допустим только как вся первая метка (`*.example.com`) и соответствует **ровно одной** метке.
Значит, `www.example.com` подходит, а `example.com`, `a.b.example.com`, `.example.com` и `www.example.com.evil.test` — нет. Напишите тесты на все эти случаи.
(В настоящем коде вы никогда не пишете это сами: это делает библиотека TLS. Упражнение показывает, как мелкие ошибки в таком правиле превращаются в дыры безопасности.)

<details>
<summary>Подсказка</summary>

`pattern.strip_prefix("*.")` возвращает `Some(suffix)` для подстановочных имён. Для хоста `split_once('.')` даёт первую метку и остаток; остаток должен **точно** равняться суффиксу, а первая метка не должна быть пустой.

</details>

<details>
<summary>Ответ</summary>

```rust
/// Does a certificate name (`pattern`) match the host the client wants to reach?
/// Rules: names are compared case-insensitively; a wildcard is allowed only as the whole
/// left-most label (`*.example.com`) and matches exactly one label.
pub fn host_matches(pattern: &str, host: &str) -> bool {
    let pattern = pattern.to_ascii_lowercase();
    let host = host.to_ascii_lowercase();
    match pattern.strip_prefix("*.") {
        None => pattern == host,
        Some(suffix) => match host.split_once('.') {
            Some((label, rest)) => !label.is_empty() && rest == suffix,
            None => false,
        },
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn exact_names_match_ignoring_case() {
        assert!(host_matches("www.example.com", "WWW.Example.com"));
        assert!(!host_matches("www.example.com", "example.com"));
    }

    #[test]
    fn wildcard_matches_exactly_one_label() {
        assert!(host_matches("*.example.com", "www.example.com"));
        assert!(!host_matches("*.example.com", "example.com"));
        assert!(!host_matches("*.example.com", "a.b.example.com"));
        assert!(!host_matches("*.example.com", ".example.com"));
    }

    #[test]
    fn lookalike_suffixes_do_not_match() {
        assert!(!host_matches("*.example.com", "www.example.com.evil.test"));
        assert!(!host_matches("*.example.com", "www.badexample.com"));
    }
}
```

`cargo test` показывает 3 успешных теста.

Почему это работает: остаток хоста после первой точки должен в точности равняться суффиксу, поэтому `a.b.example.com` (остаток `b.example.com`) и `www.example.com.evil.test` не проходят. Небрежный `host.ends_with("example.com")` принял бы `www.badexample.com`.

</details>

## Вопросы для самопроверки

1. Назовите три вещи, которые даёт TLS.

<details>
<summary>Ответ</summary>

Конфиденциальность (шифрование), целостность (обнаружение подмены) и аутентификацию сервера (а по желанию и клиента) с помощью сертификатов.

</details>

2. Почему клиент доверяет сертификату `www.rust-lang.org`?

<details>
<summary>Ответ</summary>

Потому что он подписан промежуточным CA, тот подписан корневым CA, который есть в хранилище доверия клиента, потому что даты действительны и потому что имя хоста есть в SAN. Цепочка подписей ведёт к корню, которому клиент уже доверяет.

</details>

3. Что такое прямая секретность?

<details>
<summary>Ответ</summary>

Ключи сессии получаются из временных ключей обмена, поэтому даже если долговременный закрытый ключ сервера украдут позже, старый записанный трафик расшифровать нельзя.

</details>

4. Коллега говорит: «Работает только с `curl -k`». Что это означает?

<details>
<summary>Ответ</summary>

Проверка сертификата не проходит (неизвестный издатель, неверное имя, истёк срок или нет промежуточного сертификата). `-k` скрывает проблему и убирает защиту от атак «человек посередине». Найдите и устраните причину либо доверьтесь конкретному сертификату через `--cacert`.

</details>

5. Чего не хватает самоподписанному сертификату?

<details>
<summary>Ответ</summary>

Третьей стороны, которая за него ручается. Его издатель — он сам, поэтому никакая цепочка не ведёт к доверенному корню, и клиенты его отвергают, если им не сказано доверять именно этому сертификату.

</details>

## Итоги

- TLS даёт конфиденциальность, целостность и аутентификацию сервера поверх TCP; HTTPS — это HTTP поверх TLS.
- Рукопожатие использует временный обмен ключами (прямая секретность), формирование ключей, сертификат с подписью для подтверждения личности и AEAD-шифр для данных.
- Сертификат связывает открытый ключ с именами хостов (SAN) на определённый срок. Доверие идёт от корня в хранилище доверия через промежуточные сертификаты к листовому.
- Клиент обязан проверить цепочку, даты и имя хоста. Если любая проверка не пройдена, он должен остановиться.
- `curl -v` и `openssl s_client` показывают, что происходит на самом деле; `openssl req -x509` создаёт локальный тестовый сертификат, которому вы доверяете только через явные параметры.
- Никогда не отключайте проверку и никогда не кладите тестовый сертификат в системное хранилище доверия. Никогда не коммитьте закрытые ключи.
- В Rust протокол и проверки выполняет `rustls`; вы передаёте доверенные корни и имя.

## Источники и что почитать

- [RFC 8446 — TLS 1.3](https://datatracker.ietf.org/doc/html/rfc8446) — спецификация TLS 1.3 со схемой рукопожатия (RFC, Request for Comments, «запрос комментариев», — документ официальной серии интернет-стандартов).
- [Документация rustls](https://docs.rs/rustls/0.23.46/rustls/) — API, использованный в этом уроке.
- [Let's Encrypt — How It Works](https://letsencrypt.org/how-it-works/) — как сертификаты выдаются и продлеваются автоматически.
- [OpenSSL manual: s_client](https://docs.openssl.org/3.6/man1/openssl-s_client/) — все параметры использованной выше команды.
- [curl: SSL certificate verification](https://curl.se/docs/sslcerts.html) — как `curl` проверяет сертификаты и что делают его параметры.
- [Mozilla — Web Security Guidelines](https://infosec.mozilla.org/guidelines/web_security) — рекомендуемые настройки TLS для сервисов.

---

[🏠 Начало](../START-HERE.md) · [← 10.10 Криптография II: шифрование и подписи](10-cryptography-encryption-and-signatures.md) · [📋 Этап 10](README.md) · [10.12 Безопасность памяти и проверка небезопасного кода →](12-memory-safety-and-reviewing-unsafe.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
