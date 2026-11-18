<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 06.10 FFI: вызов кода на C](10-ffi-calling-c.md) · [📋 Этап 06](README.md) · [06.12 Контрольная точка: стадия 1 пройдена →](12-checkpoint-stage-1-complete.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/06-rust-advanced/11-project-multithreaded-web-server.md) <!-- nav -->

# Урок 06.11 — Проект: многопоточный веб-сервер

> **Этап 06 — Rust: продвинутый уровень** · Урок 11 из 12 · ⏱ около 4 часов (2 занятия)

## Что вы узнаете

- Как устроен веб-сервер на самом нижнем уровне: TCP-соединение, HTTP-запрос в виде обычного текста и ответ.
- Как построить пул потоков с ограниченной очередью и как корректно его остановить.
- Как защитить сервер: ограничения размера, тайм-ауты, ответы «очередь полна» и безопасные пути к файлам.
- Как организовать проект побольше: библиотека плюс тонкий бинарный файл, модульные тесты, интеграционный тест, приёмочный скрипт.
- Как использовать Git в проекте: ветка, один коммит на веху, слияние и тег.

## Перед началом

- Вы прошли [Урок 06.10 — FFI: вызов кода на C](10-ffi-calling-c.md).
- Вы знаете потоки, каналы, `Arc` и `Mutex`: [Урок 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md). Пул потоков строится именно из этих инструментов.
- Вы знаете модули и обработку ошибок: [Урок 04.6](../04-structs-enums-collections-errors/06-modules-crates-and-packages.md) и [Урок 04.10](../04-structs-enums-collections-errors/10-error-handling.md).
- Вы знаете `clap` (аргументы командной строки): [Урок 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md). Аргументы можно разбирать и вручную.
- Вы знаете ветки, слияния и теги Git: [Урок 01.6](../01-git-and-version-control/06-branches-and-merging.md) и [Урок 01.10](../01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md). Ваш учебный репозиторий находится в `~/projects/rust`.
- Установлен `curl` (он есть на каждом Mac). `curl --version` должен напечатать версию.

## Новые слова

| Слово | Значение |
|---|---|
| **сервер** (server) | Программа, которая ждёт подключения других программ (клиентов) и отвечает на их запросы. |
| **клиент** (client) | Программа, которая подключается к серверу и отправляет запросы. `curl` и ваш браузер — клиенты. |
| **IP-адрес** (Internet Protocol address) | Число, которое называет компьютер в сети, например `127.0.0.1`. |
| **localhost** / петля (loopback) | Адрес `127.0.0.1`: «этот же компьютер». К нему могут подключиться только программы на вашем компьютере. |
| **порт** (port) | Число (0–65535), которое выбирает одну программу на компьютере. Наш сервер использует порт `7878`. |
| **TCP** (Transmission Control Protocol, протокол управления передачей) | Правила надёжного упорядоченного потока байтов между двумя программами. Реализует его операционная система. |
| **сокет** (socket) | Дескриптор, через который программа отправляет и получает данные по сети. |
| **HTTP** (HyperText Transfer Protocol, протокол передачи гипертекста) | Текстовый язык, на котором общаются браузеры и веб-серверы поверх TCP. |
| **строка запроса** (request line) | Первая строка HTTP-запроса: метод, путь и версия, например `GET /health HTTP/1.1`. |
| **заголовок** (header) | Строка `Name: value` после строки запроса (или после строки статуса в ответе). |
| **код статуса** (status code) | Число в ответе, которое говорит, что произошло: `200` OK, `404` Not Found и так далее. |
| **путь** (path) | Часть URL после хоста, например `/static/note.txt`. |
| **URL** (Uniform Resource Locator, унифицированный указатель ресурса) | Веб-адрес, например `http://127.0.0.1:7878/health`. |
| **пул потоков** (thread pool) | Фиксированная группа рабочих потоков, которые берут задания из очереди, чтобы не запускать новый поток на каждое задание. |
| **рабочий поток** (worker) | Один поток пула. |
| **задание** (job) | Один кусок работы для пула: здесь «обработать это соединение». |
| **обратное давление** (backpressure) | Способ сказать «помедленнее», когда система заполнена, вместо принятия неограниченной работы. |
| **корректное завершение** (graceful shutdown) | Остановка программы с предварительным завершением текущей работы, а не её обрыв посередине. |
| **тайм-аут** (timeout) | Предел того, сколько вы ждёте чего-либо. |
| **обход пути** (path traversal) | Атака, при которой путь вроде `/static/../secret` пытается выйти за пределы разрешённой папки. |
| **DoS** (Denial of Service, отказ в обслуживании) | Атака или происшествие, из-за которых сервис становится недоступен, например из-за слишком большого объёма присланных данных. |
| **КиБ** (KiB, кибибайт) | 1024 байта. `8 KiB` = 8192 байта. |
| **CLI** (Command-Line Interface, интерфейс командной строки) | Программа, которой пользуются, вводя команды и параметры в терминале. |
| **SIGINT** | Сигнал, который терминал посылает, когда вы нажимаете Ctrl-C. Сигнал — это сообщение от операционной системы программе (Этап 13 объясняет сигналы подробно). |
| **приёмочный тест** (acceptance test) | Проверка, показывающая, что готовая программа делает то, что говорит контракт. |
| **README** | Текстовый файл в начале проекта (`README.md`), в котором сказано, что это за проект и как им пользоваться. |
| **API** (Application Programming Interface, программный интерфейс) | Функции и типы, которые библиотека предлагает использующему её коду. |
| **TLS** (Transport Layer Security, защита транспортного уровня) | Шифрование сетевых соединений (буква «S» в HTTPS, то есть HTTP Secure, защищённый HTTP). Наш сервер его не использует; Этап 15 его объясняет. |
| **CI** (Continuous Integration, непрерывная интеграция) | Сервер, который автоматически запускает ваши проверки при каждом push. Этап 09 учит этому. |
| **веха** (milestone) | Один шаг плана с ясной проверкой «готово, когда». |

## 1. Как работает веб-сервер

### TCP и сокеты

Когда вы вводите URL в браузере, происходит три вещи: браузер открывает **TCP-соединение** с IP-адресом и портом сервера, отправляет по нему **HTTP-запрос** в виде текста и читает **HTTP-ответ**. В Rust эти инструменты даёт `std::net`:

- `TcpListener::bind("127.0.0.1:7878")` начинает слушать порт.
- `listener.accept()` (или цикл `listener.incoming()`) ждёт клиента и возвращает `TcpStream`: соединение.
- `TcpStream` реализует `Read` и `Write`, поэтому вы пользуетесь им как файлом.

