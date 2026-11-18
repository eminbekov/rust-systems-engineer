<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 11.12 Контейнеры и основы Docker](12-containers-and-docker-basics.md) · [📋 Этап 11](README.md) · [11.14 Docker Compose →](14-docker-compose.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/11-linux-shell-and-containers/13-dockerfiles-for-rust.md) <!-- nav -->

# Урок 11.13 — Dockerfile для программ на Rust

> **Этап 11 — Linux, командная оболочка и контейнеры** · Урок 13 из 16 · ⏱ около 135 минут

## Что вы узнаете

- Что такое Dockerfile и как `docker build` превращает его в образ.
- Почему наивный образ программы на Rust весит больше двух гигабайт и как **многоэтапная сборка** делает его около 140 МБ (мегабайт), а образ distroless — около 55 МБ.
- Как работает кэш сборки и как сделать его полезным для Rust (слой зависимостей, `.dockerignore`).
- Как запустить службу от пользователя без прав root, с проверкой работоспособности и корректной остановкой.
- Как найти классическую ошибку «порт опубликован, а сервер ничего не отвечает».

## Перед началом

- Вы прошли [Урок 11.12 — Контейнеры и основы Docker](12-containers-and-docker-basics.md). Docker Desktop запущен, вы знаете `docker run`, `-p`, `-e` и почему PID 1 (процесс с идентификатором процесса 1, первый в контейнере) должен обрабатывать `SIGTERM`.
- Вы знаете адрес привязки ([Урок 11.11](11-networking-commands.md)) и как программа на Rust обрабатывает сигналы ([Урок 11.5](05-processes-and-signals.md)).
- Все команды выполняются на **Mac**, в папке `~/projects/rust/code/p11-hello`. Docker собирает образы Linux для архитектуры вашего Mac (arm64).
- Все образы и контейнеры, которые мы создаём, называются `course-hello…`.

## Новые слова

| Слово | Значение |
|---|---|
| **Dockerfile** | Текстовый файл с шагами сборки образа: начать с базового образа, скопировать файлы, выполнить команды, задать команду запуска. |
| **базовый образ** (base image) | Образ, указанный в `FROM`; новый образ строится поверх него. |
| **контекст сборки** (build context) | Папка, файлы которой `docker build` отправляет сборщику. `COPY` может использовать только файлы из неё. |
| **`.dockerignore`** | Файл со списком имён, которые не нужно отправлять в контексте сборки (как `.gitignore` для Docker). |
| **многоэтапная сборка** (multi-stage build) | Dockerfile с несколькими строками `FROM`. Ранние этапы собирают программу; последний этап — маленький образ, который поставляется. |
| **кэш сборки** (build cache) | Docker запоминает результат каждого шага. Если шаг и всё перед ним не изменились, старый результат используется повторно (`CACHED`). |
| **`CMD` / `ENTRYPOINT`** | Команда, которую контейнер выполняет при запуске. `ENTRYPOINT` — постоянная программа, `CMD` — команда или аргументы по умолчанию. |
| **exec-форма / shell-форма** | `CMD ["/app"]` (список: запускает программу напрямую) против `CMD /app` (строка: запускает её через `/bin/sh -c`). |
| **проверка работоспособности** (health check) | Команда, которую Docker регулярно выполняет внутри контейнера; её код возврата говорит, здорова ли служба. |
| **образ distroless** | Минимальный образ без оболочки и менеджера пакетов: только файлы времени выполнения (например, библиотека C), которые нужны программам. |
| **glibc** (GNU C Library; GNU — проект свободного ПО, который написал и многие стандартные инструменты Linux) | Библиотека C большинства дистрибутивов Linux. Обычная программа на Rust под Linux использует её, поэтому образу времени выполнения нужна совместимая версия. |
| **UID** (User Identifier, идентификатор пользователя) | Номер пользователя. `0` — root. Служба без прав root использует другой номер, например `10001`. |

## 1. Служба

Нам нужно что-то маленькое для упаковки. `p11-hello` — крошечная веб-служба: `GET /` отвечает приветствием, `GET /health` — `ok`. Она читает `BIND` и `GREETING` из окружения (двенадцать факторов, [Урок 08.15](../08-software-design/15-the-twelve-factor-app.md)), корректно останавливается по `SIGTERM` и умеет проверять себя командой `p11-hello --check`. Создайте её на Mac:

```bash
cd ~/projects/rust
cargo new code/p11-hello
cd code/p11-hello
cargo add signal-hook@0.4
```

Замените `src/main.rs`:

