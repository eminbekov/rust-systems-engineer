<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 16.12 Логи, метрики и трассировки с OpenTelemetry](12-logs-metrics-and-traces.md) · [📋 Этап 16](README.md) · [16.14 Тестирование бэкенд-сервиса →](14-testing-a-backend-service.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/16-backend-services/13-configuration-health-checks-and-containers.md) <!-- nav -->

# Урок 16.13 — Конфигурация, проверки работоспособности и контейнеры

> **Этап 16 — Бэкенд-сервисы** · Урок 13 из 16 · ⏱ около 135 минут

## Что вы узнаете

- Как читать конфигурацию сервиса из переменных окружения в типизированную структуру, проверять её при запуске и не пускать секреты в логи.
- Чем проверка живости отличается от проверки готовности и как написать обе на axum.
- Как корректно остановиться, когда Docker отправляет `SIGTERM`.
- Как упаковать сервис в маленький образ без прав root с проверкой работоспособности, которой не нужна командная оболочка.
- Как запускать сервис и его базу данных вместе через Docker Compose в правильном порядке.

## Перед началом

- Вы прошли [Урок 16.12 — Логи, метрики и трассировки с OpenTelemetry](12-logs-metrics-and-traces.md).
- Вы знаете правила двенадцати факторов, особенно «храните конфигурацию в окружении» ([Урок 08.15](../08-software-design/15-the-twelve-factor-app.md)).
- Вы знаете многоэтапные Dockerfile для Rust ([Урок 11.13](../11-linux-shell-and-containers/13-dockerfiles-for-rust.md)) и файлы Compose ([Урок 11.14](../11-linux-shell-and-containers/14-docker-compose.md)), сигналы ([Урок 11.5](../11-linux-shell-and-containers/05-processes-and-signals.md)) и корректное завершение ([Урок 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md)).
- Вы умеете подключаться к PostgreSQL через sqlx ([Урок 16.6](06-databases-with-sqlx.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **конфигурация** (configuration) | Настройки, которые меняются между окружениями (ваш ноутбук, тест, продакшен), но не между запусками: адреса, порты, пароли. |
| **быстрый отказ** (fail fast) | Остановиться при запуске с понятным сообщением, когда что-то не так, вместо того чтобы упасть позже посреди запроса. |
| **скрыть** (redact) | Заменить секретный текст заполнителем вроде `<redacted>` перед печатью. |
| **проверка работоспособности** (health check) | Небольшой запрос, который инструмент регулярно отправляет сервису: «ты в порядке?». |
| **живость** (liveness) | «Процесс жив и может отвечать». Если это не так, сервис нужно перезапустить. |
| **готовность** (readiness) | «Сервис может делать настоящую работу сейчас» (его база данных отвечает). Если нет — не посылайте ему трафик, но и не перезапускайте. |
| **оркестратор** (orchestrator) | Программа, которая запускает контейнеры, следит за ними и перезапускает их: Docker Compose на одной машине, Kubernetes на многих (Этап 21). |
| **`SIGTERM`** | Сигнал «пожалуйста, остановись», который первым отправляют `docker stop` и Kubernetes. После периода ожидания (10 с в Docker) они отправляют `SIGKILL`, который нельзя перехватить. |
| **distroless-образ** (distroless image) | Базовый образ только с тем, что нужно программе для работы (библиотеки C, сертификаты), без командной оболочки и менеджера пакетов. |
| **пользователь без прав root** (non-root user) | Обычный пользователь без прав администратора. Запуск контейнера не от root ограничивает вред от взлома. |
| **код возврата** (exit code) | Число, которое программа возвращает при завершении: `0` — успех, всё остальное — неудача. Его читают инструменты проверки работоспособности. |

## 1. Конфигурация из окружения

Сервису нужны настройки: на каком адресе слушать, где база данных, какой пароль использовать. Правило двенадцати факторов (Урок 08.15) говорит: читайте их из **переменных окружения**, чтобы один и тот же образ работал на ноутбуке и в продакшене с разными значениями.

Три правила делают конфигурацию безопасной:

1. **Читайте всё один раз, при запуске, в одну типизированную структуру.** Остальной код использует `config.port`, а не `std::env::var("APP_PORT")` посреди обработчика.
2. **Быстрый отказ.** Отсутствующее или испорченное значение сразу останавливает программу с сообщением, в котором названа переменная.
3. **Никогда не печатайте секреты.** В URL (Uniform Resource Locator, адрес) базы данных есть пароль.

Создайте проект:

```bash
cd ~/projects/rust/code
cargo new p16-config-health
cd p16-config-health
cargo add axum@0.8 tracing@0.1 tracing-subscriber@0.3
cargo add sqlx@0.9 --features postgres,runtime-tokio
cargo add tokio@1 --features macros,rt-multi-thread,signal,time,net
```

Создайте `src/config.rs`:

```rust
use std::fmt;
use std::net::{IpAddr, SocketAddr};

/// Все настройки сервиса, прочитанные один раз при запуске.
#[derive(Clone)]
pub struct Config {
    pub host: IpAddr,
    pub port: u16,
    pub database_url: String,
}

#[derive(Debug, PartialEq)]
pub enum ConfigError {
    Missing(&'static str),
    Invalid { key: &'static str, value: String },
}

impl fmt::Display for ConfigError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            ConfigError::Missing(key) => write!(f, "environment variable {key} is not set"),
            ConfigError::Invalid { key, value } => write!(f, "{key} has an invalid value: {value:?}"),
        }
    }
}

impl std::error::Error for ConfigError {}

impl Config {
    /// Читает настоящее окружение.
    pub fn from_env() -> Result<Self, ConfigError> {
        Self::from_lookup(|key| std::env::var(key).ok())
    }

    /// Читает настройки через `get`, чтобы тесты могли передать свои значения.
    pub fn from_lookup(get: impl Fn(&str) -> Option<String>) -> Result<Self, ConfigError> {
        let host = parse_or(&get, "APP_HOST", IpAddr::from([127, 0, 0, 1]))?;
        let port = parse_or(&get, "APP_PORT", 8080)?;
        let database_url = get("DATABASE_URL").ok_or(ConfigError::Missing("DATABASE_URL"))?;
        Ok(Config {
            host,
            port,
            database_url,
        })
    }

    pub fn listen_addr(&self) -> SocketAddr {
        SocketAddr::new(self.host, self.port)
    }
}

/// Разбирает необязательную переменную или возвращает значение по умолчанию, если она не задана.
fn parse_or<T: std::str::FromStr>(
    get: &impl Fn(&str) -> Option<String>,
    key: &'static str,
    default: T,
) -> Result<T, ConfigError> {
    match get(key) {
        None => Ok(default),
        Some(value) => value.parse().map_err(|_| ConfigError::Invalid { key, value }),
    }
}

/// Никогда не печатайте URL базы данных: в нём пароль.
impl fmt::Debug for Config {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Config")
            .field("host", &self.host)
            .field("port", &self.port)
            .field("database_url", &"<redacted>")
            .finish()
    }
}
```

**Что происходит:**

1. `Config` хранит типизированные значения: `IpAddr`, `u16`, `String`. Если структура существует, каждое значение корректно; никакому другому коду не нужно разбирать и проверять их снова (проектирование через типы, Урок 08.5).
2. `from_lookup` принимает **функцию**, которая возвращает значение переменной. `from_env` передаёт функцию, читающую настоящее окружение. Тесты передают свою (раздел «Попробуйте сами»). Это внедрение зависимостей через замыкание (Урок 08.9).
3. `parse_or` использует значение по умолчанию, когда переменной нет, и возвращает `ConfigError::Invalid`, когда она есть, но не разбирается. Необязательные настройки получают разумные значения по умолчанию для разработки; у URL базы данных умолчания нет, потому что неправильная база по умолчанию хуже ошибки.
4. Мы пишем `Debug` вручную, а не выводим его автоматически, чтобы `{config:?}` печатал `<redacted>` вместо пароля.

> 🔒 **Безопасность:** в продакшене секреты обычно приходят из менеджера секретов или из файлов, которые подключает оркестратор («секреты» Docker и Kubernetes, Урок 10.8), а не из обычных переменных в файле в Git. Упражнение 2 добавляет чтение URL из файла.

## 2. Живость и готовность

Оркестратор (Docker, Kubernetes) следит за вашим сервисом с помощью **проверок работоспособности**. В Уроке 16.1 у сервиса был один `GET /health`; здесь мы делим его на два, потому что вопросов два, и если их смешать, случаются аварии:

| Проверка | Вопрос | Если она не проходит, оркестратор… | Наш эндпоинт |
|---|---|---|---|
| **Живость** | Жив ли процесс и может ли отвечать? | перезапускает контейнер | `GET /healthz` |
| **Готовность** | Может ли он делать настоящую работу прямо сейчас? | перестаёт посылать ему трафик, ждёт | `GET /readyz` |

Почему не одна проверка? Представьте, что база данных недоступна минуту. Если бы проверка живости проверяла базу, оркестратор перезапускал бы **каждую** копию вашего сервиса снова и снова, а базе это никак не помогает. Правильная реакция — готовность: «не готов, отправляйте трафик в другое место», а процесс остаётся работать и сам восстанавливается, когда база вернётся.

Теперь `src/main.rs`:

```rust
mod config;

use std::io::{Read, Write};
use std::net::{SocketAddr, TcpStream};
use std::process::ExitCode;
use std::time::Duration;

use axum::{Router, extract::State, http::StatusCode, routing::get};
use sqlx::PgPool;
use sqlx::postgres::PgPoolOptions;
use tokio::signal::unix::{SignalKind, signal};
use tracing::{error, info, warn};

use config::Config;

#[derive(Clone)]
struct AppState {
    db: PgPool,
}

#[tokio::main]
async fn main() -> ExitCode {
    tracing_subscriber::fmt().with_ansi(false).without_time().init();

    let config = match Config::from_env() {
        Ok(config) => config,
        Err(err) => {
            error!("invalid configuration: {err}");
            return ExitCode::FAILURE;
        }
    };

    if std::env::args().nth(1).as_deref() == Some("healthcheck") {
        return healthcheck(config.port);
    }

    info!(?config, "configuration loaded");
    match run(config).await {
        Ok(()) => ExitCode::SUCCESS,
        Err(err) => {
            error!("server failed: {err}");
            ExitCode::FAILURE
        }
    }
}

async fn run(config: Config) -> Result<(), Box<dyn std::error::Error>> {
    // `connect_lazy` пока не подключается: сервис запускается, даже если база недоступна.
    let db = PgPoolOptions::new()
        .acquire_timeout(Duration::from_secs(2))
        .connect_lazy(&config.database_url)?;

    let app = Router::new()
        .route("/healthz", get(healthz))
        .route("/readyz", get(readyz))
        .with_state(AppState { db });

    let listener = tokio::net::TcpListener::bind(config.listen_addr()).await?;
    info!("listening on {}", listener.local_addr()?);
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await?;
    info!("stopped cleanly");
    Ok(())
}

/// Живость: процесс работает и может отвечать по HTTP.
async fn healthz() -> &'static str {
    "ok"
}

/// Готовность: сервис может делать настоящую работу, значит, его база должна отвечать.
async fn readyz(State(state): State<AppState>) -> (StatusCode, &'static str) {
    match sqlx::query("SELECT 1").execute(&state.db).await {
        Ok(_) => (StatusCode::OK, "ready"),
        Err(err) => {
            warn!("not ready: {err}");
            (StatusCode::SERVICE_UNAVAILABLE, "database unavailable")
        }
    }
}

/// Ждёт Ctrl+C (SIGINT) или SIGTERM, который отправляет `docker stop`.
async fn shutdown_signal() {
    let mut terminate = signal(SignalKind::terminate()).expect("cannot listen for SIGTERM");
    tokio::select! {
        _ = tokio::signal::ctrl_c() => {}
        _ = terminate.recv() => {}
    }
    info!("shutdown signal received");
}

/// `p16-config-health healthcheck`: код возврата 0, если /healthz отвечает 200.
fn healthcheck(port: u16) -> ExitCode {
    let check = || -> std::io::Result<bool> {
        let addr = SocketAddr::from(([127, 0, 0, 1], port));
        let mut stream = TcpStream::connect_timeout(&addr, Duration::from_secs(2))?;
        stream.set_read_timeout(Some(Duration::from_secs(2)))?;
        stream.write_all(b"GET /healthz HTTP/1.1\r\nHost: localhost\r\nConnection: close\r\n\r\n")?;
        let mut response = String::new();
        stream.read_to_string(&mut response)?;
        Ok(response.starts_with("HTTP/1.1 200"))
    };
    match check() {
        Ok(true) => ExitCode::SUCCESS,
        _ => ExitCode::FAILURE,
    }
}
```

**Что происходит:**

1. `main` возвращает `ExitCode`, чтобы программа могла завершиться с кодом 1 при ошибке конфигурации. Конфигурация загружается **первой**: если она неверна, больше ничего не запускается.
2. `info!(?config, ...)` пишет конфигурацию в лог в форме `Debug`, которая скрывает URL.
3. `connect_lazy` создаёт пул **без подключения**. Соединения открываются при первом использовании. Поэтому процесс запускается и отвечает на `/healthz`, даже пока база ещё стартует, а `/readyz` говорит правду. `acquire_timeout(2 s)` заставляет проверку готовности упасть через 2 секунды, а не висеть стандартные 30.
4. `/readyz` выполняет `SELECT 1`, самый дешёвый запрос, и отвечает `503 Service Unavailable`, если база не отвечает.
5. `shutdown_signal` ждёт Ctrl+C или `SIGTERM`. Затем `with_graceful_shutdown` перестаёт принимать новые соединения, даёт текущим запросам закончиться, и `run` возвращается.
6. `healthcheck` — второй режим той же программы, он объясняется в разделе 4: крошечный HTTP-клиент (HTTP, Hypertext Transfer Protocol, протокол передачи гипертекста) на стандартной библиотеке (такой вы писали в Уроке 15.8), который превращает ответ в код возврата.

Попробуйте на Mac. Без `DATABASE_URL`:

```bash
cargo run
```

```text
ERROR p16_config_health: invalid configuration: environment variable DATABASE_URL is not set
```

С URL на порт, где (пока) не работает никакая база:

```bash
DATABASE_URL=postgres://app:secret@127.0.0.1:5999/app cargo run
```

```text
 INFO p16_config_health: configuration loaded config=Config { host: 127.0.0.1, port: 8080, database_url: "<redacted>" }
 INFO p16_config_health: listening on 127.0.0.1:8080
```

Во втором терминале:

```bash
curl -i http://127.0.0.1:8080/healthz
curl -i http://127.0.0.1:8080/readyz
```

```text
HTTP/1.1 200 OK
content-type: text/plain; charset=utf-8
content-length: 2
date: Wed, 18 Nov 2026 07:35:41 GMT

ok
HTTP/1.1 503 Service Unavailable
content-type: text/plain; charset=utf-8
content-length: 20
date: Wed, 18 Nov 2026 07:35:43 GMT

database unavailable
```

Жив, но не готов: ровно то, что нужно. Сервис записал в лог `WARN p16_config_health: not ready: pool timed out while waiting for an open connection`. Нажмите Ctrl+C; лог заканчивается так:

```text
 INFO p16_config_health: shutdown signal received
 INFO p16_config_health: stopped cleanly
```

## 3. Почему важен `SIGTERM`

`docker stop` (и Kubernetes, когда переносит контейнер) сначала отправляет `SIGTERM`. Процесс, который его игнорирует, работает до конца периода ожидания, 10 секунд в Docker, а затем получает `SIGKILL`: он умирает мгновенно, посреди запросов, не вытолкнув логи и телеметрию (Урок 16.12). Ctrl+C отправляет другой сигнал, `SIGINT`, поэтому программа, которая ждёт только `ctrl_c()`, работает на ноутбуке и жёстко умирает в продакшене.

Наш `shutdown_signal` слушает оба. С ним `docker compose stop app` в разделе 5 завершилась за 0,16 секунды, и две последние строки лога выше были записаны.

> ⚠️ **Частая ошибка:** держать в памяти работу, которая теряется при остановке, например сообщения, полученные из NATS, но ещё не обработанные. Корректное завершение должно сначала перестать брать новую работу, затем закончить или вернуть ту, что уже есть (для JetStream: просто не подтверждать, Урок 16.11).

## 4. Маленький и безопасный образ контейнера

Создайте `Dockerfile` рядом с `Cargo.toml`:

```dockerfile
# ---- этап 1: собрать программу ----
FROM rust:1.100 AS build
WORKDIR /src
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release --locked

# ---- этап 2: маленький образ, который только запускает её ----
# Образы "nonroot" работают от непривилегированного пользователя (uid 65532), а не от root.
FROM gcr.io/distroless/cc-debian13:nonroot
COPY --from=build /src/target/release/p16-config-health /app/p16-config-health
ENV APP_HOST=0.0.0.0 APP_PORT=8080
EXPOSE 8080
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
  CMD ["/app/p16-config-health", "healthcheck"]
ENTRYPOINT ["/app/p16-config-health"]
```

И `.dockerignore`, чтобы большая папка `target/` не отправлялась в Docker:

```text
target
```

**Что происходит:**

1. **Этап 1** использует официальный образ Rust для компиляции. `--locked` заставляет Cargo использовать ровно версии из `Cargo.lock` и падать, если файл устарел: образ собирается из тех же зависимостей, что вы тестировали.
2. **Этап 2** начинается с `gcr.io/distroless/cc-debian13:nonroot`, опубликованного проектом distroless от Google. В нём есть библиотеки C, нужные программе на Rust (`cc`), сертификаты и данные часовых поясов, и **нет командной оболочки, менеджера пакетов и `curl`**. Меньше программ в образе — меньше того, чем может воспользоваться злоумышленник. Тег `:nonroot` запускает программу от пользователя 65532, а не от root.
3. Из этапа 1 копируется только скомпилированный файл; компилятор и исходный код остаются позади. Итоговый образ здесь занял 59,2 МБ.
4. `APP_HOST=0.0.0.0` заставляет сервис слушать на всех интерфейсах **внутри** контейнера, чтобы Docker мог пробросить к нему порт. На Mac безопаснее значение по умолчанию `127.0.0.1`; в контейнере оно сделало бы сервис недоступным (см. «Частые ошибки»).
5. `HEALTHCHECK` говорит Docker запускать `/app/p16-config-health healthcheck` каждые 10 секунд. В образе нет `curl`, поэтому программа проверяет себя сама: код возврата `0` означает «здоров». После 3 неудач подряд контейнер помечается `unhealthy`. *Exec-форма* `["...", "..."]` запускает программу напрямую; простой строке понадобился бы `/bin/sh`, которого в этом образе нет.

> ⚡ **Производительность:** при каждом изменении в `src/` этот Dockerfile заново компилирует все зависимости. Упражнение 3 показывает, как закэшировать их в отдельном слое; одна тестовая пересборка сократилась с нескольких минут до 19 секунд.

## 5. Сервис и база данных в Compose

Создайте `compose.yaml`:

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev-only-password
      POSTGRES_DB: app
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10

  app:
    build: .
    environment:
      DATABASE_URL: postgres://app:dev-only-password@db:5432/app
    ports:
      - "127.0.0.1:8080:8080"
    depends_on:
      db:
        condition: service_healthy
```

**Что происходит:**

1. `db` запускает официальный образ PostgreSQL 18. Его собственный `healthcheck` использует `pg_isready` — инструмент внутри этого образа, который спрашивает сервер, принимает ли он соединения. (В этом образе есть оболочка, поэтому `CMD-SHELL` работает.)
2. `app` собирается из нашего `Dockerfile`. Внутри сети Compose имя хоста базы — просто `db`.
3. `depends_on` с `condition: service_healthy` заставляет Compose запустить `app` только **после** того, как `db` сообщит, что здоров. Простой `depends_on` ждёт только *запуска* контейнера, а не готовности PostgreSQL.
4. Опубликован только порт 8080 сервиса `app`, и только на `127.0.0.1`. База доступна только изнутри сети.

Соберите и запустите всё:

```bash
docker compose up -d --build
docker compose ps --format 'table {{.Service}}\t{{.Status}}'
```

```text
SERVICE   STATUS
app       Up Less than a second (health: starting)
db        Up 5 seconds (healthy)
```

`db` был здоров ещё до того, как `app` запустился. Через несколько секунд:

```text
SERVICE   STATUS
app       Up 15 seconds (healthy)
db        Up 20 seconds (healthy)
```

Теперь проверьте готовность, остановив и запустив базу:

```bash
curl -w ' %{http_code}\n' http://127.0.0.1:8080/readyz
docker compose stop db
curl -w ' %{http_code}\n' http://127.0.0.1:8080/healthz
curl -w ' %{http_code}\n' http://127.0.0.1:8080/readyz
docker compose start db
```

```text
ready 200
ok 200
database unavailable 503
```

Сервис остался жив (и Docker считал его `healthy`, потому что проверка использует `/healthz`), упала только готовность. Через несколько секунд после `docker compose start db` эндпоинт `/readyz` снова отвечает `ready 200`, без перезапуска. Остановите всё командой `docker compose down`.

> 🔒 **Безопасность:** `dev-only-password` в `compose.yaml` допустим для локальной базы разработки, недоступной снаружи. Никогда не коммитьте настоящие пароли. Для общих окружений кладите их в файл `.env`, указанный в `.gitignore`, или используйте секреты Compose (Урок 10.8).

## Частые ошибки

> ⚠️ **Частая ошибка:** слушать `127.0.0.1` внутри контейнера. Порт опубликован, но снаружи никто не отвечает: `127.0.0.1` внутри контейнера — это сам контейнер. Запуск образа с `-e APP_HOST=127.0.0.1` даёт:

```text
curl: (52) Empty reply from server
```

Исправление: в контейнерах слушайте `0.0.0.0` (наш `Dockerfile` это задаёт).

> ⚠️ **Частая ошибка:** проверка работоспособности в форме оболочки в distroless-образе, например `--health-cmd 'curl -f http://localhost:8080/healthz'`. Docker сообщает:

```text
exec: "/bin/sh": stat /bin/sh: no such file or directory
```

Исправление: используйте exec-форму с программой, которая есть в образе, как наш режим `healthcheck`.

> ⚠️ **Частая ошибка:** проверять базу данных в эндпоинте живости. Тогда короткий сбой базы перезапускает все контейнеры вашего сервиса по кругу. `/healthz` должен касаться только процесса.

> ⚠️ **Частая ошибка:** контейнер сразу завершается с кодом 1. Выполните `docker compose logs app` и прочитайте первую строку: ошибка конфигурации точно называет неверную переменную, например `invalid configuration: environment variable DATABASE_URL is not set`.

## Попробуйте сами

1. Добавьте модульные тесты ниже в конец `src/config.rs` и запустите `cargo test`. Они никогда не трогают настоящее окружение: `std::env::set_var` в редакции 2024 помечен `unsafe`, потому что изменение окружения, пока другие потоки его читают, — гонка данных, а тесты идут в параллельных потоках.

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::collections::HashMap;

    fn lookup(pairs: &[(&str, &str)]) -> impl Fn(&str) -> Option<String> {
        let map: HashMap<String, String> = pairs
            .iter()
            .map(|(k, v)| (k.to_string(), v.to_string()))
            .collect();
        move |key| map.get(key).cloned()
    }

    #[test]
    fn defaults_are_used_when_optional_values_are_missing() {
        let config = Config::from_lookup(lookup(&[("DATABASE_URL", "postgres://db/app")])).unwrap();
        assert_eq!(config.listen_addr().to_string(), "127.0.0.1:8080");
    }

    #[test]
    fn missing_database_url_is_an_error() {
        let err = Config::from_lookup(lookup(&[])).unwrap_err();
        assert_eq!(err, ConfigError::Missing("DATABASE_URL"));
    }

    #[test]
    fn debug_output_hides_the_database_url() {
        let config =
            Config::from_lookup(lookup(&[("DATABASE_URL", "postgres://u:secret@db/app")])).unwrap();
        assert!(!format!("{config:?}").contains("secret"));
    }
}
```

```text
test config::tests::defaults_are_used_when_optional_values_are_missing ... ok
test config::tests::missing_database_url_is_an_error ... ok
test config::tests::debug_output_hides_the_database_url ... ok
```

2. Выполните `APP_PORT=80a DATABASE_URL=x cargo run`. Вы получите `ERROR p16_config_health: invalid configuration: APP_PORT has an invalid value: "80a"`.
3. Пока сервис работает на Mac, выполните `DATABASE_URL=x cargo run -- healthcheck; echo $?`. Будет напечатано `0`. Остановите сервис и выполните снова: `1`.

## Упражнения

### Упражнение 1 — Тест для плохого порта (легко)

Добавьте тест `port_that_is_not_a_number_is_invalid`: при `APP_PORT=80a` результат должен быть `ConfigError::Invalid { key: "APP_PORT", value: "80a" }`.

<details>
<summary>Подсказка</summary>

Используйте помощник `lookup` с двумя парами: URL базы (чтобы мог упасть только порт) и плохой порт.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[test]
    fn port_that_is_not_a_number_is_invalid() {
        let err = Config::from_lookup(lookup(&[
            ("DATABASE_URL", "postgres://db/app"),
            ("APP_PORT", "80a"),
        ]))
        .unwrap_err();
        assert_eq!(
            err,
            ConfigError::Invalid {
                key: "APP_PORT",
                value: "80a".to_string()
            }
        );
    }
```

Почему это работает: `parse_or` находит переменную, `"80a".parse::<u16>()` не удаётся, и `map_err` строит вариант `Invalid` с ключом и исходным текстом. Поскольку `ConfigError` выводит `PartialEq`, тест сравнивает значения целиком.

</details>

### Упражнение 2 — Читать секрет из файла (средне)

Оркестраторы часто отдают секреты файлами, например `/run/secrets/database_url`. Поддержите `DATABASE_URL_FILE`: если переменная задана, читайте URL из этого файла (без перевода строки в конце); иначе используйте `DATABASE_URL`. Напишите тест, который записывает временный файл и проверяет результат.

<details>
<summary>Подсказка</summary>

`std::fs::read_to_string(&path)` читает файл; `.trim()` убирает перевод строки. Если файл нельзя прочитать, верните `ConfigError::Invalid` с ключом `DATABASE_URL_FILE`. В тесте постройте путь через `std::env::temp_dir().join(...)` и в конце удалите файл.

</details>

<details>
<summary>Ответ</summary>

В `from_lookup` замените строку с `database_url`:

```rust
        let database_url = match get("DATABASE_URL_FILE") {
            Some(path) => std::fs::read_to_string(&path)
                .map(|text| text.trim().to_string())
                .map_err(|_| ConfigError::Invalid {
                    key: "DATABASE_URL_FILE",
                    value: path,
                })?,
            None => get("DATABASE_URL").ok_or(ConfigError::Missing("DATABASE_URL"))?,
        };
```

Тест:

```rust
    #[test]
    fn database_url_can_come_from_a_file() {
        let path = std::env::temp_dir().join("p16-config-test-database-url");
        std::fs::write(&path, "postgres://u:p@db/app\n").unwrap();
        let config =
            Config::from_lookup(lookup(&[("DATABASE_URL_FILE", path.to_str().unwrap())])).unwrap();
        assert_eq!(config.database_url, "postgres://u:p@db/app");
        std::fs::remove_file(path).unwrap();
    }
```

Почему это работает: файл побеждает, когда заданы оба, поэтому секрету не нужно появляться в окружении процесса, где его могут показать инструменты вроде `docker inspect`. Редакторы и `echo` добавляют перевод строки в конец файла; `trim` его убирает, иначе соединение упало бы с непонятной ошибкой.

</details>

### Упражнение 3 — Закэшировать зависимости в Docker (сложно)

Измените этап сборки так, чтобы зависимости компилировались в отдельном слое, который Docker переиспользует, пока не меняются `Cargo.toml` и `Cargo.lock`. Соберите дважды с небольшим изменением в `src/main.rs` между сборками и сравните время.

<details>
<summary>Подсказка</summary>

Скопируйте только `Cargo.toml` и `Cargo.lock`, создайте фальшивый `src/main.rs` с `fn main() {}`, соберите и удалите его. Потом скопируйте настоящий `src`. Cargo решает, что пересобирать, по времени изменения файлов, поэтому перед второй сборкой выполните `touch` для настоящего `main.rs`.

</details>

<details>
<summary>Ответ</summary>

```dockerfile
# ---- этап 1: собрать программу ----
FROM rust:1.100 AS build
WORKDIR /src
COPY Cargo.toml Cargo.lock ./
# Собрать только зависимости с пустым main, чтобы этот слой закэшировался.
RUN mkdir src && echo 'fn main() {}' > src/main.rs \
    && cargo build --release --locked \
    && rm -rf src
COPY src ./src
# `touch` заставляет Cargo заметить, что main.rs новее закэшированной сборки.
RUN touch src/main.rs && cargo build --release --locked
```

Второй этап не меняется. В одном тесте на машине курса первая сборка заняла 120 секунд, а сборка после изменения в `src/main.rs` — 19 секунд; с исходным Dockerfile та же пересборка компилировала всё заново и заняла несколько минут.

Почему это работает: Docker кэширует результат каждой инструкции и переиспользует его, пока не изменились инструкция и её входные файлы. Дорогой `cargo build` всех зависимостей теперь зависит только от `Cargo.toml` и `Cargo.lock`. Инструмент `cargo-chef` автоматизирует этот приём для рабочих пространств со многими крейтами.

</details>

## Вопросы для самопроверки

1. Почему у URL базы данных нет значения по умолчанию, а у порта есть?

<details>
<summary>Ответ</summary>

Порт по умолчанию безвреден для разработки. База по умолчанию могла бы незаметно подключить сервис не к той базе; понятная ошибка при запуске безопаснее.

</details>

2. База недоступна две минуты. Что должны отвечать `/healthz` и `/readyz` и почему?

<details>
<summary>Ответ</summary>

`/healthz`: 200, процесс жив, значит, его нельзя перезапускать. `/readyz`: 503, чтобы ему не посылали трафик, пока база снова не ответит.

</details>

3. Почему сервис слушает `SIGTERM`, а не только Ctrl+C?

<details>
<summary>Ответ</summary>

`docker stop` и Kubernetes отправляют `SIGTERM`; Ctrl+C отправляет `SIGINT`. Без обработки `SIGTERM` сервис убивают через `SIGKILL` после периода ожидания, посреди запросов и без выталкивания логов и телеметрии.

</details>

4. Почему `HEALTHCHECK` в нашем образе не может использовать `curl`?

<details>
<summary>Ответ</summary>

В distroless-образе нет ни `curl`, ни командной оболочки. Проверка должна запускать программу, которая есть в образе; здесь — сам исполняемый файл сервиса в режиме `healthcheck`.

</details>

5. Чем `depends_on: [db]` отличается от `depends_on` с `condition: service_healthy`?

<details>
<summary>Ответ</summary>

Первое ждёт только запуска контейнера `db`. Второе ждёт, пока пройдёт его проверка работоспособности, то есть пока PostgreSQL действительно начнёт принимать соединения.

</details>

## Итоги

- Читайте конфигурацию один раз при запуске в типизированную структуру; отказывайте быстро с сообщением, в котором названа переменная; давайте значения по умолчанию только там, где это безопасно.
- Скрывайте секреты в выводе `Debug` и в логах; в продакшене предпочитайте файлы секретов обычным переменным.
- Живость (`/healthz`) спрашивает «жив ли процесс?», готовность (`/readyz`) — «может ли он работать сейчас?». Никогда не проверяйте зависимости в проверке живости.
- `connect_lazy` позволяет сервису стартовать раньше базы; короткий `acquire_timeout` делает проверки готовности быстрыми.
- Обрабатывайте `SIGTERM` для корректного завершения в контейнерах.
- Многоэтапная сборка в distroless-образ `nonroot` даёт маленький образ без оболочки; программа может сама быть своей проверкой работоспособности.
- `condition: service_healthy` в Compose запускает сервисы в правильном порядке.

## Источники и что почитать

- [The Twelve-Factor App — III. Config](https://12factor.net/config) — почему конфигурация живёт в окружении.
- [Dockerfile reference](https://docs.docker.com/reference/dockerfile/) — раздел HEALTHCHECK перечисляет все параметры проверок работоспособности; там же объясняется exec-форма и форма оболочки.
- [Docker Compose — Control startup order](https://docs.docker.com/compose/how-tos/startup-order/) — `depends_on` с `service_healthy`.
- [Kubernetes — Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/concepts/configuration/liveness-readiness-startup-probes/) — те же две идеи в Kubernetes (Этап 21).
- [GoogleContainerTools/distroless на GitHub](https://github.com/GoogleContainerTools/distroless) — что содержат distroless-образы и какие теги существуют.
- [sqlx — PgPoolOptions на docs.rs](https://docs.rs/sqlx/latest/sqlx/postgres/type.PgPoolOptions.html) — `connect_lazy`, `acquire_timeout` и другие настройки пула.

---

[🏠 Начало](../START-HERE.md) · [← 16.12 Логи, метрики и трассировки с OpenTelemetry](12-logs-metrics-and-traces.md) · [📋 Этап 16](README.md) · [16.14 Тестирование бэкенд-сервиса →](14-testing-a-backend-service.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