### HTTP — это текст

Вот что на самом деле отправляет `curl`. Это настоящий вывод крошечного сервера, который печатает полученные байты (стартовая программа из раздела 3):

```text
GET / HTTP/1.1
Host: 127.0.0.1:7878
User-Agent: curl/8.7.1
Accept: */*

```

- Первая строка — **строка запроса**: метод (`GET`), путь (`/`), версия (`HTTP/1.1`).
- Затем идут **заголовки**, по одному в строке.
- Пустая строка заканчивает заголовки. (Каждая строка заканчивается двумя символами `\r\n`.)

Ответ имеет такую же форму:

```text
HTTP/1.1 200 OK
Content-Length: 6
Connection: close

hello
```

**Строка статуса**, заголовки, пустая строка и тело. `Content-Length` сообщает клиенту, сколько байтов тела ждать. `Connection: close` говорит «я закрою соединение после этого ответа», что упрощает наш сервер: один запрос на соединение.

### Зачем нужен пул потоков

Сервер, который обрабатывает клиентов **по одному**, прост, но медленные клиенты блокируют всех. Запуск **нового потока на каждое соединение** тоже прост, но злоумышленник (или напряжённый день) может заставить вас запустить десять тысяч потоков и исчерпать память. **Пул потоков** — золотая середина:

```text
                     ┌─────────────────────┐
 accept loop         │  bounded queue      │        workers (fixed number)
 (main thread)       │  [job][job][job]    │       ┌─────────┐
   TcpListener ──►───┤  (sync_channel)     ├──►────┤ worker 0│─ handle connection
   accept()          │                     │   ┌───┤ worker 1│─ handle connection
        │            └─────────────────────┘   │   │ worker 2│─ handle connection
        │  queue full?                         │   │ worker 3│─ ...
        ▼  answer 503 right away               │   └─────────┘
                                          receiver is shared:
                                          Arc<Mutex<Receiver<Job>>>
```

- Главный поток только **принимает** соединения и кладёт в очередь *задание* («обработать этот поток»).
- Фиксированное число **рабочих потоков** берёт задания из очереди по одному и выполняет их.
- У канала (`mpsc`) много отправителей, но **один** получатель. Чтобы много рабочих потоков могли получать из него, получатель оборачивают в `Arc<Mutex<...>>`: `Arc` делит его, `Mutex` позволяет брать следующее задание одному рабочему потоку за раз.
- У очереди есть **предел**. Если она полна, сервер сразу отвечает `503 Service Unavailable`, а не растёт без границ. Это и есть **обратное давление**.
- Чтобы остановиться, вы **уничтожаете отправителя**. Тогда `recv()` у каждого рабочего потока возвращает ошибку, цикл рабочего заканчивается, и пул присоединяет (join) все потоки. Это **корректное завершение**, и оно естественно ложится на `Drop`.

## 2. Спланируйте историю Git

Относитесь к этому проекту как к настоящей работе. В вашем учебном репозитории (`~/projects/rust`):

1. Работайте в **ветке**, а не в `main`: `git switch -c p06-webserver`.
2. Делайте **один коммит на веху** с сообщением в стиле Conventional Commits из [Урока 01.9](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md): `feat(p06-webserver): M3 router and responses`. Маленький, хорошо названный коммит легко проверять и отменять.
3. В конце **слейте** ветку в `main` (через `git merge --no-ff` или через pull request, если вы используете GitHub или GitLab) и **поставьте тег** на результат: `git tag -a p06-webserver-v1.0 -m "..."`.
4. Убедитесь, что `target/` игнорируется. Осторожно: когда вы выполняете `cargo new` *внутри* существующего репозитория Git, Cargo не создаёт `.gitignore` для нового пакета, поэтому `.gitignore` в корне вашего учебного репозитория должен содержать строку `target/`.

Вот этот процесс, выполненный во временном репозитории. Хеши коммитов у вас будут другие, а в этом небольшом примере вехи нумеруются с M0, тогда как план ниже начинается с M1:

```sh
git switch -c p06-webserver
git add code/p06-webserver && git commit -m "feat(p06-webserver): M0 create the project"
# ... work, then after each milestone ...
git commit -am "feat(p06-webserver): M1 parse requests"
git commit -am "feat(p06-webserver): M2 router and responses"
git commit -am "feat(p06-webserver): M3 thread pool"
git log --oneline
git switch main
git merge --no-ff p06-webserver -m "Merge branch 'p06-webserver': multithreaded web server"
git tag -a p06-webserver-v1.0 -m "p06-webserver: all milestones done"
git log --oneline --graph --decorate
```

```text
ed5a1e7 feat(p06-webserver): M3 thread pool
7a22cba feat(p06-webserver): M2 router and responses
5370d74 feat(p06-webserver): M1 parse requests
1e7d04c feat(p06-webserver): M0 create the project
b675a56 chore: start learning repository
*   3363654 (HEAD -> main, tag: p06-webserver-v1.0) Merge branch 'p06-webserver': multithreaded web server
|\  
| * ed5a1e7 (p06-webserver) feat(p06-webserver): M3 thread pool
| * 7a22cba feat(p06-webserver): M2 router and responses
| * 5370d74 feat(p06-webserver): M1 parse requests
| * 1e7d04c feat(p06-webserver): M0 create the project
|/  
* b675a56 chore: start learning repository
```

> 💡 **Совет:** выполняйте `git status` перед каждым коммитом. В нём не должно быть `target/`. Если веха сломала сборку, всё равно закоммитьте в своей ветке с сообщением вроде `wip: ...` и исправьте в следующем коммите: история в личной ветке может быть неопрятной.

## 3. Разминка: самый маленький сервер

Создайте проект и запустите самый маленький сервер. Это также веха M1.

```sh
cd ~/projects/rust
git switch -c p06-webserver
cargo new code/p06-webserver
cd code/p06-webserver
```

`src/main.rs`:

```rust
use std::io::{Read, Write};
use std::net::TcpListener;

fn main() -> std::io::Result<()> {
    let listener = TcpListener::bind("127.0.0.1:7878")?;
    println!("Listening on http://127.0.0.1:7878");

    for stream in listener.incoming() {
        let mut stream = stream?;

        let mut buffer = [0u8; 1024];
        let n = stream.read(&mut buffer)?;
        println!("received {n} bytes");
        println!("{}", String::from_utf8_lossy(&buffer[..n]));

        let body = "hello\n";
        let response = format!(
            "HTTP/1.1 200 OK\r\nContent-Length: {}\r\nConnection: close\r\n\r\n{}",
            body.len(),
            body
        );
        stream.write_all(response.as_bytes())?;
    }
    Ok(())
}
```

