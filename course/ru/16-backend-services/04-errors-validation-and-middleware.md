<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 16.3 Веб-сервисы на axum](03-web-services-with-axum.md) · [📋 Этап 16](README.md) · [16.5 Основы SQL и PostgreSQL →](05-sql-and-postgresql-basics.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/16-backend-services/04-errors-validation-and-middleware.md) <!-- nav -->

# Урок 16.4 — Ошибки, проверка ввода и middleware

> **Этап 16 — Бэкенд-сервисы** · Урок 4 из 16 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как собрать все сбои API (Application Programming Interface, программный интерфейс) в один тип ошибки, который сам превращается в JSON-тело (JavaScript Object Notation, нотация объектов JavaScript) ошибки из контракта.
- Как проверять ввод один раз, на границе, с помощью типа, который может содержать только допустимые значения.
- Как заставить собственные отказы axum использовать ваш формат ошибок.
- Что такое middleware (промежуточный слой), как слои оборачивают сервис, словно слои луковицы, и как пользоваться
  готовыми слоями `tower-http` (логи запросов, тайм-ауты, ограничение размера тела).
- Как написать собственный маленький middleware через `axum::middleware::from_fn`.

## Перед началом

- Вы прошли [Урок 16.3 — Веб-сервисы на axum](03-web-services-with-axum.md), и у вас есть его `code/p16-tasks`.
- Вы знаете `thiserror` ([Урок 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md)),
  «разбирайте, а не проверяйте» ([Урок 10.4 — Проверка ввода](../10-security-fundamentals/04-input-validation.md))
  и `tracing` ([Урок 09.10 — Логирование и трассировка](../09-testing-debugging-benchmarking/10-logging-and-tracing.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **тип ошибки** (error type) | Одно перечисление, в котором перечислены все способы, которыми может не удаться операция. Здесь: `ApiError`. |
| **проверка ввода** (validation) | Проверка того, что ввод соблюдает правила (не пустой, не слишком длинный), до его использования. |
| **newtype** | Структура с одним полем, которая оборачивает другой тип, чтобы придать ему смысл и правила, например `Title(String)`. |
| **middleware** (промежуточный слой) | Код, который выполняется вокруг **каждого** запроса (или группы запросов), до и после обработчика: логирование, тайм-ауты, ограничения, аутентификация. |
| **слой** (layer) | В tower — значение, которое оборачивает сервис в middleware. `.layer(x)` у `Router` добавляет один такой слой. |
| **тайм-аут** (timeout) | Ограничение по времени: если работа не закончена вовремя, её останавливают и возвращают ошибку. |
| **идентификатор запроса** (request ID) | Уникальное число или текст, присвоенный каждому запросу, чтобы потом можно было сопоставить его строки в логах и его ответ. |

## 1. Один тип ошибки для всего API

В Уроке 16.3 ошибки были голыми кодами статуса, а отказы axum — простым текстом. Контракт из Урока 16.2
требует, чтобы каждая ошибка выглядела как `{"code": "...", "message": "..."}`. Чистый путь к этому: одно
перечисление для всех сбоев и одно место, которое превращает его в ответ.

Добавьте `thiserror` и фичу `macros` у axum (она нужна в разделе 3), затем остальные крейты этого урока:

```bash
cd ~/projects/rust/code/p16-tasks
cargo add axum@0.8 --features macros
cargo add thiserror@2
cargo add tower-http@0.7 --features trace,timeout,limit
cargo add tracing@0.1
cargo add tracing-subscriber@0.3 --features env-filter
```

Создайте `src/error.rs`:

```rust
use axum::{
    Json,
    extract::{FromRequest, rejection::JsonRejection},
    http::StatusCode,
    response::{IntoResponse, Response},
};
use serde::Serialize;

/// Все способы, которыми может не удаться запрос к API задач.
#[derive(Debug, thiserror::Error)]
pub enum ApiError {
    #[error("task {0} not found")]
    NotFound(u64),
    #[error("{0}")]
    Validation(String),
    #[error("{message}")]
    Rejected { status: StatusCode, message: String },
}

impl ApiError {
    fn status(&self) -> StatusCode {
        match self {
            ApiError::NotFound(_) => StatusCode::NOT_FOUND,
            ApiError::Validation(_) => StatusCode::UNPROCESSABLE_ENTITY,
            ApiError::Rejected { status, .. } => *status,
        }
    }

    fn code(&self) -> &'static str {
        match self {
            ApiError::NotFound(_) => "not_found",
            ApiError::Validation(_) => "validation_failed",
            ApiError::Rejected { status, .. } => match *status {
                StatusCode::UNPROCESSABLE_ENTITY => "validation_failed",
                StatusCode::UNSUPPORTED_MEDIA_TYPE => "unsupported_media_type",
                _ => "bad_request",
            },
        }
    }
}

#[derive(Serialize)]
struct ErrorBody {
    code: &'static str,
    message: String,
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let body = ErrorBody {
            code: self.code(),
            message: self.to_string(),
        };
        (self.status(), Json(body)).into_response()
    }
}

impl From<JsonRejection> for ApiError {
    fn from(rejection: JsonRejection) -> Self {
        ApiError::Rejected {
            status: rejection.status(),
            message: rejection.body_text(),
        }
    }
}

/// Как `axum::Json`, но плохое тело становится `ApiError` (нашим JSON-форматом ошибок).
#[derive(FromRequest)]
#[from_request(via(axum::Json), rejection(ApiError))]
pub struct AppJson<T>(pub T);
```

**Что происходит:**

1. У `ApiError` три варианта: отсутствующая задача, нарушенное правило (`Validation`) и запрос, который сам axum
   не смог прочитать (`Rejected`, хранит код статуса и сообщение axum).
2. `#[error("...")]` из `thiserror` задаёт текст `Display` для каждого варианта, так что `self.to_string()` —
   сообщение для человека. `{0}` — первое поле, `{message}` — именованное поле.
3. `status()` и `code()` сопоставляют каждому варианту HTTP-статус (HyperText Transfer Protocol, протокол передачи гипертекста) и стабильный код из контракта.
4. `impl IntoResponse for ApiError` — главная часть: теперь обработчик может возвращать `Result<_, ApiError>`, и
   каждый `Err` становится JSON-телом ошибки с правильным статусом. Сопоставление живёт в **одном** месте.
5. Последняя часть (`From<JsonRejection>` и `AppJson`) — это раздел 3.

## 2. Проверка на границе: тип `Title`

Контракт говорит: у заголовка обрезаются пробелы, и в нём от 1 до 200 символов. Где должна жить эта проверка?
Не «где-то в каждом обработчике»: в одном из них вы её забудете. Урок 10.4 научил способу получше —
*разбирайте, а не проверяйте*: сделайте тип, единственный конструктор которого проверяет правила. Если у вас есть
`Title`, он корректен.

Создайте `src/title.rs`:

```rust
use crate::error::ApiError;

pub const MAX_TITLE_CHARS: usize = 200;

/// Заголовок задачи, соблюдающий правила контракта. Получить его можно только через `parse`.
#[derive(Debug, Clone, PartialEq)]
pub struct Title(String);

impl Title {
    pub fn parse(text: &str) -> Result<Title, ApiError> {
        let text = text.trim();
        if text.is_empty() {
            return Err(ApiError::Validation("title must not be empty".to_string()));
        }
        let chars = text.chars().count();
        if chars > MAX_TITLE_CHARS {
            return Err(ApiError::Validation(format!(
                "title must have at most {MAX_TITLE_CHARS} characters, got {chars}"
            )));
        }
        if text.chars().any(char::is_control) {
            return Err(ApiError::Validation(
                "title must not contain control characters".to_string(),
            ));
        }
        Ok(Title(text.to_string()))
    }

    pub fn into_string(self) -> String {
        self.0
    }
}
```

**Что происходит:**

1. Поле `Title` приватное, поэтому код вне этого модуля не может написать `Title(String::new())`. Единственная
   дверь — `parse`.
2. `trim()` убирает пробелы с обоих концов, поэтому `"   "` становится пустым и отклоняется.
3. `chars().count()` считает **символы**, а не байты: `"я"` — один символ, но два байта в UTF-8 (Урок 12.5).
   Ограничение в байтах удивило бы пользователей других алфавитов.
4. Управляющие символы (переводы строк, табуляции, «звонок» терминала `\u0007`) отклоняются: заголовок — одна
   строка текста, а такие символы могут сломать логи и экраны.
5. `into_string` возвращает внутренний `String`, когда мы сохраняем задачу.

> 🔒 **Безопасность:** каждое ограничение — ещё и защита. Без максимальной длины один клиент мог бы сохранить
> «заголовок» в 100 мегабайт. Проверяйте размер, символы и формат **каждого** поля, которое приходит снаружи.

## 3. Наши собственные отказы: `AppJson`

`Title::parse` выполняется только после того, как JSON прочитан. Когда сломан сам JSON, экстрактор `Json`
отклоняет запрос простым текстом ещё до того, как выполнится наш код. Чтобы и там был наш формат, мы оборачиваем
`Json` в собственный экстрактор `AppJson` — последние три строки `error.rs`.

**Что происходит:**

1. `#[derive(FromRequest)]` (из фичи `macros`) делает `AppJson<T>` экстрактором.
2. `via(axum::Json)` говорит: «делай работу обычным экстрактором `Json`».
3. `rejection(ApiError)` говорит: «если не получилось, преврати его отказ в `ApiError`». Превращение — это
   `impl From<JsonRejection> for ApiError` выше; он сохраняет статус axum (`400`, `415`, `422`) и его сообщение.

Теперь обработчики пишут `AppJson(new_task): AppJson<NewTask>` вместо `Json(...)`. Для ответов мы по-прежнему
используем `axum::Json`.

## 4. Обработчики с ошибками и проверкой ввода

Измените начало `src/main.rs`, чтобы объявить два модуля и импортировать то, что нужно этому уроку:

```rust
mod error;
mod title;

use std::collections::BTreeMap;
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::{Arc, Mutex};
use std::time::Duration;

use axum::{
    Json, Router,
    extract::{Path, Query, Request, State},
    http::{HeaderValue, StatusCode, header},
    middleware::{self, Next},
    response::{IntoResponse, Response},
    routing::get,
};
use serde::{Deserialize, Serialize};
use tower_http::{limit::RequestBodyLimitLayer, timeout::TimeoutLayer, trace::TraceLayer};

use error::{ApiError, AppJson};
use title::Title;
```

Структуры (`Task`, `NewTask`, `TaskChanges`, `ListFilter`, `Store`), `AppState`, `list_tasks` и `health`
остаются как в Уроке 16.3. Замените остальные четыре обработчика:

```rust
async fn create_task(
    State(state): State<AppState>,
    AppJson(new_task): AppJson<NewTask>,
) -> Result<impl IntoResponse, ApiError> {
    let title = Title::parse(&new_task.title)?;
    let mut store = state.lock().unwrap();
    store.last_id += 1;
    let task = Task {
        id: store.last_id,
        title: title.into_string(),
        done: false,
    };
    store.tasks.insert(task.id, task.clone());
    let location = format!("/tasks/{}", task.id);
    Ok((
        StatusCode::CREATED,
        [(header::LOCATION, location)],
        Json(task),
    ))
}

async fn get_task(
    State(state): State<AppState>,
    Path(id): Path<u64>,
) -> Result<Json<Task>, ApiError> {
    let store = state.lock().unwrap();
    match store.tasks.get(&id) {
        Some(task) => Ok(Json(task.clone())),
        None => Err(ApiError::NotFound(id)),
    }
}

async fn update_task(
    State(state): State<AppState>,
    Path(id): Path<u64>,
    AppJson(changes): AppJson<TaskChanges>,
) -> Result<Json<Task>, ApiError> {
    // Проверяем ввод до того, как взять блокировку.
    let title = changes.title.as_deref().map(Title::parse).transpose()?;
    let mut store = state.lock().unwrap();
    let task = store.tasks.get_mut(&id).ok_or(ApiError::NotFound(id))?;
    if let Some(title) = title {
        task.title = title.into_string();
    }
    if let Some(done) = changes.done {
        task.done = done;
    }
    Ok(Json(task.clone()))
}

async fn delete_task(
    State(state): State<AppState>,
    Path(id): Path<u64>,
) -> Result<StatusCode, ApiError> {
    let mut store = state.lock().unwrap();
    match store.tasks.remove(&id) {
        Some(_) => Ok(StatusCode::NO_CONTENT),
        None => Err(ApiError::NotFound(id)),
    }
}
```

**Что происходит:**

1. Каждый обработчик, который может завершиться неудачей, возвращает `Result<_, ApiError>`. Оператор `?`
   превращает любой `ApiError` в JSON-ответ с ошибкой.
2. `create_task` **сначала** разбирает заголовок. Если он некорректен, функция возвращается раньше, чем тронет
   хранилище: ни задачи, ни израсходованного идентификатора.
3. В `update_task` заголовок необязателен. `changes.title.as_deref()` превращает `Option<String>` в
   `Option<&str>`; `.map(Title::parse)` даёт `Option<Result<Title, ApiError>>`; `.transpose()` переворачивает это в
   `Result<Option<Title>, ApiError>`, так что `?` может вернуть ошибку. Отсутствующий заголовок остаётся `None`.
4. `create_task` возвращает `Result<impl IntoResponse, ApiError>`: тип успеха — по-прежнему кортеж из Урока 16.3.

> 💡 **Совет:** проверяйте весь ввод до того, как менять какое-либо состояние. Тогда запрос либо полностью
> выполняется, либо ничего не меняет, а об этом гораздо проще рассуждать. Урок 16.7 даёт ту же гарантию для базы
> данных с помощью транзакций.

## 5. Middleware: код вокруг каждого запроса

Некоторые задачи относятся к **каждому** запросу, а не к одному обработчику: написать строку лога, остановить
слишком долгие запросы, отклонить огромные тела, проверить вход. Копировать это в каждый обработчик было бы
однообразно, и это легко забыть. **Middleware** делает это один раз, вокруг всего маршрутизатора.

В tower middleware добавляется как **слой**. Каждый слой оборачивает всё, что внутри него, словно слои луковицы.
Запрос идёт внутрь через каждый слой к обработчику; ответ возвращается наружу:

```text
 request ─▶ ┌─ request_id ───────────────────────────────────────┐
            │  ┌─ TraceLayer (logs) ──────────────────────────┐  │
            │  │  ┌─ TimeoutLayer (10 s) ──────────────────┐  │  │
            │  │  │  ┌─ RequestBodyLimitLayer (16 KiB) ──┐ │  │  │
            │  │  │  │   router ─▶ handler               │ │  │  │
            │  │  │  └───────────────────────────────────┘ │  │  │
            │  │  └────────────────────────────────────────┘  │  │
            │  └──────────────────────────────────────────────┘  │
 response ◀─└────────────────────────────────────────────────────┘
```

Три готовых слоя берутся из `tower-http` (по одной фиче Cargo на каждый):

- **`TraceLayer`** создаёт спан `tracing` для каждого запроса и пишет в лог, когда запрос начался и когда ответ
  готов, со статусом и задержкой.
- **`TimeoutLayer`** даёт каждому запросу ограничение по времени. Если обработчик не закончил за 10 секунд, слой
  отбрасывает его (Урок 14.14: отбросить future — значит отменить его) и отвечает выбранным вами статусом. Мы
  выбираем `503 Service Unavailable`: «сервер не смог ответить вовремя».
- **`RequestBodyLimitLayer`** отклоняет тела больше предела с `413 Payload Too Large`, чтобы никто не мог
  заставить сервис прочитать в память гигабайт.

Четвёртый — наш собственный. `axum::middleware::from_fn` превращает обычную асинхронную функцию в middleware.
Она получает запрос и `next` (всё, что внутри неё); она может работать до вызова `next.run(request)` и после
него. Добавьте её в `src/main.rs` вместе с новыми `app` и `main`:

```rust
static NEXT_REQUEST_ID: AtomicU64 = AtomicU64::new(1);

/// Наш собственный middleware: добавляет каждому ответу заголовок `x-request-id`.
async fn request_id(request: Request, next: Next) -> Response {
    let id = NEXT_REQUEST_ID.fetch_add(1, Ordering::Relaxed);
    let mut response = next.run(request).await;
    response
        .headers_mut()
        .insert("x-request-id", HeaderValue::from(id));
    response
}

fn app(state: AppState) -> Router {
    Router::new()
        .route("/health", get(health))
        .route("/tasks", get(list_tasks).post(create_task))
        .route(
            "/tasks/{id}",
            get(get_task).patch(update_task).delete(delete_task),
        )
        .with_state(state)
        // Последний .layer() — самый внешний: запросы проходят через него первым.
        .layer(RequestBodyLimitLayer::new(16 * 1024))
        .layer(TimeoutLayer::with_status_code(
            StatusCode::SERVICE_UNAVAILABLE,
            Duration::from_secs(10),
        ))
        .layer(TraceLayer::new_for_http())
        .layer(middleware::from_fn(request_id))
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "p16_tasks=debug,tower_http=debug".into()),
        )
        .init();

    let state = AppState::default();
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::info!("listening on http://127.0.0.1:3000");
    axum::serve(listener, app(state)).await.unwrap();
}
```

**Что происходит:**

1. `request_id` берёт следующий номер, даёт остальному сервису построить ответ (`next.run(request).await`), затем
   добавляет заголовок. Код до `next.run` видит запрос; код после него видит ответ.
2. В `app` каждый `.layer(...)` оборачивает всё, что написано **до** него. Поэтому последний, `request_id`, —
   внешний слой луковицы, а ограничение тела — самый внутренний. Чтобы проследить путь запроса, читайте цепочку
   снизу вверх.
3. `main` настраивает `tracing`, как в Уроке 09.10. Без переменной `RUST_LOG` он показывает события уровня
   `debug` нашего крейта и `tower_http` (куда пишет `TraceLayer`). `tracing::info!` заменяет `println!`.

> ⚡ **Производительность:** middleware выполняется для каждого запроса, поэтому делайте его дешёвым. Атомарный
> счётчик и один заголовок почти ничего не стоят; запрос к базе данных в middleware стоил бы по запросу на каждый
> запрос.

> 📏 **Соглашение:** `x-request-id` — распространённое имя для такого заголовка; в tower-http даже есть готовые
> слои для него (`SetRequestIdLayer`, `PropagateRequestIdLayer`).

## 6. Запуск

`cargo run`, затем во втором терминале создайте задачу с пробелами вокруг заголовка:

```bash
curl -i -H 'content-type: application/json' -d '{"title":"  Buy milk  "}' http://127.0.0.1:3000/tasks
```

```text
HTTP/1.1 201 Created
content-type: application/json
location: /tasks/1
x-request-id: 1
content-length: 40
date: Wed, 18 Nov 2026 07:16:05 GMT

{"id":1,"title":"Buy milk","done":false}
```

Пробелы обрезаны, а в ответе есть `x-request-id`. Терминал сервиса показывает лог `TraceLayer` (в вашем
терминале — в цвете):

```text
2026-11-18T07:16:05.006327Z  INFO p16_tasks: listening on http://127.0.0.1:3000
2026-11-18T07:16:05.174757Z DEBUG request{method=POST uri=/tasks version=HTTP/1.1}: tower_http::trace::on_request: started processing request
2026-11-18T07:16:05.174895Z DEBUG request{method=POST uri=/tasks version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=0 ms status=201
2026-11-18T07:16:05.174936Z DEBUG request{method=POST uri=/tasks version=HTTP/1.1}: tower_http::trace::on_eos: end of stream stream_duration=0 ms
```

Теперь плохие запросы из Урока 16.3, раздел 7, и несколько новых:

```bash
curl -s -H 'content-type: application/json' -d '{"title":"   "}' http://127.0.0.1:3000/tasks; echo
curl -s -H 'content-type: application/json' -d '{"title":"a\u0007b"}' http://127.0.0.1:3000/tasks; echo
curl -s -H 'content-type: application/json' -d '{"title":' http://127.0.0.1:3000/tasks; echo
curl -s -d '{"title":"x"}' http://127.0.0.1:3000/tasks; echo
curl -s -H 'content-type: application/json' -d '{"name":"x"}' http://127.0.0.1:3000/tasks; echo
curl -s http://127.0.0.1:3000/tasks/7; echo
curl -s -X PATCH -H 'content-type: application/json' -d '{"title":""}' http://127.0.0.1:3000/tasks/1; echo
```

```text
{"code":"validation_failed","message":"title must not be empty"}
{"code":"validation_failed","message":"title must not contain control characters"}
{"code":"bad_request","message":"Failed to parse the request body as JSON: title: EOF while parsing a value at line 1 column 9"}
{"code":"unsupported_media_type","message":"Expected request with `Content-Type: application/json`"}
{"code":"validation_failed","message":"Failed to deserialize the JSON body into the target type: missing field `title` at line 1 column 12"}
{"code":"not_found","message":"task 7 not found"}
{"code":"validation_failed","message":"title must not be empty"}
```

Теперь у каждой ошибки форма из контракта. (`\u0007` внутри JSON — управляющий символ «звонок».) Наконец,
отправьте тело примерно в 20 000 байт, больше предела в 16 КиБ (кибибайт, 1024 байта):

```bash
python3 -c "print('{\"title\":\"' + 'x'*20000 + '\"}')" > big.json
curl -i -H 'content-type: application/json' --data @big.json http://127.0.0.1:3000/tasks
```

```text
HTTP/1.1 413 Payload Too Large
content-type: text/plain; charset=utf-8
x-request-id: 10
content-length: 21
date: Wed, 18 Nov 2026 07:16:05 GMT

length limit exceeded
```

**Что происходит:**

1. `python3` (установлен на каждом Mac) пишет файл с заголовком в 20 000 символов. `--data @big.json` отправляет
   файл как тело.
2. `RequestBodyLimitLayer` видит, что `content-length` больше 16 КиБ, и отвечает `413` **до того**, как запрос
   дойдёт до маршрутизатора; тело даже не читается. Поскольку этот ответ приходит от слоя, а не от нашего кода, он
   простым текстом. Для такого редкого случая это допустимо; `x-request-id` всё равно на месте, потому что
   `request_id` — самый внешний слой.

Остановите сервис и удалите `big.json`.

## 7. Тесты ошибок

Тесты из Урока 16.3 по-прежнему проходят. Пусть `unknown_task_is_not_found` проверяет ещё и тело, и добавьте тест
ошибки проверки ввода:

```rust
    #[tokio::test]
    async fn unknown_task_is_not_found() {
        let app = app(AppState::default());
        let request = Request::get("/tasks/42").body(Body::empty()).unwrap();
        let (status, body) = send(app, request).await;
        assert_eq!(status, StatusCode::NOT_FOUND);
        assert_eq!(body, r#"{"code":"not_found","message":"task 42 not found"}"#);
    }

    #[tokio::test]
    async fn empty_title_is_rejected() {
        let app = app(AppState::default());
        let (status, body) = send(app, post_json("/tasks", r#"{"title":"   "}"#)).await;
        assert_eq!(status, StatusCode::UNPROCESSABLE_ENTITY);
        assert_eq!(
            body,
            r#"{"code":"validation_failed","message":"title must not be empty"}"#
        );
    }
```

```text
running 3 tests
test tests::unknown_task_is_not_found ... ok
test tests::empty_title_is_rejected ... ok
test tests::created_task_can_be_read_back ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Слои — часть `app()`, поэтому тесты с `oneshot` тоже проходят через них.

Число тестов в выводах этого этапа учитывает только тесты из текста урока, если урок не говорит иного. Если вы
сохранили тесты из прошлых упражнений (здесь — три теста Урока 16.3), у вас их больше, и это нормально.

> 🧪 **Тестирование:** проверяйте **неудачные пути** так же тщательно, как удачные. Клиенты зависят от точных кодов
> ошибок; тест ошибки защищает их так же, как тест контракта из Урока 16.2.

## Частые ошибки

**1. `AppJson` без преобразования `From`.** Если нет `impl From<JsonRejection> for ApiError`, `AppJson` вообще не
экстрактор. С `#[axum::debug_handler]` на обработчике компилятор говорит об этом:

```text
error[E0277]: the trait bound `AppJson<NewTask>: FromRequest<Arc<std::sync::Mutex<Store>>, _>` is not satisfied
  --> src/main.rs:71:24
   |
71 |     AppJson(new_task): AppJson<NewTask>,
   |                        ^^^^^^^ unsatisfied trait bound
   = note: Function argument is not a valid axum extractor.
```

Добавьте реализацию `From`: атрибуту `rejection(ApiError)` она нужна.

**2. Изменение состояния до проверки.** Если бы `create_task` увеличивал `last_id` и только потом разбирал
заголовок, отклонённый запрос всё равно израсходовал бы идентификатор. Сначала проверяйте, затем блокируйте и
меняйте.

**3. Внутренние подробности в сообщениях об ошибках.** Когда в Уроке 16.6 появится база данных, захочется
написать `ApiError::Validation(db_error.to_string())`. Это отправит подробности SQL (Structured Query Language, язык структурированных запросов) посторонним. Для неожиданных
сбоев нужен свой вариант, который отвечает только `{"code":"internal","message":"internal error"}` и пишет
подробности в лог.

**4. Слои в неправильном порядке.** Если бы `TraceLayer` был внутри `TimeoutLayer`, запрос, остановленный
тайм-аутом, не оставил бы строки «finished» в логе. Внешние слои видят всё, что делают внутренние; логирование и
идентификаторы запросов ставьте снаружи.

## Попробуйте сами

1. Запустите сервис через `RUST_LOG=tower_http=info cargo run`. Логи запросов пропадают: `TraceLayer` пишет на
   уровне `debug`.
2. Отправьте заголовок ровно из 200 символов (`'x'*200` в команде Python) и из 201. Первый создаётся, второй
   получает `title must have at most 200 characters, got 201`.
3. Сделайте коммит: `git add code/p16-tasks && git commit -m "p16-tasks: JSON errors, validation, middleware"`.

## Упражнения

### Упражнение 1 — Модульные тесты для `Title` (легко)

Добавьте `#[cfg(test)] mod tests` в конец `src/title.rs`, который проверяет: пробелы обрезаются; `""` и `" \t "`
отклоняются; 200 кириллических букв `я` (400 байт) принимаются, а 201 `x` отклоняется; заголовок с `\n`
отклоняется.

<details>
<summary>Подсказка</summary>

`"я".repeat(MAX_TITLE_CHARS)` строит длинную строку. Для отказов достаточно `assert!(Title::parse(...).is_err())`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn trims_spaces() {
        assert_eq!(Title::parse("  Buy milk ").unwrap().into_string(), "Buy milk");
    }

    #[test]
    fn rejects_empty_and_blank() {
        assert!(Title::parse("").is_err());
        assert!(Title::parse(" \t ").is_err());
    }

    #[test]
    fn length_limit_counts_characters_not_bytes() {
        let ok = "я".repeat(MAX_TITLE_CHARS); // 200 символов, 400 байт
        assert!(Title::parse(&ok).is_ok());
        let too_long = "x".repeat(MAX_TITLE_CHARS + 1);
        assert!(Title::parse(&too_long).is_err());
    }

    #[test]
    fn rejects_control_characters() {
        assert!(Title::parse("a\nb").is_err());
    }
}
```

```text
test title::tests::rejects_empty_and_blank ... ok
test title::tests::rejects_control_characters ... ok
test title::tests::trims_spaces ... ok
test title::tests::length_limit_counts_characters_not_bytes ... ok
```

Почему это работает: `Title` — маленький «чистый» тип, поэтому модульные тесты быстрые и не требуют
маршрутизатора. Тест с кириллицей доказывает, что предел считает символы: версия с `text.len()` (байты) его
отклонила бы.

</details>

### Упражнение 2 — Заголовок `server` у каждого ответа (средне)

Напишите второй middleware `server_header`, который добавляет `server: p16-tasks` к каждому ответу, и добавьте его в
`app` как самый внешний слой. Напишите тест, который отправляет `GET /no-such-path` через `oneshot` и проверяет,
что даже у этого ответа `404` есть заголовок.

<details>
<summary>Подсказка</summary>

`header::SERVER` — имя заголовка, `HeaderValue::from_static("p16-tasks")` — значение. В тесте сохраните весь ответ:
`let response = app.oneshot(request).await.unwrap();` и прочитайте `response.headers()["server"]`.

</details>

<details>
<summary>Ответ</summary>

```rust
async fn server_header(request: Request, next: Next) -> Response {
    let mut response = next.run(request).await;
    response
        .headers_mut()
        .insert(header::SERVER, HeaderValue::from_static("p16-tasks"));
    response
}

// в app(), после слоя request_id:
//         .layer(middleware::from_fn(server_header))

    #[tokio::test]
    async fn every_response_names_the_server() {
        let app = app(AppState::default());
        let request = Request::get("/no-such-path").body(Body::empty()).unwrap();
        let response = app.oneshot(request).await.unwrap();
        assert_eq!(response.status(), StatusCode::NOT_FOUND);
        assert_eq!(response.headers()["server"], "p16-tasks");
    }
```

Почему это работает: слои оборачивают **весь** маршрутизатор, включая запасной обработчик, который отвечает `404`
на неизвестные пути, поэтому заголовок появляется у каждого ответа. Решение на уровне обработчиков пропустило бы
именно эти случаи.

</details>

### Упражнение 3 — JSON-ошибки для плохих путей (сложно)

`GET /tasks/abc` всё ещё отвечает простым текстом, потому что у `Path` свой отказ. Создайте `AppPath<T>` в
`error.rs` так же, как `AppJson`, используйте его в трёх обработчиках с `{id}` и напишите тест, что
`GET /tasks/abc` возвращает `400` с
``{"code":"bad_request","message":"Invalid URL: Cannot parse `abc` to a `u64`"}``.

<details>
<summary>Подсказка</summary>

`Path` читает только *части* запроса (не тело), поэтому derive — `FromRequestParts`, а тип отказа —
`axum::extract::rejection::PathRejection`. У него тоже есть `status()` и `body_text()`.

</details>

<details>
<summary>Ответ</summary>

```rust
// в src/error.rs импорт становится таким:
//     extract::{
//         FromRequest, FromRequestParts,
//         rejection::{JsonRejection, PathRejection},
//     },

impl From<PathRejection> for ApiError {
    fn from(rejection: PathRejection) -> Self {
        ApiError::Rejected {
            status: rejection.status(),
            message: rejection.body_text(),
        }
    }
}

/// Как `axum::extract::Path`, но плохой путь становится `ApiError`.
#[derive(FromRequestParts)]
#[from_request(via(axum::extract::Path), rejection(ApiError))]
pub struct AppPath<T>(pub T);

// в src/main.rs: `use error::{ApiError, AppJson, AppPath};`, уберите `Path` из
// импорта `extract::{...}` и напишите `AppPath(id): AppPath<u64>` в get_task,
// update_task и delete_task.

    #[tokio::test]
    async fn bad_id_gives_json_error() {
        let app = app(AppState::default());
        let request = Request::get("/tasks/abc").body(Body::empty()).unwrap();
        let (status, body) = send(app, request).await;
        assert_eq!(status, StatusCode::BAD_REQUEST);
        assert_eq!(
            body,
            r#"{"code":"bad_request","message":"Invalid URL: Cannot parse `abc` to a `u64`"}"#
        );
    }
```

Когда все упражнения сделаны, `cargo test` запускает девять тестов:

```text
test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: экстракторы, которые не читают тело, реализуют `FromRequestParts`; derive работает так же.
Теперь у каждой ошибки, которую производит наш код или экстракторы axum, одна форма. (Если хотите, сделайте то же
для `Query`: `QueryRejection`.)

</details>

## Вопросы для самопроверки

1. Зачем `ApiError` реализует `IntoResponse`?

<details>
<summary>Ответ</summary>

Чтобы обработчики могли возвращать `Result<_, ApiError>` и использовать `?`. Тогда превращение ошибки в код статуса
и JSON-тело живёт в одном месте, и у всех ошибок одна форма.

</details>

2. Что значит «разбирайте, а не проверяйте» для `Title`?

<details>
<summary>Ответ</summary>

Вместо проверки `String` во многих местах `Title::parse` один раз проверяет правила и возвращает `Title`. Поскольку
поле приватное, каждый существующий `Title` прошёл проверки.

</details>

3. Что означают `via(axum::Json)` и `rejection(ApiError)` на `AppJson`?

<details>
<summary>Ответ</summary>

Извлекать обычным экстрактором `Json`, а если не получилось, превращать его отказ в `ApiError` (через
`From<JsonRejection>`), чтобы клиент получил наш JSON-формат ошибок.

</details>

4. В `.layer(A).layer(B).layer(C)` какой слой первым видит запрос?

<details>
<summary>Ответ</summary>

`C`, последний: он самый внешний. Запрос идёт C → B → A → обработчик, а ответ возвращается A → B → C.

</details>

5. Что происходит, когда обработчик работает дольше, чем позволяет `TimeoutLayer`?

<details>
<summary>Ответ</summary>

Слой перестаёт ждать, отбрасывает future обработчика (что его отменяет) и возвращает ответ с настроенным статусом,
здесь `503`.

</details>

## Итоги

- Одно перечисление ошибок (`ApiError`) с `impl IntoResponse` даёт каждому сбою одинаковое сопоставление статусов и
  JSON-тело; обработчики возвращают `Result<_, ApiError>` и используют `?`.
- Проверяйте на границе с помощью newtype, единственный конструктор которого проверяет правила (`Title::parse`);
  считайте символы, а не байты; ограничивайте всё.
- `#[derive(FromRequest)]` с `via(...)` и `rejection(...)` заставляет собственные отказы axum использовать ваш
  формат (`FromRequestParts` — для `Path` и `Query`).
- Middleware выполняется вокруг каждого запроса. Слои оборачивают как луковица; последний `.layer()` — самый
  внешний.
- `tower-http` даёт `TraceLayer`, `TimeoutLayer` и `RequestBodyLimitLayer`; `middleware::from_fn` превращает
  асинхронную функцию в ваш собственный middleware.
- Никогда не отправляйте внутренние подробности в ответе с ошибкой; пишите их в лог.

## Источники и что почитать

- [axum::error_handling](https://docs.rs/axum/0.8/axum/error_handling/index.html) — как axum понимает ошибки и почему обработчик не может завершиться неудачей без ответа.
- [axum::middleware](https://docs.rs/axum/0.8/axum/middleware/index.html) — `from_fn`, порядок слоёв и картинка с луковицей.
- [axum `FromRequest` derive](https://docs.rs/axum/0.8/axum/extract/derive.FromRequest.html) — `via` и `rejection` подробно.
- [tower-http](https://docs.rs/tower-http/0.7/tower_http/) — все готовые HTTP-middleware: trace, timeout, limit, CORS (Cross-Origin Resource Sharing, совместное использование ресурсов между источниками), сжатие, идентификаторы запросов.
- [thiserror](https://docs.rs/thiserror/2) — синтаксис `#[error(...)]`.
- [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/) — классическая статья, на которой основана идея `Title` (примеры на Haskell, идея общая).

---

[🏠 Начало](../START-HERE.md) · [← 16.3 Веб-сервисы на axum](03-web-services-with-axum.md) · [📋 Этап 16](README.md) · [16.5 Основы SQL и PostgreSQL →](05-sql-and-postgresql-basics.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