```rust
use std::io::{ErrorKind, Read, Write};
use std::net::{TcpListener, TcpStream};
use std::process::ExitCode;
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, Ordering};
use std::thread;
use std::time::Duration;

use signal_hook::consts::{SIGINT, SIGTERM};

/// Settings of the service, read from environment variables.
#[derive(Debug, PartialEq)]
struct Config {
    bind: String,
    greeting: String,
}

impl Config {
    /// `get` looks up a variable by name (so tests need no real environment).
    fn from_vars(get: impl Fn(&str) -> Option<String>) -> Config {
        Config {
            bind: get("BIND").unwrap_or_else(|| "127.0.0.1:8080".to_string()),
            greeting: get("GREETING").unwrap_or_else(|| "Hello".to_string()),
        }
    }
}

/// Decides the answer for the first line of an HTTP request.
fn route(request_line: &str, greeting: &str) -> (u16, String) {
    let mut parts = request_line.split_whitespace();
    match (parts.next(), parts.next()) {
        (Some("GET"), Some("/")) => (200, format!("{greeting} from p11-hello\n")),
        (Some("GET"), Some("/health")) => (200, "ok\n".to_string()),
        (Some("GET"), Some(_)) => (404, "not found\n".to_string()),
        _ => (400, "bad request\n".to_string()),
    }
}

/// The short text that follows the status number in the first line of a response.
fn reason(status: u16) -> &'static str {
    match status {
        200 => "OK",
        404 => "Not Found",
        _ => "Bad Request",
    }
}

fn handle(mut stream: TcpStream, greeting: &str) -> std::io::Result<()> {
    stream.set_read_timeout(Some(Duration::from_secs(2)))?;
    let mut buffer = [0u8; 2048];
    let size = stream.read(&mut buffer)?;
    let request = String::from_utf8_lossy(&buffer[..size]);
    let request_line = request.lines().next().unwrap_or("");

    let (status, body) = route(request_line, greeting);
    println!("{request_line} -> {status}");
    write!(
        stream,
        "HTTP/1.1 {status} {}\r\nContent-Length: {}\r\nConnection: close\r\n\r\n{body}",
        reason(status),
        body.len()
    )
}

/// `p11-hello --check`: ask our own /health and report by exit code.
/// (A tiny image has no `curl`, so the program checks itself.)
fn check(bind: &str) -> bool {
    let port = bind.rsplit(':').next().unwrap_or("8080");
    let Ok(mut stream) = TcpStream::connect(format!("127.0.0.1:{port}")) else {
        return false;
    };
    let _ = stream.set_read_timeout(Some(Duration::from_secs(2)));
    let request = "GET /health HTTP/1.1\r\nHost: check\r\nConnection: close\r\n\r\n";
    let mut reply = String::new();
    stream.write_all(request.as_bytes()).is_ok()
        && stream.read_to_string(&mut reply).is_ok()
        && reply.starts_with("HTTP/1.1 200")
}

fn main() -> std::io::Result<ExitCode> {
    let config = Config::from_vars(|name| std::env::var(name).ok());

    if std::env::args().nth(1).as_deref() == Some("--check") {
        return Ok(if check(&config.bind) {
            ExitCode::SUCCESS
        } else {
            ExitCode::FAILURE
        });
    }

    // SIGTERM and SIGINT only set a flag; the loop below notices it.
    let stop = Arc::new(AtomicBool::new(false));
    signal_hook::flag::register(SIGTERM, Arc::clone(&stop))?;
    signal_hook::flag::register(SIGINT, Arc::clone(&stop))?;

    let listener = TcpListener::bind(&config.bind)?;
    listener.set_nonblocking(true)?;
    println!("listening on {}", listener.local_addr()?);

    while !stop.load(Ordering::Relaxed) {
        match listener.accept() {
            Ok((stream, _)) => {
                let result = stream
                    .set_nonblocking(false)
                    .and_then(|()| handle(stream, &config.greeting));
                if let Err(error) = result {
                    eprintln!("connection error: {error}");
                }
            }
            Err(error) if error.kind() == ErrorKind::WouldBlock => {
                thread::sleep(Duration::from_millis(50));
            }
            Err(error) => eprintln!("accept error: {error}"),
        }
    }
    println!("got a stop signal, bye");
    Ok(ExitCode::SUCCESS)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn config_has_defaults() {
        let config = Config::from_vars(|_| None);
        assert_eq!(config.bind, "127.0.0.1:8080");
        assert_eq!(config.greeting, "Hello");
    }

    #[test]
    fn config_reads_variables() {
        let config = Config::from_vars(|name| match name {
            "BIND" => Some("0.0.0.0:3000".to_string()),
            _ => None,
        });
        assert_eq!(config.bind, "0.0.0.0:3000");
    }

    #[test]
    fn routes_requests() {
        assert_eq!(
            route("GET / HTTP/1.1", "Hi"),
            (200, "Hi from p11-hello\n".into())
        );
        assert_eq!(route("GET /health HTTP/1.1", "Hi").0, 200);
        assert_eq!(route("GET /nope HTTP/1.1", "Hi").0, 404);
        assert_eq!(route("", "Hi").0, 400);
    }
}
```