**Что происходит:**

1. `TcpListener::bind` просит операционную систему слушать `127.0.0.1:7878`. `?` возвращает ошибку, если порт занят.
2. `listener.incoming()` — итератор, который ждёт (блокируется), пока клиент подключится, и выдаёт `TcpStream` для каждого соединения.
3. `stream.read(&mut buffer)` читает до 1024 байтов запроса в массив и возвращает, сколько получил. Мы их печатаем.
4. Мы строим текст ответа: строка статуса, заголовки, пустая строка (`\r\n\r\n`), тело. `write_all` отправляет его. Когда `stream` выходит из области видимости в конце тела цикла, соединение закрывается.

Запустите в одном терминале (`cargo run`) и используйте другой терминал:

```sh
curl -i http://127.0.0.1:7878/
```

```text
HTTP/1.1 200 OK
Content-Length: 6
Connection: close

hello
```

Первый терминал напечатает `received 77 bytes` и текст запроса, который вы видели выше. Коммит: `git add -A . && git commit -m "feat(p06-webserver): M1 hello server"`.

Теперь эксперимент, который показывает, зачем нужен проект. Когда этому серверу (или любому серверу, работающему по одному запросу) нужно 2 секунды на каждый запрос, второй клиент ждёт первого. Ваш готовый сервер, запущенный сначала с одним рабочим потоком, а потом с четырьмя, даёт такие настоящие числа для двух одновременных запросов к медленному маршруту `/sleep`:

```text
threads=1
  request 1 took 2.012805s
  request 2 took 4.012874s
threads=4
  request 1 took 2.016573s
  request 2 took 2.018657s
```

С одним рабочим потоком второй запрос ждёт 2 секунды в очереди. С четырьмя оба заканчиваются вместе. К концу проекта вы это воспроизведёте.

## Проект

### Что вы создадите

`p06-webserver`: небольшой HTTP/1.1-сервер, который

- слушает TCP-порт и обрабатывает соединения на **пуле рабочих потоков**;
- отдаёт домашнюю страницу, проверку работоспособности, медленный маршрут и файлы из папки;
- правильно отвечает на ошибки (400, 403, 404, 405, 408, 431, 503);
- защищает себя ограничениями размера, тайм-аутами и ограниченной очередью;
- печатает одну строку **журнала доступа** на запрос;
- корректно останавливается по **Ctrl-C**.

Это библиотека (`src/lib.rs` с модулями) плюс тонкий `main.rs`, с модульными тестами, интеграционным тестом, приёмочным скриптом, README и чистыми `cargo clippy` и `cargo fmt`.

### Контракт

**Командная строка:**

```text
p06-webserver [--addr 127.0.0.1:7878] [--threads 4] [--queue 32] [--root ./public]
```

| Параметр | Значение | По умолчанию |
|---|---|---|
| `--addr` | адрес и порт для прослушивания | `127.0.0.1:7878` |
| `--threads` | число рабочих потоков (не меньше 1) | `4` |
| `--queue` | сколько принятых соединений может ждать свободного рабочего потока | `32` |
| `--root` | папка с `hello.html` и файлами для `/static/` | `./public` |

`--threads 0` печатает `error: --threads must be at least 1` в стандартный поток ошибок и завершается с кодом `2`. Если адрес использовать нельзя (например, порт занят), напечатайте `error: cannot listen on <addr>: <reason>` и завершитесь с кодом `1`.

При запуске печатайте `Listening on http://<addr>` в стандартный вывод.

**Маршруты** (в каждом ответе есть `Content-Type`, `Content-Length` и `Connection: close`):

| Запрос | Ответ |
|---|---|
| `GET /` | `200`, файл `hello.html` из `--root`, как `text/html; charset=utf-8` |
| `GET /health` | `200`, тело `ok` |
| `GET /sleep` | ждёт 2 секунды, затем `200`, тело `slept` + перевод строки |
| `GET /static/<file>` | `200` и файл из `--root`; `Content-Type` зависит от расширения (`.html`, `.txt`, `.css`, `.json`; всё остальное — `application/octet-stream`) |
| любой другой путь | `404`, тело `404 Not Found` + перевод строки |
| известный путь с методом, отличным от `GET` | `405` с заголовком `Allow: GET` |

**Ошибки и ограничения:**

| Ситуация | Ответ |
|---|---|
| Строка запроса не имеет вида `METHOD /path HTTP/x` (не то число частей, путь без начального `/`, версия не начинается с `HTTP/`) | `400` |
| Строка запроса + заголовки больше **8 КиБ** | `431` |
| Нет полного запроса в течение **5 секунд** после подключения | `408` |
| Путь `/static/` выходит за пределы `--root` (`..`, абсолютный путь, символьная ссылка за пределы папки) | `403` |
| Файл `/static/` не существует | `404` |
| Все рабочие потоки заняты и очередь полна | `503`, его пишет сам цикл приёма соединений |

**Журнал доступа** — одна строка в стандартном выводе на каждый обработанный запрос:

```text
<ip> "<METHOD> <path>" <status> <body-bytes> <milliseconds>ms
```

Например, `127.0.0.1 "GET /health" 200 2 0.1ms`. Если запрос не удалось разобрать, пишите `"-"` вместо метода и пути: `127.0.0.1 "-" 431 36 0.1ms`.

**Завершение:** по Ctrl-C (SIGINT) сервер перестаёт принимать новые соединения, даёт завершиться выполняющимся запросам, печатает `Shutting down worker N` для каждого рабочего потока (когда пул уничтожается), печатает `Server stopped` и завершается с кодом `0`.

**Рекомендуемый API библиотеки** (см. «API» выше; чтобы ваши тесты выглядели как в подсказках; названия можно выбрать другие, если сохранить поведение):

```rust
// src/http.rs
pub const MAX_HEADER_BYTES: usize = 8 * 1024;
pub struct Request { pub method: String, pub path: String, pub version: String, pub headers: Vec<(String, String)> }
pub enum ParseError { Closed, BadRequestLine, TooLarge, Timeout, Io(std::io::Error) }
pub fn parse_request<R: std::io::BufRead>(reader: R) -> Result<Request, ParseError>;
pub struct Response { pub status: u16, pub reason: &'static str, pub content_type: &'static str,
                      pub extra_headers: Vec<(&'static str, String)>, pub body: Vec<u8> }
impl Response { pub fn write_to<W: std::io::Write>(&self, out: &mut W) -> std::io::Result<()>; /* + constructors */ }

// src/router.rs
pub enum Resolved { File(std::path::PathBuf), Forbidden, NotFound }
pub fn resolve_static(root: &std::path::Path, relative: &str) -> Resolved;
pub fn route(request: &Request, root: &std::path::Path) -> Response;   // no sockets: easy to test

// src/pool.rs
pub struct PoolFull;
pub struct ThreadPool { /* ... */ }
impl ThreadPool {
    pub fn new(size: usize, queue: usize) -> ThreadPool;
    pub fn execute<F: FnOnce() + Send + 'static>(&self, job: F) -> Result<(), PoolFull>;
}                                                    // impl Drop: join all workers

// src/server.rs
pub struct Server { /* ... */ }
impl Server {
    pub fn bind(addr: &str, threads: usize, queue: usize, root: std::path::PathBuf,
                shutdown: std::sync::Arc<std::sync::atomic::AtomicBool>) -> std::io::Result<Server>;
    pub fn local_addr(&self) -> std::io::Result<std::net::SocketAddr>;
    pub fn run(self);                                // returns after `shutdown` is set
}
```

**Структура проекта:**

```text
code/p06-webserver/
├── Cargo.toml          ← clap и ctrlc
├── README.md           ← что это, как запустить, как тестировать
├── public/
│   ├── hello.html      ← любая небольшая HTML-страница
│   └── note.txt        ← строка текста для /static/note.txt
├── scripts/
│   └── accept.sh       ← приёмочный скрипт из этого урока
├── src/
│   ├── main.rs         ← аргументы, обработчик Ctrl-C, запуск сервера
│   ├── lib.rs          ← объявляет модули
│   ├── http.rs  router.rs  pool.rs  server.rs
└── tests/
    └── server.rs       ← интеграционный тест: запускает настоящий сервер на порту 0
```

Зависимости в `Cargo.toml` (версии из реестра крейтов, проверены 2026-11-17):

```toml
[dependencies]
clap = { version = "4.6", features = ["derive"] }
ctrlc = "3.5"
```

**`ctrlc`** — маленький крейт: `ctrlc::set_handler(move || { ... })` запускает ваше замыкание, когда пользователь нажимает Ctrl-C. В замыкании вы только устанавливаете флаг `AtomicBool`. Остальная программа читает этот флаг.

### Пошаговый план

Коммитьте после каждой вехи в ветке `p06-webserver`. Перед каждым коммитом запускайте `cargo fmt` и `cargo clippy --all-targets -- -D warnings` (`-D warnings` превращает предупреждения в ошибки).

#### M1 — Сервер «hello»

**Цель:** самый маленький сервер из раздела 3 работает, и `curl` получает ответ.

**Указания:** используйте код из разминки. Создайте `public/hello.html` и `public/note.txt`.

<details>
<summary>Подсказка</summary>

Если `cargo run` говорит `Address already in use`, старая копия сервера всё ещё работает. Остановите её через Ctrl-C в её терминале. Если не можете найти, `lsof -i :7878` покажет идентификатор процесса, а `kill <pid>` его остановит.

</details>

**Готово, когда:** `curl -i http://127.0.0.1:7878/` печатает `HTTP/1.1 200 OK` и тело `hello`. Коммит: `feat(p06-webserver): M1 hello server`.

#### M2 — Разбор запроса

**Цель:** функция `parse_request`, которая превращает байты от клиента в `Request`, а ошибки — в `ParseError`. Перенесите код в библиотеку: `src/lib.rs` (с `pub mod http;`) и `src/http.rs`.

**Указания:**

- Принимайте любой `BufRead` (`BufReader<&TcpStream>` на сервере, `&[u8]` в тестах: срез байтов реализует `BufRead`).
- Прочитайте первую строку через `read_line`. Разделите её через `split_whitespace`. В ней должно быть ровно три части; путь должен начинаться с `/`; версия должна начинаться с `HTTP/`. Иначе верните `BadRequestLine`.
- Читайте строки заголовков до пустой строки. Разделяйте каждую по первому `:` (`split_once`) и обрезайте пробелы.
- Если `read_line` в самом начале вернул `0` байтов, клиент закрыл соединение, ничего не прислав: `Closed`.
- **Ограничения уже здесь:** никогда не читайте больше `MAX_HEADER_BYTES` суммарно. Одна бесконечная строка не должна заполнить вашу память. Используйте `reader.take(limit)`, чтобы ограничить один `read_line`. Если предел достигнут без перевода строки, верните `TooLarge`.
- Сопоставьте виды ошибок ввода-вывода `WouldBlock` и `TimedOut` с `Timeout` (так выглядит тайм-аут чтения сокета).

<details>
<summary>Подсказка</summary>

У `BufRead::read_line` нет собственного ограничения длины. `Read::take(n)` оборачивает читателя так, что из него можно прочитать не больше `n` байтов; `(&mut reader).take(n)` заимствует, а не перемещает. Помните про бюджет на все строки вместе: `remaining = MAX - used_so_far`.

</details>

**Тесты, которые нужно написать (в `http.rs`):** простой GET с двумя заголовками разбирается (метод, путь, версия, список заголовков); `"HELLO\r\n\r\n"`, `"GET /x\r\n\r\n"` и `"GET x HTTP/1.1\r\n\r\n"` дают `BadRequestLine`; пустой ввод даёт `Closed`; строка заголовка в 9000 байтов даёт `TooLarge`; 1000 маленьких заголовков дают `TooLarge`.

**Готово, когда:** `cargo test` показывает, что тесты вашего разбора проходят, например:

```text
test http::tests::parses_a_simple_get ... ok
test http::tests::rejects_a_bad_request_line ... ok
test http::tests::empty_input_means_closed ... ok
test http::tests::huge_headers_are_too_large ... ok
test http::tests::many_small_headers_are_too_large_too ... ok
```

Коммит: `feat(p06-webserver): M2 parse requests`.

#### M3 — Маршрутизатор, ответы и безопасные статические файлы

**Цель:** `route(&Request, &Path) -> Response` реализует таблицу маршрутов, а `Response::write_to` пишет корректный HTTP-ответ. Сервер их использует.

**Указания:**

- `Response` хранит статус, текст причины, тип содержимого, дополнительные заголовки и тело. `write_to` пишет строку статуса, `Content-Type`, `Content-Length` (длина тела в **байтах**), `Connection: close`, дополнительные заголовки, пустую строку, тело.
- `route` — чистая функция: на вход `Request`, на выходе `Response`. Никаких сокетов. Поэтому её можно тестировать без сети.
- Делайте `match` по `request.path.as_str()`. Проверяйте метод для каждого маршрута; если путь известен, а метод не `GET`, верните 405 с `Allow: GET`.
- **Статические файлы и обход пути.** `resolve_static(root, relative)` должен (1) отвергать любой компонент пути, который не обычное имя (`..`, `/`, префикс диска): используйте `Path::components()` и `Component::Normal`; (2) канонизировать и `root`, и `root.join(relative)` (`Path::canonicalize` раскрывает `.`/`..` и символьные ссылки и падает, если файла нет); (3) проверить, что канонический путь к файлу `starts_with` канонический корень и `is_file()`.
- Пока подключите это в `server.rs` так: для каждого соединения — разобрать запрос, выбрать маршрут, записать ответ (всё ещё в одном потоке).