**Что происходит:**

1. `Config::from_vars` принимает функцию, которая ищет переменные, поэтому тестам не нужно настоящее окружение ([Урок 08.15](../08-software-design/15-the-twelve-factor-app.md)). Адрес привязки по умолчанию — безопасная петля.
2. `route` решает, что ответить, по строке запроса; в ней нет ввода-вывода, поэтому тест проверяет её напрямую. `handle` читает запрос, печатает одну строку журнала и пишет ответ (идея Урока 11.11).
3. `signal_hook::flag::register` заставляет `SIGTERM` и `SIGINT` взводить флаг, а не убивать процесс. Слушатель **неблокирующий**: `accept` возвращает `WouldBlock`, когда никто не подключается, и цикл спит 50 мс, так что замечает флаг в течение 50 мс, заканчивает текущий запрос и возвращается. Код возврата — 0.
4. `check` — собственная проверка работоспособности программы: она подключается к своему порту на `127.0.0.1` и ждёт `HTTP/1.1 200`. В маленьком образе нет `curl`, поэтому программа проверяет себя сама (раздел 5).
5. `main` возвращает `ExitCode`, чтобы `--check` мог сообщить Docker об успехе или неудаче.
6. Сервер обслуживает одно соединение за раз: для этого урока достаточно. Урок 06.11 и Этап 14 показывают, как обслуживать много соединений одновременно.

```bash
cargo test
cargo run
```

```text
running 3 tests
test tests::config_has_defaults ... ok
test tests::config_reads_variables ... ok
test tests::routes_requests ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

`cargo run` печатает `listening on 127.0.0.1:8080` и ждёт; `Ctrl + C` останавливает его, теперь со строкой `got a stop signal, bye`. Как всегда, выполните также `cargo clippy` и `cargo fmt`. Команда `cargo test` создала `Cargo.lock`; он нужен Dockerfile.

## 2. Первый Dockerfile

**Dockerfile** — это рецепт. Создайте `Dockerfile.naive` в `code/p11-hello`:

```dockerfile
FROM rust:1.100
WORKDIR /app
COPY . .
RUN cargo build --release
CMD ["./target/release/p11-hello"]
```

| Инструкция | Значение |
|---|---|
| `FROM rust:1.100` | Начать с официального образа Rust (Debian Linux с `rustc`, `cargo` и набором инструментов C). Каждый Dockerfile начинается с `FROM`. |
| `WORKDIR /app` | Создать `/app` и сделать его текущей папкой для следующих шагов и для контейнера. |
| `COPY . .` | Скопировать контекст сборки (текущую папку Mac) в `/app`. |
| `RUN cargo build --release` | Выполнить команду **во время сборки**; изменения файлов становятся новым слоем. |
| `CMD [...]` | Команда, которую контейнер выполняет при запуске. Во время сборки не выполняется. |

Соберите образ. `-f` выбирает файл, `-t` называет образ (`имя:тег`), а последняя `.` — контекст сборки:

```bash
docker build -f Dockerfile.naive -t course-hello:naive . --progress=plain
```

```text
#6 [2/4] WORKDIR /app
#7 [3/4] COPY . .
#8 [4/4] RUN cargo build --release
#8 6.015    Compiling libc v0.2.191
#8 6.015    Compiling signal-hook v0.4.6
#8 6.903    Compiling errno v0.3.15
#8 6.954    Compiling signal-hook-registry v1.4.8
#8 7.226    Compiling p11-hello v0.1.0 (/app)
#8 7.507     Finished `release` profile [optimized] target(s) in 7.10s
#9 naming to docker.io/library/course-hello:naive done
```

(Сокращено. `--progress=plain` печатает весь журнал вместо живого отображения. Первая сборка ещё и скачивает `rust:1.100`, около гигабайта, и занимает больше времени.) Каждый пронумерованный шаг — один слой. Работает, но посмотрите на размер:

```bash
docker image ls course-hello
```

```text
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
course-hello:naive   bef36fef6a39       2.24GB          565MB
```

**2,24 ГБ** (гигабайта) для программы в 600 КБ (килобайт: тысяча байтов). В образе по-прежнему лежат весь набор инструментов Rust, исходный код, папка `target/` со всеми промежуточными файлами и компилятор, который серверу не нужен. Это стоит времени загрузки, места на диске и **поверхности атаки**: каждая лишняя программа в образе — то, чем может воспользоваться злоумышленник.

## 3. Многоэтапная сборка

Лекарство: собирать в одном образе, поставлять другой. В **многоэтапном** Dockerfile несколько строк `FROM`; `COPY --from=` берёт файлы из более раннего этапа. Создайте `Dockerfile` (имя по умолчанию) в `code/p11-hello`:

```dockerfile
# ---- build stage: the full Rust toolchain ----
FROM rust:1.100 AS build
WORKDIR /app