<details>
<summary>Подсказка</summary>

`Path::components()` выдаёт `Component::Normal(name)` для обычных имён и `Component::ParentDir` для `..`. `std::fs::read(path)` даёт байты файла. Выбирайте тип содержимого по `path.extension().and_then(|e| e.to_str())`. Помните: если `canonicalize` падает из-за отсутствия файла, это `NotFound`, а не `Forbidden`.

</details>

**Тесты, которые нужно написать (в `router.rs`):** `/health` — 200 с телом `ok`; неизвестный путь — 404; `POST /health` — 405 с `Allow: GET`; `/` возвращает содержимое `hello.html` из временной папки (используйте `std::env::temp_dir()`); `/static/note.txt` отдаётся; `resolve_static` возвращает `Forbidden` для `../Cargo.toml`, `a/../../x` и `/etc/passwd`; отсутствующий статический файл — 404.

**Готово, когда:** запустите сервер (`cargo run`) и выполните:

```sh
curl -i http://127.0.0.1:7878/health
curl -i http://127.0.0.1:7878/nope
curl -i -X POST http://127.0.0.1:7878/health
curl -i --path-as-is http://127.0.0.1:7878/static/../Cargo.toml
```

```text
HTTP/1.1 200 OK
Content-Type: text/plain; charset=utf-8
Content-Length: 2
Connection: close

ok
```

```text
HTTP/1.1 404 Not Found
Content-Type: text/plain; charset=utf-8
Content-Length: 14
Connection: close

404 Not Found
```

```text
HTTP/1.1 405 Method Not Allowed
Content-Type: text/plain; charset=utf-8
Content-Length: 23
Connection: close
Allow: GET

405 Method Not Allowed
```

```text
HTTP/1.1 403 Forbidden
Content-Type: text/plain; charset=utf-8
Content-Length: 14
Connection: close

403 Forbidden
```

(`--path-as-is` не даёт `curl` убрать `..` перед отправкой.) Коммит: `feat(p06-webserver): M3 router and responses`.

> 🔒 **Безопасность:** обход пути — одна из старейших веб-атак: `GET /static/../../etc/passwd`. Никогда не строите путь к файлу из пользовательского ввода, не проверив, что итоговый путь всё ещё внутри разрешённой папки. Проверяйте это явно тестом, как здесь. (Формы с процентным кодированием вроде `%2e%2e` — дополнительная цель; ваш сервер их не декодирует, так что они просто не совпадают ни с одним файлом.)

#### M4 — Пул потоков

**Цель:** `ThreadPool::new(size, queue)` и `execute(job)`. Сервер передаёт каждое принятое соединение пулу. Два одновременных запроса `/sleep` теперь заканчиваются вместе.

**Указания:**

- Используйте `mpsc::sync_channel::<Job>(queue)`, где `type Job = Box<dyn FnOnce() + Send + 'static>`. *Синхронный* канал имеет фиксированную ёмкость. У его отправителя есть `try_send`, который сразу падает, когда очередь полна: `TrySendError::Full`.
- Оберните получателя: `Arc<Mutex<Receiver<Job>>>`. Дайте каждому из `size` рабочих потоков клон `Arc` и `JoinHandle`.
- Цикл рабочего потока: заблокировать, `recv()`, **освободить блокировку**, выполнить задание. Если `recv()` вернул `Err`, отправитель исчез: выйдите из цикла.
- Храните отправителя в `Option<SyncSender<Job>>`, чтобы `Drop` мог сделать `take()`. В `Drop`: уничтожьте отправителя, затем для каждого рабочего потока напечатайте `Shutting down worker {id}` и сделайте `join` его потока (`JoinHandle` рабочего потока тоже лежит в `Option`, так что его можно вынуть через `take()` из `&mut self`).
- `execute` возвращает `Err(PoolFull)`, если `try_send` не удался.

<details>
<summary>Подсказка</summary>

Строка, которая берёт задание, должна завершить блокировку **в той же инструкции**: сохраните результат `receiver.lock().unwrap().recv()` в переменную через `let`, затем используйте её. Временный `MutexGuard` уничтожается на точке с запятой. См. «Частые ошибки» в конце: другая форма держит блокировку на всё время задания, и ваш пул будет выполнять только одно задание за раз.

</details>

**Тесты, которые нужно написать (в `pool.rs`):** десять заданий на трёх рабочих потоках выполняются все, а когда пул уничтожен, они завершены (используйте счётчик `Arc<AtomicUsize>` и проверьте его после `drop`); при одном рабочем потоке и очереди из одного места заблокированное первое задание, второе задание в очереди и третье задание дают `Err(PoolFull)` (используйте канал, чтобы держать первое задание заблокированным, затем отпустите его).

**Готово, когда:** `cargo test` проходит, а при запуске сервера сначала с одним рабочим потоком, потом с четырьмя два одновременных запроса ведут себя как числа в разделе 3:

```sh
cargo run -- --threads 1
# in another terminal:
curl -s -o /dev/null -w "request 1 took %{time_total}s\n" http://127.0.0.1:7878/sleep &
curl -s -o /dev/null -w "request 2 took %{time_total}s\n" http://127.0.0.1:7878/sleep &
wait
```

При `--threads 1` один запрос занимает около 2 с, другой около 4 с. При `--threads 4` оба занимают около 2 с. Коммит: `feat(p06-webserver): M4 thread pool`.

#### M5 — Ограничения: тайм-ауты, большие заголовки, полная очередь

**Цель:** сервер защищает себя.

**Указания:**

- **Тайм-ауты.** На каждом принятом потоке вызовите `set_read_timeout(Some(Duration::from_secs(5)))` и `set_write_timeout(...)`. Чтение, которое ждёт слишком долго, падает с `WouldBlock` или `TimedOut`, ваш разбор сопоставляет это с `Timeout`; отвечайте `408`.
- **Размер заголовков.** `ParseError::TooLarge` становится `431`. `BadRequestLine` становится `400`.
- **Полная очередь.** Когда `pool.execute` возвращает `Err(PoolFull)`, **сам цикл приёма** пишет ответ `503` в этот поток (все рабочие потоки заняты, он не может их попросить) и идёт дальше.
- **Не закрывайте слишком рано.** Если вы записали `431` и закрыли сокет, пока клиент всё ещё отправляет свой гигантский заголовок, операционная система сбросит соединение, и `curl` может ничего не напечатать. После записи ответа с ошибкой вызовите `stream.shutdown(Shutdown::Write)` («я закончил отправлять»), затем прочитайте и отбросьте то, что клиент ещё присылает, до предела (`io::copy(&mut (&stream).take(1 << 20), &mut io::sink())`).
- **Режим потока.** В следующей вехе вы сделаете слушателя неблокирующим. На macOS принятый сокет *наследует* этот режим, и тогда чтения сразу падают с `WouldBlock`. Вызывайте `stream.set_nonblocking(false)` для каждого принятого потока.

<details>
<summary>Подсказка</summary>

Чтобы поток был доступен и заданию, и циклу приёма (для случая 503), используйте `stream.try_clone()`: он даёт второй дескриптор того же соединения. Задание получает клон; если `execute` не удался, оригинал всё ещё у вас в руках.

</details>

**Тесты:** расширьте тесты разбора (тесты размера вы сделали в M2). Тест очереди из M4 покрывает `PoolFull`.

**Готово, когда:**

```sh
curl -i -H "X-Big: $(head -c 9000 /dev/zero | tr '\0' a)" http://127.0.0.1:7878/
```

```text
HTTP/1.1 431 Request Header Fields Too Large
Content-Type: text/plain; charset=utf-8
Content-Length: 36
Connection: close

431 Request Header Fields Too Large
```

Запустите сервер с `--threads 1 --queue 1`, затем быстро запустите четыре медленных запроса и посмотрите, как два из них получают отказ:

```sh
for i in 1 2 3 4; do
  (curl -s -o /dev/null -w "request $i -> %{http_code}\n" http://127.0.0.1:7878/sleep &)
  sleep 0.1
done
```

```text
request 3 -> 503
request 4 -> 503
request 1 -> 200
request 2 -> 200
```

(Порядок строк может отличаться; суть в двух `503` и двух `200`.) А простаивающее соединение получает `408` через 5 секунд:

```sh
(sleep 6 | nc 127.0.0.1 7878) | head -3
```

```text
HTTP/1.1 408 Request Timeout
Content-Type: text/plain; charset=utf-8
Content-Length: 20
```

Коммит: `feat(p06-webserver): M5 limits and timeouts`.

#### M6 — Журнал доступа, командная строка и корректное завершение

**Цель:** строка журнала, параметры из контракта и Ctrl-C.

**Указания:**

- **Журнал:** измеряйте через `Instant::now()` в начале обработчика; печатайте строку после записи ответа. Используйте `println!`; каждый вызов пишет целую строку, поэтому строки из разных потоков не перемешиваются.
- **CLI:** используйте `clap` с `#[derive(Parser)]` (Урок 05.11) и значениями по умолчанию из контракта. Проверку `--threads 0` сделайте сами.
- **Флаг завершения:** создайте `Arc<AtomicBool>`. `ctrlc::set_handler` устанавливает его в `true`. Цикл приёма проверяет его на каждом обороте.
- **Проблема приёма.** `listener.accept()` *блокируется*, поэтому флаг не проверяется, пока не подключится другой клиент. Выберите и **запишите в README** одно решение и его цену: (а) `listener.set_nonblocking(true)`; при `ErrorKind::WouldBlock` поспать 20 мс и снова проверить флаг (просто, но есть крошечная задержка и холостые пробуждения); или (б) в обработчике Ctrl-C ещё и один раз подключиться к собственному адресу, чтобы разбудить `accept` (без опроса, но больше подвижных частей).
- **Завершение работы:** `run(self)` забирает сервер во владение. Когда цикл заканчивается, `self` уничтожается, что уничтожает пул, который печатает `Shutting down worker N` и присоединяет каждый рабочий поток, поэтому выполняющиеся запросы сначала завершаются. Затем `main` печатает `Server stopped` и возвращает `ExitCode::SUCCESS`.

<details>
<summary>Подсказка</summary>

Замыкание, которое передают в `ctrlc::set_handler`, должно быть `'static + Send`, поэтому перемещайте (`move`) в него `Arc::clone` флага. Используйте `Ordering::SeqCst` и для `store`, и для `load`; это самый простой порядок и здесь достаточно быстрый (порядки объясняются на Этапе 14).

</details>

**Готово, когда:** запустите сервер, затем:

```sh
(curl -s -i http://127.0.0.1:7878/sleep > /tmp/sleep.txt &)
sleep 0.5
# now press Ctrl-C in the server's terminal
```

Терминал сервера покажет что-то вроде этого. Простаивающие рабочие потоки останавливаются сразу. Рабочий поток, занятый вашим запросом, останавливается только после его завершения, поэтому положение строки журнала доступа среди строк `Shutting down` может отличаться от запуска к запуску:

```text
Listening on http://127.0.0.1:7878
Shutting down worker 0
127.0.0.1 "GET /sleep" 200 6 2005.2ms
Shutting down worker 1
Shutting down worker 2
Shutting down worker 3
Server stopped
```

а `head -1 /tmp/sleep.txt` показывает `HTTP/1.1 200 OK`: выполнявшийся запрос **не** был оборван. В терминале сервера `echo $?` печатает `0`. Журнал других запросов выглядит так:

```text
127.0.0.1 "GET /health" 200 2 0.1ms
127.0.0.1 "GET /nope" 404 14 0.0ms
127.0.0.1 "POST /health" 405 23 0.1ms
127.0.0.1 "GET /static/../Cargo.toml" 403 14 0.1ms
127.0.0.1 "-" 431 36 0.1ms
```

И аргументы:

```sh
cargo run -- --threads 0; echo "exit: $?"
```

```text
error: --threads must be at least 1
exit: 2
```

Коммит: `feat(p06-webserver): M6 access log, cli and graceful shutdown`.

#### M7 — Качество, README, слияние и тег

**Цель:** проект, который не стыдно кому-то показать.

**Указания:**