# 1. Build only the dependencies. This layer is reused until Cargo.toml or Cargo.lock change.
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo 'fn main() {}' > src/main.rs \
    && cargo build --release \
    && rm -r src

# 2. Build our own code.
COPY src ./src
RUN touch src/main.rs && cargo build --release

# ---- runtime stage: a small image with only what the program needs ----
FROM debian:trixie-slim
RUN useradd --system --uid 10001 --no-create-home --shell /usr/sbin/nologin app
COPY --from=build /app/target/release/p11-hello /usr/local/bin/p11-hello
USER app
ENV BIND=0.0.0.0:8080
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --retries=3 CMD ["/usr/local/bin/p11-hello", "--check"]
CMD ["/usr/local/bin/p11-hello"]
```

Создайте также `.dockerignore` (раздел 4 объясняет его):

```text
target
.git
Dockerfile*
```

Соберите и сравните:

```bash
docker build -t course-hello:1 .
docker image ls course-hello
```

```text
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
course-hello:1       76c341ce8a35        141MB         30.4MB
course-hello:naive   bef36fef6a39       2.24GB          565MB
```

От 2,24 ГБ до **141 МБ**. (`DISK USAGE` — размер в распакованном виде на диске; `CONTENT SIZE` — сжатый размер, который передаётся по сети.)

**Что происходит:**

1. `FROM rust:1.100 AS build` начинает первый этап и называет его `build`. Все инструменты Rust остаются на этом этапе.
2. Последний `FROM debian:trixie-slim` начинает **новый образ** с небольшого Debian. Результатом становится только этот этап. Этап сборки выбрасывается (он остаётся в кэше сборки, но не в образе).
3. `COPY --from=build …` берёт **один файл** — скомпилированную программу — с первого этапа.
4. `useradd` создаёт системного пользователя `app` с фиксированным номером 10001, без домашней папки и без оболочки входа; `USER app` заставляет контейнер работать от этого пользователя, **а не от root**. Проверьте позже через `docker top`.
5. `ENV BIND=0.0.0.0:8080` задаёт умолчание для контейнера: слушать все адреса (Урок 11.11) на порту выше 1024. `EXPOSE 8080` — документация; он ничего не публикует.
6. `HEALTHCHECK` каждые 10 секунд запускает внутри контейнера `p11-hello --check` (раздел 5).
7. `CMD ["/usr/local/bin/p11-hello"]` записан в **exec-форме**: сама программа становится PID 1 (раздел 6).

Слои результата немногочисленны и малы:

```text
0B	CMD ["/usr/local/bin/p11-hello"]
0B	HEALTHCHECK {Test:[CMD /usr/local/bin/p11-he…
0B	EXPOSE [8080/tcp]
0B	ENV BIND=0.0.0.0:8080
0B	USER app
582kB	COPY /app/target/release/p11-hello /usr/loca…
41kB	RUN /bin/sh -c useradd --system --uid 10001 …
110MB	# debian.sh --arch 'arm64' out/ 'trixie' '@1…
```

(`docker history --format '{{.Size}}\t{{.CreatedBy}}' course-hello:1`; самый новый слой первым.) Из 141 МБ 110 МБ — основа Debian, а 0,58 МБ — наша программа.

> 💡 **Совет:** почему `debian:trixie-slim`? Обычная программа на Rust под Linux компонуется с библиотекой C **glibc**. Программа должна работать с glibc не старее той, с которой она собрана. `rust:1.100` построен на Debian 13 «trixie», поэтому образ времени выполнения тоже Debian 13. Если поставить программу, собранную в более новой системе, в более старую, получите ошибки вида ``version `GLIBC_2.xx' not found``. Ошибка `exec format error` означает, что программа собрана для другой архитектуры процессора (например, amd64 на машине arm64).

## 4. Кэш сборки

Docker сохраняет результат каждого шага. При повторной сборке он использует шаг повторно, если **инструкция и всё перед ней не изменились** (а для `COPY` ещё и скопированные файлы не изменились). Для Rust это важно, потому что компиляция зависимостей — самая медленная часть. Поэтому Dockerfile собирает в два шага:

1. Скопировать только `Cargo.toml` и `Cargo.lock`, создать фиктивный `main.rs` и собрать. Это компилирует все **зависимости** и кладёт их в один слой.
2. Скопировать настоящий `src` и собрать снова. Компилируется только наш собственный крейт.

Измените приветствие по умолчанию в `src/main.rs` (например, на `"Hello again"`) и соберите снова:

```bash
docker build -t course-hello:1 . --progress=plain
```

```text
#8 [build 2/6] WORKDIR /app
#8 CACHED
#9 [build 3/6] COPY Cargo.toml Cargo.lock ./
#9 CACHED
#10 [build 4/6] RUN mkdir src && echo 'fn main() {}' > src/main.rs     && cargo build --release     && rm -r src
#10 CACHED
#11 [build 5/6] COPY src ./src
#12 [build 6/6] RUN touch src/main.rs && cargo build --release
#12 0.195    Compiling p11-hello v0.1.0 (/app)
#12 0.453     Finished `release` profile [optimized] target(s) in 0.28s
```

Шаги со 2 по 4, с зависимостями, — `CACHED`. Заново скомпилирован только наш крейт. (С одной маленькой зависимостью экономия составляет несколько секунд; в настоящих службах с десятками крейтов — минуты.) Важны три детали:

- `touch src/main.rs` гарантирует, что Cargo заметит настоящий исходный код: иначе фиктивный файл из шага 4 окажется новее скопированного, Cargo решит, что ничего не изменилось, и поставит пустую программу.
- **`.dockerignore`** не пускает `target/` (гигабайты в настоящем проекте, да ещё и собранные для macOS!), `.git` и Dockerfile в контекст сборки. Без него `COPY` отправил бы их сборщику, контекст стал бы огромным, а кэш ломался бы при каждом изменении файла сборки. Размер контекста виден в журнале сборки: `transferring context: …`.
- Порядок инструкций — это порядок кэша: редко меняющееся (базовый образ, зависимости) ставьте первым, а часто меняющееся (ваш исходный код) — последним.

> ⚡ **Производительность:** инструмент `cargo-chef` автоматизирует слой зависимостей для больших рабочих пространств, а BuildKit может хранить кэш загрузок Cargo между сборками через `RUN --mount=type=cache,target=/usr/local/cargo/registry …`. Оба описаны в документации Docker про [кэш сборки](https://docs.docker.com/build/cache/). Измеряйте, прежде чем их добавлять.

## 5. Запуск: порты, адрес привязки и проверка работоспособности

```bash
docker run -d --name course-hello -p 18080:8080 course-hello:1
curl -s http://localhost:18080/
docker top course-hello -eo user,pid,comm
docker exec course-hello id
```

```text
Hello from p11-hello
USER                PID                 COMMAND
10001               46281               p11-hello
uid=10001(app) gid=999(app) groups=999(app)
```

Служба отвечает и работает от пользователя 10001, а не от root. Проверка работоспособности видна в `docker ps`:

```bash
docker ps --filter name=course-hello --format '{{.Names}}  {{.Status}}'
```

```text
course-hello  Up 1 second (health: starting)
course-hello  Up 13 seconds (healthy)
```

(Одна и та же команда: один раз сразу после запуска и один раз через 12 секунд.) Docker выполнил внутри контейнера `p11-hello --check`, получил код 0 и пометил контейнер как **healthy** (здоровый). Оркестраторы и Compose (Урок 11.14) используют это, чтобы решить, когда служба готова и нужно ли её перезапускать.

Теперь классическая ошибка из Урока 11.11. Удалите контейнер и запустите его снова с адресом **петли**:

```bash
docker rm -f course-hello
docker run -d --name course-hello -p 18081:8080 -e BIND=127.0.0.1:8080 course-hello:1
curl -sS -m 3 http://localhost:18081/; echo "curl status $?"
docker exec course-hello /usr/local/bin/p11-hello --check; echo "check inside: $?"
```

```text
curl: (56) Recv failure: Connection reset by peer
curl status 56
check inside: 0
```

Порт опубликован, но сервер слушает только собственную петлю контейнера, поэтому трафик, который Docker приносит на сетевой интерфейс контейнера, никого не находит: соединение сбрасывается. Проверка **изнутри** всё равно проходит, потому что она тоже использует петлю; через 13 секунд `docker ps` сообщит `healthy` для службы, до которой никто не может достучаться. Запомните: **проверка работоспособности изнутри не доказывает, что опубликованный порт работает**. Всегда проверяйте снаружи через `curl`. Исправление — переменная окружения:

```bash
docker rm -f course-hello
docker run -d --name course-hello -p 18081:8080 -e BIND=0.0.0.0:8080 -e GREETING=Salom course-hello:1
curl -sS -m 3 http://localhost:18081/
docker rm -f course-hello
```

```text
Salom from p11-hello
```

> 🔒 **Безопасность:** в контейнере `0.0.0.0` — правильно, потому что у контейнера своя сеть. Кто сможет достучаться до службы, вы решаете при **публикации** порта: `-p 127.0.0.1:18081:8080` оставляет её только на вашем Mac.

## 6. Остановка: exec-форма против shell-формы

Остановите контейнер и прочитайте его журнал:

```bash
docker run -d --name course-hello -p 18080:8080 course-hello:1
curl -s http://localhost:18080/ > /dev/null
SECONDS=0; docker stop course-hello; echo "took ${SECONDS}s"
docker logs course-hello
docker inspect course-hello --format 'exit={{.State.ExitCode}}'
docker rm course-hello
```

```text
course-hello
took 0s
listening on 0.0.0.0:8080
GET / HTTP/1.1 -> 200
got a stop signal, bye
exit=0
```

`SIGTERM` дошёл до нашей программы сразу, она завершилась и вышла с кодом 0 менее чем за секунду. Это работает, потому что `CMD` записан в **exec-форме** (список JSON; JSON, JavaScript Object Notation, — текстовый формат со списками вида `["…", "…"]`): Docker запускает программу напрямую, и она — PID 1. Сравните с **shell-формой**, `CMD /usr/local/bin/p11-hello` (без скобок). Тогда Docker запускает `/bin/sh -c "/usr/local/bin/p11-hello"`. В таком образе `docker top` показывает:

```text
PID                 COMMAND             COMMAND
61500               sh                  /bin/sh -c /usr/local/bin/p11-hello
61514               p11-hello           /usr/local/bin/p11-hello
```

PID 1 — это оболочка, а наша программа — её дочерний процесс. `SIGTERM` приходит оболочке, которая его не пересылает. `docker stop` прождал весь льготный период (по документации по умолчанию это 10 секунд, а `docker stop -t СЕКУНДЫ` его меняет; на тестовой машине вышло около 3 секунд), затем убил всё: журнал заканчивается строкой `listening on 0.0.0.0:8080` без `got a stop signal`, а код возврата — `137` (убит). **Всегда записывайте `CMD` и `ENTRYPOINT` в exec-форме.**

## 7. Меньше и безопаснее: distroless

В `debian:trixie-slim` всё ещё есть оболочка, `apt` и десятки инструментов. Образ **distroless** хранит только файлы, нужные программе для работы: библиотеку C, сертификаты, пользователя без прав root. Их публикует Google; `gcr.io/distroless/cc-debian13` подходит к нашей glibc. Создайте `Dockerfile.distroless`:

```dockerfile
FROM rust:1.100 AS build
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
RUN mkdir src && echo 'fn main() {}' > src/main.rs \
    && cargo build --release \
    && rm -r src
COPY src ./src
RUN touch src/main.rs && cargo build --release

FROM gcr.io/distroless/cc-debian13:nonroot
COPY --from=build /app/target/release/p11-hello /p11-hello
ENV BIND=0.0.0.0:8080
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --retries=3 CMD ["/p11-hello", "--check"]
ENTRYPOINT ["/p11-hello"]
```

```bash
docker build -f Dockerfile.distroless -t course-hello:2 .
docker image ls course-hello
```

```text
IMAGE                ID             DISK USAGE   CONTENT SIZE   EXTRA
course-hello:1       3590ce5b3761        141MB         30.4MB
course-hello:2       7ce130e19ae3       54.6MB         11.2MB
course-hello:naive   bef36fef6a39       2.24GB          565MB
```

| Образ | Размер на диске | Есть оболочка | Пользователь |
|---|---|---|---|
| `rust:1.100` + всё (наивный) | 2,24 ГБ | да | root |
| многоэтапная сборка на `debian:trixie-slim` | 141 МБ | да | `app` (10001) |
| многоэтапная сборка на distroless `cc-debian13:nonroot` | 54,6 МБ | нет | `nonroot` (65532) |

Тег `:nonroot` делает номером пользователя по умолчанию 65532. Оболочки нет, поэтому `docker exec` нечего запускать:

```text
OCI runtime exec failed: exec failed: unable to start container process: exec: "sh": executable file not found in $PATH
```

Это достоинство (злоумышленник не найдёт оболочки), но отладка усложняется. Используйте `docker logs`, тесты на этапе сборки или временно образ `gcr.io/distroless/cc-debian13:debug-nonroot`, в котором есть небольшая оболочка. Помните, что **проверка работоспособности здесь не может использовать `curl`**; именно поэтому существует `p11-hello --check`.

**Дополнительное ужесточение при запуске.** Docker может отнять ещё больше прав. Службе они не нужны:

```bash
docker run -d --name course-hello --read-only --cap-drop ALL --security-opt no-new-privileges \
  --memory 64m --pids-limit 50 -p 18080:8080 course-hello:2
curl -s http://localhost:18080/
docker stats --no-stream --format '{{.Name}} {{.MemUsage}}' course-hello
docker rm -f course-hello
```

```text
Hello from p11-hello
course-hello 1.113MiB / 64MiB
```

`--read-only` делает файловую систему контейнера доступной только для чтения (программа ничего не пишет), `--cap-drop ALL` убирает все особые возможности (capabilities) Linux, `no-new-privileges` запрещает получать права, а ограничения задают потолок памяти и числа процессов. Служба использует 1,1 МиБ (мебибайта, единиц по 1024 × 1024 байтов). Включённые параметры можно прочитать обратно командой `docker inspect course-hello --format '{{.HostConfig.ReadonlyRootfs}} {{.HostConfig.CapDrop}} {{.HostConfig.Memory}}'`, которая печатает `true [ALL] 67108864`.

> 🔒 **Безопасность:** правило для боевых образов: маленькая основа, пользователь без прав root, никаких секретов внутри, точные теги версий, только нужные программе файлы. На Этапе 10 вы научились проверять зависимости через `cargo audit` ([Урок 10.13](../10-security-fundamentals/13-supply-chain-security.md)); запускайте его и перед сборкой образа.

Уберите собранные вами образы:

```bash
docker image rm course-hello:naive course-hello:1 course-hello:2
```

## Частые ошибки

1. **`COPY` не срабатывает или образ огромный**, потому что `target/` попал в контекст. *Исправление:* `.dockerignore` с `target`.
2. **Кэш никогда не помогает**, потому что `COPY . .` стоит до сборки зависимостей. *Исправление:* сначала копируйте `Cargo.toml` и `Cargo.lock`, соберите, затем копируйте `src`.
3. **Контейнер поставляет пустую программу** (ничего не печатает), потому что закэширована сборка с фиктивным `main.rs`. *Исправление:* `touch src/main.rs` перед настоящей сборкой.
4. **`CMD /app` в shell-форме** прячет `SIGTERM` от программы. *Исправление:* `CMD ["/app"]`.
5. **Сервер в контейнере привязан к `127.0.0.1`.** Сброс соединения или пустой ответ снаружи. *Исправление:* `BIND=0.0.0.0:…` и проверка через `curl` с хоста.
6. **Работа от root** (нет `USER`). *Исправление:* создайте пользователя и переключитесь на него; или используйте образ `:nonroot`.
7. **Незафиксированные теги и секреты в сборке** (`FROM rust:latest`, `ENV API_KEY=…`, `COPY .env`). Теги переезжают, а секреты остаются в слоях навсегда. *Исправление:* фиксируйте `rust:1.100` и передавайте секреты, когда контейнер **запускается**.

## Попробуйте сами

1. Соберите `Dockerfile.naive` и многоэтапный `Dockerfile`; сравните `docker image ls course-hello`.
2. Запустите многоэтапный образ с `-p 127.0.0.1:18080:8080`, обратитесь к `/` и `/health` и прочитайте `docker logs`.
3. Запустите его с `-e BIND=127.0.0.1:8080` и посмотрите на сбой. Затем исправьте.
4. Остановите его через `docker stop` и прочитайте код возврата. Затем в копии Dockerfile замените `CMD` на shell-форму и сравните время остановки.
5. Удалите созданные образы и контейнеры.

## Упражнения

### Упражнение 1 — Прочитайте журнал сборки (легко)

Измените один символ приветствия в `src/main.rs` и пересоберите многоэтапный образ с `--progress=plain`. Какие шаги помечены `CACHED`, какие выполняются снова и почему? Что произойдёт, если изменить `Cargo.toml` (добавить строку-комментарий)?

<details>
<summary>Подсказка</summary>

Шаги перед первым изменившимся входом остаются в кэше; после него все шаги выполняются заново.

</details>

<details>
<summary>Ответ</summary>

После изменения в `src/main.rs`: базовые образы, `WORKDIR`, `COPY Cargo.toml Cargo.lock` и сборка зависимостей — `CACHED`. `COPY src ./src` и `RUN touch … && cargo build --release` выполняются снова, как и итоговый `COPY --from=build`. Если добавить комментарий в `Cargo.toml`, шаг `COPY Cargo.toml Cargo.lock ./` видит изменённый файл, поэтому **все шаги после него** выполняются заново, включая сборку зависимостей. Поэтому перед шагом зависимостей копируются только `Cargo.toml` и `Cargo.lock`: они меняются намного реже, чем исходный код.

</details>

### Упражнение 2 — Новый маршрут, тест и пересборка (средне)

Добавьте маршрут `GET /version`, который отвечает версией пакета (`env!("CARGO_PKG_VERSION")`). Сначала напишите тест (`cargo test` должен упасть), затем сделайте так, чтобы он проходил, пересоберите образ и проверьте `curl http://localhost:18080/version` в контейнере.

<details>
<summary>Подсказка</summary>

`env!("CARGO_PKG_VERSION")` во время компиляции заменяется версией из `Cargo.toml`. Добавьте ещё одну ветку `match` в `route` перед веткой `404` и ещё один `assert_eq!` в `routes_requests`.

</details>

<details>
<summary>Ответ</summary>

В `route` добавьте после ветки `/health`:

```rust
        (Some("GET"), Some("/version")) => (200, format!("{}\n", env!("CARGO_PKG_VERSION"))),
```

а в тесте:

```rust
        assert_eq!(
            route("GET /version HTTP/1.1", "Hi"),
            (200, "0.1.0\n".into())
        );
```

Пока вы не добавили ветку, тест падает с `left: (404, "not found\n")`, потому что маршрут попадает в ветку `404`. После этого `cargo test` печатает `test result: ok. 3 passed`. Затем:

```bash
docker build -t course-hello:1 .
docker run -d --name course-hello -p 18080:8080 course-hello:1
curl -s http://localhost:18080/version
docker rm -f course-hello
```

Последний `curl` печатает `0.1.0`.

Почему это работает: `env!` читает переменную `CARGO_PKG_VERSION`, которую Cargo задаёт во время сборки, поэтому бинарный файл несёт свою версию. Поскольку зависимости закэшированы, пересборка компилирует только наш крейт.

</details>

## Вопросы для самопроверки

1. Почему наивный образ весит 2,24 ГБ и что меняет многоэтапная сборка?

<details>
<summary>Ответ</summary>

В нём весь набор инструментов Rust, исходники и папка `target/`. Многоэтапная сборка компилирует на одном этапе и копирует в маленький итоговый образ только готовую программу.

</details>

2. В каком порядке Dockerfile для Rust должен копировать файлы, чтобы кэш помогал?

<details>
<summary>Ответ</summary>

Сначала `Cargo.toml` и `Cargo.lock`, собрать зависимости; затем скопировать `src` и собрать программу. Зависимости меняются редко, исходники часто, поэтому дорогой слой зависимостей остаётся в кэше.

</details>

3. Почему `CMD ["/app"]` лучше, чем `CMD /app`?

<details>
<summary>Ответ</summary>

Exec-форма запускает программу напрямую как PID 1, поэтому она получает `SIGTERM` от `docker stop`. Shell-форма запускает `/bin/sh -c`, оболочка становится PID 1 и не пересылает сигнал, поэтому остановка ждёт льготный период и затем убивает.

</details>

4. Служба в контейнере слушает `127.0.0.1:8080`. Почему до неё не достучаться через `-p 8080:8080` и почему внутренняя проверка работоспособности всё равно проходит?

<details>
<summary>Ответ</summary>

Опубликованный трафик приходит на сетевой интерфейс контейнера, а не на его петлю. Проверка работоспособности выполняется внутри контейнера и использует петлю, поэтому успешна. Служба должна слушать `0.0.0.0`.

</details>

5. Почему проверка работоспособности использует `p11-hello --check`, а не `curl`?

<details>
<summary>Ответ</summary>

В маленьком образе или образе distroless нет ни `curl`, ни оболочки. Программа содержит собственную проверку, поэтому образу ничего лишнего не нужно.

</details>

## Итоги

- Dockerfile — список инструкций; каждый шаг — кэшируемый слой. `docker build -t имя:тег .` собирает его, а `.` — контекст сборки.
- Наивный образ Rust — больше двух гигабайт. Многоэтапная сборка компилирует на этапе `rust:1.100` и поставляет только бинарный файл в `debian:trixie-slim` (141 МБ) или distroless `cc-debian13` (55 МБ).
- Кэшируйте зависимости: сначала скопируйте `Cargo.toml` и `Cargo.lock`, соберите с фиктивным `main.rs`, затем скопируйте `src` и сделайте `touch`. Используйте `.dockerignore`.
- Работайте от пользователя без прав root, пишите `CMD`/`ENTRYPOINT` в exec-форме, чтобы программа была PID 1 и обрабатывала `SIGTERM`, и читайте конфигурацию из переменных окружения.
- Внутри контейнера слушайте `0.0.0.0`; кто сможет достучаться до службы, решайте способом публикации порта. Проверка работоспособности изнутри не доказывает, что порт работает снаружи.
- Основа времени выполнения должна соответствовать glibc образа сборки; фиксируйте версии всех образов.

## Источники и что почитать

- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/) — каждая инструкция, включая `HEALTHCHECK`, `USER` и формы `CMD` и `ENTRYPOINT`.
- [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/) — официальное руководство по этапам и `COPY --from`.
- [Build cache](https://docs.docker.com/build/cache/) — как работает инвалидация и монтирования кэша.
- [Building best practices](https://docs.docker.com/build/building/best-practices/) — фиксация версий, `.dockerignore`, маленькие образы.
- [Distroless images](https://github.com/GoogleContainerTools/distroless) — образы, их теги (`nonroot`, `debug`) и что в них входит.
- [Docker Official Image: rust](https://hub.docker.com/_/rust) и [debian](https://hub.docker.com/_/debian) — доступные теги и их базовые системы.

---

[🏠 Начало](../START-HERE.md) · [← 11.12 Контейнеры и основы Docker](12-containers-and-docker-basics.md) · [📋 Этап 11](README.md) · [11.14 Docker Compose →](14-docker-compose.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