- **Интеграционный тест** `tests/server.rs`: привяжите `Server` к `127.0.0.1:0` (порт `0` означает «операционная система выберет свободный порт»; `local_addr()` скажет, какой), запустите его в потоке, отправляйте сырые запросы через `TcpStream`, проверяйте ответы (200 для `/health`, 404, 405 с `Allow: GET`, 400 для `NONSENSE`), затем установите флаг завершения и сделайте `join` потока сервера.
- **Приёмочный скрипт** `scripts/accept.sh` (ниже). Он должен печатать только строки `PASS`.
- **Проверки качества:** `cargo test`, `cargo clippy --all-targets -- -D warnings`, `cargo fmt --check` — всё чисто.
- **README.md** в проекте: что это; как запустить; параметры; маршруты; как запускать тесты и приёмочный скрипт; устройство в пяти строках (цикл приёма, ограниченная очередь, рабочие потоки, завершение); цена выбранного решения для цикла приёма; известные ограничения (один запрос на соединение, нет TLS).
- **Git:** убедитесь, что у каждой вехи есть коммит; слейте с `--no-ff`; поставьте тег `p06-webserver-v1.0`.

Приёмочный скрипт. Сохраните его как `scripts/accept.sh`, запустите сервер (`cargo run`, параметры по умолчанию) и выполните `bash scripts/accept.sh` из папки проекта:

```sh
#!/usr/bin/env bash
# Acceptance checks for p06-webserver.
# Start the server first:  cargo run -- --threads 4
# Run from the project folder:  bash scripts/accept.sh [base-url]
base="${1:-http://127.0.0.1:7878}"
pass=0
fail=0

check() { # check "name" expected actual
    if [ "$2" = "$3" ]; then
        echo "PASS  $1"
        pass=$((pass + 1))
    else
        echo "FAIL  $1 (expected '$2', got '$3')"
        fail=$((fail + 1))
    fi
}

code() { curl -s -o /dev/null -w "%{http_code}" "$@"; }

check "GET /health is 200"          200  "$(code "$base/health")"
check "GET /health body is ok"      ok   "$(curl -s "$base/health")"
check "GET / is 200"                200  "$(code "$base/")"
check "GET /nope is 404"            404  "$(code "$base/nope")"
check "POST /health is 405"         405  "$(code -X POST "$base/health")"
check "405 has Allow: GET"          "Allow: GET" "$(curl -s -i -X POST "$base/health" | tr -d '\r' | grep '^Allow:')"
check "GET /static/note.txt is 200" 200  "$(code "$base/static/note.txt")"
check "missing static file is 404"  404  "$(code "$base/static/missing.txt")"
check "path traversal is refused"   403  "$(code --path-as-is "$base/static/../Cargo.toml")"
check "huge header is 431"          431  "$(code -H "X-Big: $(head -c 9000 /dev/zero | tr '\0' a)" "$base/")"

start=$(date +%s)
curl -s -o /dev/null "$base/sleep" &
curl -s -o /dev/null "$base/sleep" &
wait
elapsed=$(( $(date +%s) - start ))
if [ "$elapsed" -le 3 ]; then
    check "two /sleep requests run together (took ${elapsed}s)" yes yes
else
    check "two /sleep requests run together (took ${elapsed}s)" yes no
fi

echo "passed: $pass, failed: $fail"
[ "$fail" -eq 0 ]
```

**Готово, когда:** при запущенном сервере с параметрами по умолчанию:

```text
PASS  GET /health is 200
PASS  GET /health body is ok
PASS  GET / is 200
PASS  GET /nope is 404
PASS  POST /health is 405
PASS  405 has Allow: GET
PASS  GET /static/note.txt is 200
PASS  missing static file is 404
PASS  path traversal is refused
PASS  huge header is 431
PASS  two /sleep requests run together (took 2s)
passed: 11, failed: 0
```

При `--threads 1` последняя проверка не проходит, как и должно быть (`FAIL  two /sleep requests run together (took 4s)`): это доказывает, что скрипт действительно проверяет конкурентность. И:

```text
$ cargo test
test result: ok. 15 passed; ...      ← модульные тесты (у вас число может отличаться)
test result: ok. 1 passed; ...       ← интеграционный тест
$ cargo clippy --all-targets -- -D warnings
    Finished `dev` profile ...
$ cargo fmt --check && echo "fmt ok"
fmt ok
```

Наконец, из корня репозитория: `git switch main && git merge --no-ff p06-webserver` и `git tag -a p06-webserver-v1.0 -m "p06-webserver: all milestones done"`. Сообщение коммита для последнего шага в ветке: `docs(p06-webserver): M7 README and acceptance script`.

### Дополнительные цели

- **Keep-alive.** Обслуживайте несколько запросов в одном соединении (поведение HTTP/1.1 по умолчанию). Нужны `Connection: keep-alive`, цикл в обработчике и тайм-аут простоя.
- **Процентное декодирование** пути (`%20`, `%2e%2e`): сначала декодируйте, потом проверяйте на обход.
- **Метод `HEAD`** для `/` и `/health` (те же заголовки, без тела).
- **`/metrics`** со счётчиками запросов по кодам статуса (`Arc<Mutex<HashMap<u16, u64>>>` или атомарные переменные), простым текстом.
- **Нагрузочный тест.** Установите `oha` (`cargo install oha`) и выполните `oha -n 2000 -c 50 http://127.0.0.1:7878/health`. Запишите в README число запросов в секунду для `--threads 1`, `4` и `16`. Нагрузочному тестированию позже посвящён целый этап.
- **Сравните с async.** Перепишите тот же сервер на Tokio (Урок 06.9): `tokio::net::TcpListener`, `tokio::spawn` и `Semaphore` для предела очереди. Сравните размер кода и поведение.
- **CI.** Добавьте рабочий процесс GitHub Actions, который запускает `cargo fmt --check`, `cargo clippy` и `cargo test` при каждом push. Непрерывной интеграции как следует учит Этап 09.

## Частые ошибки

**1. Удержание блокировки во время выполнения задания.** Эти два цикла рабочего потока выглядят почти одинаково. Пулом является только один из них:

```rust
// Correct: the guard is a temporary dropped at the end of the `let` statement.
loop {
    let message = rx.lock().unwrap().recv();
    match message {
        Ok(job) => job(),
        Err(_) => break,
    }
}

// Wrong: the guard lives for the whole loop body, so only one worker runs at a time.
while let Ok(job) = rx.lock().unwrap().recv() {
    job();
}
```

Настоящий эксперимент: три рабочих потока и три задания по 200 мс:

```text
let message = ...recv();  -> about 200 ms
while let Ok(job) = ...   -> about 600 ms
```

Временное значение, созданное в условии `while let`, живёт до конца тела. Три задания выполнились одно за другим.

**2. Чтения сразу падают с `WouldBlock` (os error 35).** Слушатель неблокирующий, а на macOS принятые сокеты наследуют это свойство. Вызывайте `stream.set_nonblocking(false)` после `accept`. Настоящая проверка на macOS:

```text
read failed: WouldBlock - Resource temporarily unavailable (os error 35)
```

**3. `curl` ничего не печатает вместо `431`.** Вы закрыли соединение, пока клиент ещё отправлял данные. В одном настоящем тесте без шага «shutdown и слив» `curl` получил ответ только в 1 попытке из 30; с ним — в 30 из 30.

**4. Доверие запросу.** Чтение строк без ограничения, склейка путей без проверки или возможность клиенту держать соединение открытым вечно — каждое даёт способ уронить или заморозить сервер. Каждое ограничение в контракте существует не зря.

**5. Предупреждение clippy, которое вы, вероятно, встретите.** Запись параметра как `root: &PathBuf` даёт:

```text
error: writing `&PathBuf` instead of `&Path` involves a new object where a slice will do
  --> src/server.rs:93:58
   |
93 | fn handle(mut stream: TcpStream, peer: SocketAddr, root: &PathBuf) {
   |                                                          ^^^^^^^^
```

Исправление: принимайте `&Path`. (`&PathBuf` автоматически приводится к `&Path`, поэтому вызывающий код не меняется.)

**6. `unwrap()` на сетевых результатах.** Клиент, который отключился, заставляет `write` упасть с `BrokenPipe`. Игнорируйте такие ошибки отдельного соединения (`let _ = ...`) и держите сервер живым. Останавливать должна только ошибка, из-за которой бесполезен *весь сервер*.

**7. Паники внутри рабочего потока.** Паника в задании убивает этот рабочий поток, и пул постепенно теряет рабочие потоки. Для этого проекта достаточно держать код заданий без паник; дополнительная цель — `std::panic::catch_unwind` вокруг каждого задания.

## Вопросы для самопроверки

1. Почему получатель канала обёрнут в `Arc<Mutex<...>>`?

<details>
<summary>Ответ</summary>

У канала `mpsc` один получатель, но брать из него задания должны многие рабочие потоки. `Arc` позволяет всем рабочим потокам владеть одним получателем; `Mutex` гарантирует, что `recv` вызывает только один рабочий поток за раз.

</details>

2. Что даёт ограниченная очередь (`sync_channel(n)`) по сравнению с неограниченной?

<details>
<summary>Ответ</summary>

Предел. Когда ждут `n` заданий, `try_send` падает, и сервер может сразу ответить `503` (обратное давление), а не использовать всё больше и больше памяти.

</details>

3. Как уничтожение `ThreadPool` останавливает рабочие потоки?

<details>
<summary>Ответ</summary>

`Drop` уничтожает отправителя. Когда все отправители исчезли, `recv()` в каждом рабочем потоке возвращает `Err`, рабочий поток выходит из цикла, а `Drop` затем присоединяет (join) каждый поток, так что выполняющиеся задания сначала завершаются.

</details>

4. Чем опасен `GET /static/../Cargo.toml` и как это остановить?

<details>
<summary>Ответ</summary>

`..` поднимается вверх за пределы раздаваемой папки, так что клиент мог бы прочитать любой файл, который может прочитать сервер. Отвергайте компоненты пути, которые не обычные имена, и проверяйте, что канонический итоговый путь всё ещё внутри канонического корня.

</details>

5. Почему сервер отвечает `503` из цикла приёма, а не из рабочего потока?

<details>
<summary>Ответ</summary>

`503` возникает именно тогда, когда все рабочие потоки заняты и очередь полна, поэтому свободного рабочего потока, который мог бы его записать, нет. Цикл приёма — единственный свободный поток.

</details>

6. Что такое корректное завершение и что делает ваш сервер по Ctrl-C?

<details>
<summary>Ответ</summary>

Остановка после завершения текущей работы. По Ctrl-C обработчик устанавливает флаг, цикл приёма останавливается, пул уничтожается (рабочие потоки заканчивают текущее задание и выходят), печатается `Server stopped`, код выхода равен 0.

</details>

7. Зачем делать один коммит Git на веху?

<details>
<summary>Ответ</summary>

Каждый коммит — маленький работающий именованный шаг: его легко проверять, отменять (`git revert`) и искать в нём ошибку (`git bisect`). История коммитов рассказывает, как развивался проект.

</details>

## Итоги

- Веб-сервер — это цикл: принять TCP-соединение, прочитать HTTP-запрос (текст), записать ответ (текст).
- Пул потоков использует фиксированное число рабочих потоков и канал. Получатель делится как `Arc<Mutex<Receiver<Job>>>`.
- Ограниченная очередь плюс `try_send` даёт обратное давление: отвечайте `503`, когда полно.
- Уничтожение отправителя — чистый сигнал остановиться; `Drop` присоединяет рабочие потоки (корректное завершение).
- Никогда не доверяйте вводу: ограничивайте размеры, ставьте тайм-ауты и проверяйте, что пути к файлам остаются внутри корня.
- Логику, которой не нужны сокеты (`parse_request`, `route`), держите в чистых функциях, чтобы её было легко тестировать.
- Закрывайте соединения осторожно: завершите запись и слейте остаток, иначе клиент может так и не увидеть вашу ошибку.
- Работайте в ветке, коммитьте каждую веху, сливайте и ставьте тег. Перед каждым коммитом запускайте тесты, `clippy` и `fmt`.

## Источники и что почитать

- [The Rust Programming Language — Chapter 21: Final Project: Building a Multithreaded Web Server](https://doc.rust-lang.org/book/ch21-00-final-project-a-web-server.html) — официальное пошаговое руководство, которое этот проект расширяет.
- [std::net::TcpListener](https://doc.rust-lang.org/std/net/struct.TcpListener.html) и [std::net::TcpStream](https://doc.rust-lang.org/std/net/struct.TcpStream.html) — привязка, приём, тайм-ауты, `set_nonblocking`, `shutdown`.
- [std::sync::mpsc::sync_channel](https://doc.rust-lang.org/std/sync/mpsc/fn.sync_channel.html) — ограниченные каналы и `try_send`.
- [MDN (Mozilla Developer Network) — An overview of HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Overview) и [HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status) — язык, на котором говорит ваш сервер.
- [RFC (Request for Comments) 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112) — формальная спецификация формата сообщений (длинная; прочитайте раздел «Message Format»).
- [ctrlc crate](https://docs.rs/ctrlc/latest/ctrlc/) — обработчик Ctrl-C.
- [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/) — соглашение о сообщениях коммитов, используемое в этом проекте.
- [Pro Git — Git Branching](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell) — ветки, слияния и `--no-ff`.

---

[🏠 Начало](../START-HERE.md) · [← 06.10 FFI: вызов кода на C](10-ffi-calling-c.md) · [📋 Этап 06](README.md) · [06.12 Контрольная точка: стадия 1 пройдена →](12-checkpoint-stage-1-complete.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
