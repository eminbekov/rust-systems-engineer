<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 16.5 Основы SQL и PostgreSQL](05-sql-and-postgresql-basics.md) · [📋 Этап 16](README.md) · [16.7 Миграции и транзакции →](07-migrations-and-transactions.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/16-backend-services/06-databases-with-sqlx.md) <!-- nav -->

# Урок 16.6 — Работа с базами данных через sqlx

> **Этап 16 — Бэкенд-сервисы** · Урок 6 из 16 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как программа на Rust общается с PostgreSQL через `sqlx`: пул соединений, запросы с параметрами, типизированные строки.
- Чем запросы, проверяемые при **запуске** программы, отличаются от запросов, проверяемых при её **компиляции**.
- Как *офлайн-режим* sqlx позволяет собирать программу без базы данных (для CI — Continuous Integration, непрерывной интеграции — и для Docker).
- Как перенести сервис задач из памяти в PostgreSQL и как превращать ошибки базы данных в безопасный `500`.
- Как тестировать сервис на настоящей свежей базе данных с помощью `#[sqlx::test]`.

## Перед началом

- Вы прошли [Урок 16.5 — Основы SQL и PostgreSQL](05-sql-and-postgresql-basics.md). Контейнер `p16-postgres`
  работает (если нет — `docker start p16-postgres`), а в базе данных `tasks` есть таблицы `tasks` и `comments`. Обе
  команды `CREATE TABLE` вы сохранили в `code/p16-tasks/schema.sql`.
- Ваш `code/p16-tasks` — версия из [Урока 16.4](04-errors-validation-and-middleware.md).
- Вы помните SQL-внедрение (Structured Query Language, язык структурированных запросов) и параметры из [Урока 10.5](../10-security-fundamentals/05-injection-attacks.md).

## Новые слова

| Слово | Значение |
|---|---|
| **sqlx** | Асинхронный крейт Rust для SQL-баз данных. Вы пишете обычный SQL; sqlx отправляет его с параметрами и превращает строки в значения Rust. Это не ORM (Object-Relational Mapper, объектно-реляционное отображение). |
| **ORM** (Object-Relational Mapper, объектно-реляционное отображение) | Библиотека, которая сама генерирует SQL по вызовам методов у объектов. sqlx намеренно этого не делает. |
| **драйвер** (driver) | Код, который говорит на сетевом протоколе базы данных. В sqlx есть драйвер PostgreSQL, написанный на Rust. |
| **строка подключения** (connection string) | Один URL (Uniform Resource Locator, единый указатель ресурса) со всем, что нужно для подключения: `postgres://user:password@host:port/database`. |
| **пул соединений** (connection pool) | Набор открытых соединений с базой данных, которые запросы берут и возвращают, чтобы ни одному запросу не приходилось открывать своё соединение. |
| **параметр запроса** (query parameter, bind parameter) | Заполнитель вроде `$1` в тексте SQL, значение которого отправляется отдельно. Это защита от SQL-внедрения. |
| **запрос, проверяемый при компиляции** (compile-time checked query) | Запрос, записанный макросом sqlx (`query!`, `query_as!`), который компилятор проверяет на настоящей базе данных: синтаксис SQL, имена столбцов и типы. |
| **офлайн-режим** (offline mode) | Сборка по сохранённым описаниям запросов (папка `.sqlx`) вместо работающей базы данных. |
| **файл `.env`** | Файл со строками `NAME=value`, который инструменты и программы могут загрузить как переменные окружения во время разработки. |

## 1. sqlx на одной картинке

```text
  handler ──▶ pool.acquire() ──▶ connection ──TCP──▶ PostgreSQL
     ▲          (borrow)              │  "SELECT ... WHERE id = $1"  +  [7]
     │                                ▼
     └────── Task { id: 7, ... } ◀── rows ◀────────────────────────────
```

sqlx даёт три вещи: **пул** соединений, способ отправлять SQL **с параметрами** и превращение строк результата в
структуры Rust. Каждый запрос вы по-прежнему пишете сами, на SQL. В этом и смысл: вы точно знаете, что выполняет
база данных, и всё, что вы изучили в Уроке 16.5, применяется напрямую.

## 2. Подготовка

Добавьте sqlx (драйвер `postgres`, Tokio как рантайм, макросы) и `dotenvy` — небольшой крейт, который загружает
файл `.env`:

```bash
cd ~/projects/rust/code/p16-tasks
cargo add sqlx@0.9 --features runtime-tokio,postgres,macros
cargo add dotenvy@0.15
```

```toml
dotenvy = "0.15"
sqlx = { version = "0.9", features = ["runtime-tokio", "postgres", "macros"] }
```

sqlx включает ещё несколько фич по умолчанию, среди них `migrate`, которую использует Урок 16.7. Теперь сообщите
программе и макросам sqlx, где находится база данных. Создайте `code/p16-tasks/.env` с одной строкой:

```text
DATABASE_URL=postgres://postgres:dev-password@127.0.0.1:5432/tasks
```

Части строки подключения: протокол `postgres`, пользователь `postgres`, пароль `dev-password`, хост
`127.0.0.1`, порт `5432`, база данных `tasks`.

> 🔒 **Безопасность:** в `.env` лежит пароль, поэтому он **не** должен попасть в Git. Прямо сейчас добавьте строку
> `.env` в `code/p16-tasks/.gitignore`. В настоящих окружениях переменную задаёт платформа, а не файл в репозитории
> (двенадцать факторов, Урок 08.15; Урок 16.13).

## 3. Первые запросы

Прежде чем переписывать сервис, попробуйте sqlx в маленькой отдельной программе. Создайте `src/bin/db_demo.rs`:

```rust
use sqlx::postgres::PgPoolOptions;

#[derive(Debug, sqlx::FromRow)]
struct Task {
    id: i64,
    title: String,
    done: bool,
}

#[tokio::main]
async fn main() -> Result<(), sqlx::Error> {
    dotenvy::dotenv().ok();
    let url = std::env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .connect(&url)
        .await?;

    // Проверяется базой данных, когда программа выполняется.
    let task: Task =
        sqlx::query_as("INSERT INTO tasks (title) VALUES ($1) RETURNING id, title, done")
            .bind("Learn sqlx")
            .fetch_one(&pool)
            .await?;
    println!("inserted: {task:?}");

    // Проверяется компилятором, на настоящей базе данных, когда программа собирается.
    let open = sqlx::query_as!(
        Task,
        "SELECT id, title, done FROM tasks WHERE NOT done ORDER BY id"
    )
    .fetch_all(&pool)
    .await?;
    for task in &open {
        println!("open: {} {} (done: {})", task.id, task.title, task.done);
    }
    Ok(())
}
```

Запустите через `cargo run --bin db_demo`:

```text
inserted: Task { id: 11, title: "Learn sqlx", done: false }
open: 2 Write lesson (done: false)
open: 6 Read a book (done: false)
open: 11 Learn sqlx (done: false)
```

Ваши идентификаторы зависят от того, что вы делали в Уроке 16.5; пропуски — это неудачные вставки оттуда.

**Что происходит:**

1. `dotenvy::dotenv().ok()` читает `.env` и задаёт его переменные для этого процесса (`.ok()` игнорирует
   отсутствие файла, так что настоящая переменная окружения тоже работает). Затем `std::env::var` читает
   `DATABASE_URL`.
2. `PgPoolOptions::new().max_connections(5).connect(&url)` открывает **пул**: до пяти соединений, которые
   открываются по необходимости и используются повторно. Клонировать `PgPool` дёшево; все клоны разделяют одни и те
   же соединения.
3. `sqlx::query_as("...")` готовит запрос, строки которого становятся `Task`. `$1` — **параметр**;
   `.bind("Learn sqlx")` задаёт его значение. Значение передаётся отдельно от текста SQL, поэтому никакой ввод не может
   изменить SQL (Урок 10.5).
4. `#[derive(sqlx::FromRow)]` учит sqlx строить `Task` из строки по именам столбцов. `fetch_one` ожидает ровно одну
   строку; её даёт `RETURNING`.
5. `sqlx::query_as!` (с `!`, это макрос) делает ту же работу, но проверяется **при компиляции** (раздел 4).
   `fetch_all` возвращает `Vec<Task>`.

Методы `fetch_*`:

| Метод | Возвращает | Когда использовать |
|---|---|---|
| `fetch_one` | `T`, ошибку, если строки нет | должна существовать ровно одна строка (`INSERT ... RETURNING`) |
| `fetch_optional` | `Option<T>` | ноль или одна строка (`WHERE id = $1`) |
| `fetch_all` | `Vec<T>` | список |
| `execute` | сколько строк затронуто | `UPDATE`/`DELETE` без `RETURNING` |

> ⚠️ **Частая ошибка:** в PostgreSQL нет беззнаковых целых. `BIGINT` в Rust — это `i64`, поэтому отныне
> идентификаторы — `i64`, а не `u64`, как в Уроке 16.3.

## 4. Запросы, которые проверяет компилятор

Первый запрос (`query_as`, функция) для компилятора — просто текст. Опечатка в имени столбца обнаружится, только
когда эта строка **выполнится**, возможно, в продакшене. Макрос устроен иначе: во время компиляции он подключается к
базе данных из `DATABASE_URL`, просит PostgreSQL проанализировать SQL и сверяет результат с вашими типами Rust.

Сделайте нарочно опечатку — `titel` вместо `title` — в `query_as!` и выполните `cargo build`:

```text
error: error returned from database: column "titel" does not exist at line 3861
  --> src/bin/db_demo.rs:28:16
   |
28 |       let open = sqlx::query_as!(
   |  ________________^
29 | |         Task,
30 | |         "SELECT id, titel, done FROM tasks WHERE NOT done ORDER BY id"
31 | |     )
   | |_____^
```

**Компилятор** сообщает собственную ошибку PostgreSQL. («line 3861» — строка внутри исходного кода PostgreSQL; не
обращайте внимания.) Исправьте опечатку, а теперь поменяйте поле структуры на `id: i32`:

```text
error[E0277]: the trait bound `i32: From<i64>` is not satisfied
  --> src/bin/db_demo.rs:28:16
   |
28 |       let open = sqlx::query_as!(
   |  ________________^
...
   | |_____^ the trait `From<i64>` is not implemented for `i32`
```

PostgreSQL сообщил, что столбец — `BIGINT` (`i64`), а `i64` не всегда помещается в `i32`. Верните `i64`.

Как макросы сопоставляют типы:

| Столбец | Тип Rust от макроса |
|---|---|
| `BIGINT NOT NULL` | `i64` |
| `TEXT NOT NULL` | `String` |
| `BOOLEAN NOT NULL` | `bool` |
| любой столбец **без** `NOT NULL` | `Option<…>` от этого типа |

Это ещё одна причина ставить `NOT NULL` везде, где можно: типы Rust остаются простыми.

> 💡 **Совет:** используйте макросы (`query!`, `query_as!`, `query_scalar!`) для каждого запроса с неизменным текстом.
> Функции (`query`, `query_as`) — когда код должен собираться без базы данных и без данных `.sqlx` (так делает
> Урок 16.14) или когда часть текста SQL выбирается во время выполнения (частая ошибка 3). Значения всегда идут в
> параметры, никогда в текст.

## 5. Офлайн-режим

Проверкам при компиляции нужна работающая база данных при каждой сборке. На ноутбуке это нормально. В CI
(Урок 09.12) или внутри `docker build` (Урок 16.13) базы данных нет. Остановите контейнер и «потрогайте» файл, чтобы
макросы выполнились снова:

```bash
docker stop p16-postgres
touch src/bin/db_demo.rs
cargo build
```

```text
error: error communicating with database: Connection refused (os error 61)
```

Решение — **офлайн-режим**: вы один раз сохраняете описания всех запросов в репозитории, и сборки используют их
вместо базы данных. Инструмент для этого — `sqlx-cli`. Установите его один раз (это займёт несколько минут; фичи
делают его компактным):

```bash
cargo install --locked sqlx-cli --no-default-features --features rustls,postgres
cargo sqlx --version
```

```text
sqlx-cli-sqlx 0.9.0
```

(У вас может оказаться более новая `0.9.x`.) Снова запустите базу данных и подготовьте данные:

```bash
docker start p16-postgres
cargo sqlx prepare
```

```text
query data written to .sqlx in the current directory; please check this into version control
```

В папке `.sqlx` теперь лежит по одному JSON-файлу (JavaScript Object Notation, нотация объектов JavaScript) на запрос: текст SQL, имена и типы столбцов, какие параметры он
принимает. Сборка с `SQLX_OFFLINE=true` использует только эти файлы:

```bash
docker stop p16-postgres
touch src/bin/db_demo.rs
SQLX_OFFLINE=true cargo build
docker start p16-postgres
```

```text
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.60s
```

> 📏 **Соглашение:** коммитьте `.sqlx` в Git (как и сказано в сообщении) и снова выполняйте `cargo sqlx prepare`,
> когда добавляете или меняете запрос. В CI `cargo sqlx prepare --check` завершается ошибкой, если сохранённые данные
> устарели.

## 6. Сервис задач на PostgreSQL

Теперь замените хранилище в памяти. Четыре изменения, файл за файлом.

**`src/error.rs`:** идентификатор становится `i64`, а новый вариант оборачивает любую `sqlx::Error`. `#[from]` (из
`thiserror`) позволяет `?` автоматически превращать `sqlx::Error` в `ApiError`:

```rust
/// Все способы, которыми может не удаться запрос к API задач.
#[derive(Debug, thiserror::Error)]
pub enum ApiError {
    #[error("task {0} not found")]
    NotFound(i64),
    #[error("{0}")]
    Validation(String),
    #[error("{message}")]
    Rejected { status: StatusCode, message: String },
    #[error("internal error")]
    Database(#[from] sqlx::Error),
}
```

В `status()` добавьте `ApiError::Database(_) => StatusCode::INTERNAL_SERVER_ERROR,`, в `code()` —
`ApiError::Database(_) => "internal",`, а `into_response` пусть пишет подробности в лог, прежде чем строить тело:

```rust
impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        if let ApiError::Database(error) = &self {
            // Подробности идут в лог, никогда клиенту.
            tracing::error!(%error, "database error");
        }
        let body = ErrorBody {
            code: self.code(),
            message: self.to_string(),
        };
        (self.status(), Json(body)).into_response()
    }
}
```

Текст `Display` у `Database` — просто `internal error`, поэтому клиент никогда не видит, что пошло не так внутри.
`%error` записывает ошибку с её текстом `Display` как поле события лога (Урок 09.10).

**`src/main.rs`, импорты:** уберите `BTreeMap`, `Arc` и `Mutex` и добавьте типы пула:

```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::time::Duration;

use sqlx::PgPool;
use sqlx::postgres::PgPoolOptions;
```

(Импорты `axum`, `serde`, `tower_http`, `error` и `title` остаются.) В `Task` замените `id: u64` на `id: i64`. Удалите
`Store` и `type AppState` и добавьте вместо них структуру состояния:

```rust
#[derive(Clone)]
struct AppState {
    pool: PgPool,
}
```

**Обработчики:** каждый становится одним SQL-запросом:

```rust
async fn list_tasks(
    State(state): State<AppState>,
    Query(filter): Query<ListFilter>,
) -> Result<Json<Vec<Task>>, ApiError> {
    let tasks = sqlx::query_as!(
        Task,
        "SELECT id, title, done FROM tasks
         WHERE ($1::boolean IS NULL OR done = $1)
         ORDER BY id",
        filter.done
    )
    .fetch_all(&state.pool)
    .await?;
    Ok(Json(tasks))
}

async fn create_task(
    State(state): State<AppState>,
    AppJson(new_task): AppJson<NewTask>,
) -> Result<impl IntoResponse, ApiError> {
    let title = Title::parse(&new_task.title)?.into_string();
    let task = sqlx::query_as!(
        Task,
        "INSERT INTO tasks (title) VALUES ($1) RETURNING id, title, done",
        title
    )
    .fetch_one(&state.pool)
    .await?;
    let location = format!("/tasks/{}", task.id);
    Ok((
        StatusCode::CREATED,
        [(header::LOCATION, location)],
        Json(task),
    ))
}

async fn get_task(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Json<Task>, ApiError> {
    let task = sqlx::query_as!(Task, "SELECT id, title, done FROM tasks WHERE id = $1", id)
        .fetch_optional(&state.pool)
        .await?;
    task.map(Json).ok_or(ApiError::NotFound(id))
}

async fn update_task(
    State(state): State<AppState>,
    Path(id): Path<i64>,
    AppJson(changes): AppJson<TaskChanges>,
) -> Result<Json<Task>, ApiError> {
    let title = changes
        .title
        .as_deref()
        .map(Title::parse)
        .transpose()?
        .map(Title::into_string);
    let task = sqlx::query_as!(
        Task,
        "UPDATE tasks
         SET title = COALESCE($2, title), done = COALESCE($3, done)
         WHERE id = $1
         RETURNING id, title, done",
        id,
        title,
        changes.done
    )
    .fetch_optional(&state.pool)
    .await?;
    task.map(Json).ok_or(ApiError::NotFound(id))
}

async fn delete_task(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<StatusCode, ApiError> {
    let result = sqlx::query!("DELETE FROM tasks WHERE id = $1", id)
        .execute(&state.pool)
        .await?;
    if result.rows_affected() == 0 {
        return Err(ApiError::NotFound(id));
    }
    Ok(StatusCode::NO_CONTENT)
}
```

**Что происходит:**

1. В `list_tasks` `filter.done` — это `Option<bool>`; sqlx отправляет `None` как `NULL`.
   `$1::boolean IS NULL OR done = $1` означает «фильтра нет или фильтр совпадает», так что один запрос обслуживает и
   `/tasks`, и `/tasks?done=false`. `::boolean` сообщает PostgreSQL тип `$1`.
2. `create_task`, как и раньше, сначала проверяет ввод, а затем позволяет базе данных выбрать `id` (`RETURNING`).
3. `get_task` использует `fetch_optional`: отсутствие строки становится `None`, а `ok_or` превращает это в `NotFound`.
4. `update_task` использует `COALESCE(a, b)`, который возвращает первое значение, не равное `NULL`. Отсутствующее
   поле отправляется как `NULL`, поэтому `COALESCE($2, title)` сохраняет старый заголовок. Поиск строки и её изменение
   происходят в **одном** операторе, а `RETURNING` возвращает новую строку; если строки с таким `id` нет,
   `fetch_optional` возвращает `None`.
5. `delete_task` использует `execute`, который сообщает `rows_affected()`; `0` значит, что задачи не было.
6. Каждый `.await?` превращает `sqlx::Error` в `ApiError::Database` через `#[from]`.

**`main`:** загрузить `.env`, построить пул, отдать его маршрутизатору. Сама `app()` не меняется:

```rust
#[tokio::main]
async fn main() {
    dotenvy::dotenv().ok();
    tracing_subscriber::fmt()
        .with_env_filter(
            tracing_subscriber::EnvFilter::try_from_default_env()
                .unwrap_or_else(|_| "p16_tasks=debug,tower_http=debug".into()),
        )
        .init();

    let database_url = std::env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let pool = PgPoolOptions::new()
        .max_connections(5)
        .acquire_timeout(Duration::from_secs(3))
        .connect(&database_url)
        .await
        .expect("cannot connect to the database");

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    tracing::info!("listening on http://127.0.0.1:3000");
    axum::serve(listener, app(AppState { pool })).await.unwrap();
}
```

`acquire_timeout` ограничивает, сколько запрос ждёт свободного соединения: три секунды, затем ошибка вместо
бесконечного ожидания.

> ⚡ **Производительность:** пул — это не «чем больше, тем лучше». Каждое соединение PostgreSQL — процесс на сервере
> базы данных со своей памятью, а у сервера есть предел (по умолчанию 100), общий для **всех** экземпляров вашего
> сервиса. Начинайте с малого (5–10 на экземпляр) и измеряйте.

## 7. Запуск

`cargo run`, затем поговорите с сервисом. Задачи, которые вы создали через `psql` и `db_demo`, уже там (в выводе ниже
опущен заголовок `date`):

```bash
curl -s http://127.0.0.1:3000/tasks; echo
curl -i -H 'content-type: application/json' -d '{"title":"Learn PostgreSQL"}' http://127.0.0.1:3000/tasks
```

```text
[{"id":2,"title":"Write lesson","done":false},{"id":6,"title":"Read a book","done":false},{"id":7,"title":"Buy oat milk","done":true},{"id":8,"title":"Milk the cow","done":true},{"id":11,"title":"Learn sqlx","done":false}]
HTTP/1.1 201 Created
content-type: application/json
location: /tasks/12
x-request-id: 2
content-length: 49

{"id":12,"title":"Learn PostgreSQL","done":false}
```

Перезапустите сервис и запросите задачу 12: она на месте, потому что теперь живёт в PostgreSQL. Затем посмотрите,
что происходит, когда база данных исчезает. Остановите её, **пока сервис работает**, и отправьте запрос:

```bash
docker stop p16-postgres
curl -s http://127.0.0.1:3000/tasks/6; echo
docker start p16-postgres
```

```text
{"code":"internal","message":"internal error"}
```

Клиент получает безопасный ответ примерно через три секунды (`acquire_timeout`). В логе сервиса есть подробности:

```text
2026-11-18T07:31:54.069175Z ERROR request{method=GET uri=/tasks/6 version=HTTP/1.1}: p16_tasks::error: database error error=pool timed out while waiting for an open connection
2026-11-18T07:31:54.069283Z ERROR request{method=GET uri=/tasks/6 version=HTTP/1.1}: tower_http::trace::on_failure: response failed classification=Status code: 500 Internal Server Error latency=3002 ms
```

Сервис не упал. Как только PostgreSQL вернётся, следующий запрос снова сработает: пул сам откроет новые соединения.

## 8. Тесты с настоящей базой данных

Тестам из Урока 16.4, работавшим в памяти, теперь нужна база данных. `#[sqlx::test]` даёт каждому тесту свою
**свежую, пустую** базу данных: он подключается через `DATABASE_URL`, создаёт новую базу со сгенерированным именем,
передаёт тесту пул и удаляет базу, когда тест проходит. Тесты никогда не видят данных друг друга и могут идти
параллельно.

В свежей базе нет таблиц. Небольшой помощник сначала выполняет `schema.sql` из Урока 16.5. В модуле `tests`
замените `app(AppState::default())` этим помощником и превратите каждый тест в `#[sqlx::test]`, который получает пул:

```rust
    /// Создаёт таблицы в пустой тестовой базе данных, которую даёт нам `#[sqlx::test]`.
    async fn test_app(pool: PgPool) -> Router {
        sqlx::raw_sql(include_str!("../schema.sql"))
            .execute(&pool)
            .await
            .unwrap();
        app(AppState { pool })
    }

    #[sqlx::test]
    async fn created_task_can_be_read_back(pool: PgPool) {
        let app = test_app(pool).await;

        let (status, body) = send(app.clone(), post_json("/tasks", r#"{"title":"Buy milk"}"#)).await;
        assert_eq!(status, StatusCode::CREATED);
        assert_eq!(body, r#"{"id":1,"title":"Buy milk","done":false}"#);

        let request = Request::get("/tasks/1").body(Body::empty()).unwrap();
        let (status, body) = send(app, request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, r#"{"id":1,"title":"Buy milk","done":false}"#);
    }
```

Так же измените `unknown_task_is_not_found` и `empty_title_is_rejected` (`#[sqlx::test]`, параметр `pool: PgPool`,
`let app = test_app(pool).await;`). `include_str!` вставляет текст файла в программу при компиляции; `raw_sql`
выполняет несколько команд SQL сразу. Запустите `cargo test`:

```text
running 7 tests
test title::tests::rejects_empty_and_blank ... ok
test title::tests::trims_spaces ... ok
test title::tests::rejects_control_characters ... ok
test title::tests::length_limit_counts_characters_not_bytes ... ok
test tests::empty_title_is_rejected ... ok
test tests::unknown_task_is_not_found ... ok
test tests::created_task_can_be_read_back ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.28s
```

(Четыре теста `title::tests` — из упражнения 1 Урока 16.4; тесты маршрутизатора из других его упражнений здесь
не показаны. Если вы их сохранили, превратите их в `#[sqlx::test]` так же.) Поскольку каждая тестовая база новая, первая задача
всегда получает `id` 1, и проверки точного JSON продолжают работать.

> 🧪 **Тестирование:** эти тесты медленнее чистых модульных тестов (здесь четверть секунды), зато проверяют
> настоящее: ваш SQL, ваши ограничения, ваше сопоставление типов. Урок 16.14 идёт дальше и запускает сам PostgreSQL
> из тестов через `testcontainers`, так что `cargo test` работает на любой машине с Docker.

## Частые ошибки

**1. Нет `DATABASE_URL` при сборке.** Без `.env` (и без переменной или `.sqlx`) макросы не могут работать:

```text
error: set `DATABASE_URL` to use query macros online, or run `cargo sqlx prepare` to update the query cache
```

Проверьте, что `.env` лежит в папке пакета (рядом с `Cargo.toml`) и что база данных запущена, или собирайте с
`SQLX_OFFLINE=true`. Собственному анализу RustRover нужно то же самое, так что держите контейнер запущенным, пока
работаете.

**2. Беззнаковые идентификаторы.** `Path(id): Path<u64>`, переданный в запрос:

```text
error[E0277]: the trait bound `u64: sqlx::Encode<'_, Postgres>` is not satisfied
```

В PostgreSQL нет беззнаковых типов; для `BIGINT` используйте `i64`. Отрицательный `id` в URL просто не найдёт строку:
`404`.

**3. Сборка SQL через `format!`.** Значение внутри текста SQL, как в
`sqlx::query(&format!("... WHERE title = '{title}'"))`, — это SQL-внедрение (Урок 10.5). sqlx 0.9 отвергает такой код
при компиляции: функции принимают только строковый литерал (`&'static str`):

```text
error[E0277]: dynamic SQL strings should be audited for possible injections
    = help: the trait `SqlSafeStr` is not implemented for `&String`
```

Значения идут в `$1`, `$2`. Только когда часть SQL не может быть параметром (например, имя столбца, выбранное из
фиксированного списка в вашем коде), вы собираете текст сами и оборачиваете его в `sqlx::AssertSqlSafe(text)`: это
значит «я проверил, что эта строка безопасна». Макросы никогда не принимают ничего, кроме литерала.

**4. Показ ошибок базы данных.** `message: error.to_string()` для `sqlx::Error` может содержать имена таблиц, SQL и
подробности подключения. Пишите их в лог; отвечайте `internal error`.

**5. Забытый `cargo sqlx prepare`.** Вы добавляете запрос, CI собирает в офлайн-режиме и падает с ошибкой, что
запроса нет в кэше. Выполняйте `prepare` перед коммитом.

## Попробуйте сами

1. Посмотрите один файл в `.sqlx`. Найдите текст SQL, столбец `title` с `"type_info": "Text"` и `"nullable"`.
2. Запустите сервис и `psql` рядом. Вставьте задачу в `psql` и прочитайте её через `curl`; удалите одну через `curl`
   и проверьте в `psql`. Два клиента, один источник истины.
3. Пока идёт `cargo test`, поищите в `psql` тестовые базы: `\l` показывает имена, начинающиеся с `_sqlx_test`. Они
   исчезают, когда тесты проходят.
4. Сделайте коммит (без `.env`!): `git add code/p16-tasks && git commit -m "p16-tasks: store tasks in PostgreSQL"`.

## Упражнения

### Упражнение 1 — `limit` в SQL (легко)

Верните параметр запроса `limit` из Урока 16.3 (упражнение 2), но в SQL: добавьте `limit: Option<i64>` в
`ListFilter`, добавьте `LIMIT $2` в запрос, используйте 50, если он не указан, и держите его между 1 и 100. Напишите
`#[sqlx::test]`, который создаёт задачи `a`, `b`, `c` и проверяет, что `?limit=2` возвращает первые две.

<details>
<summary>Подсказка</summary>

`filter.limit.unwrap_or(50).clamp(1, 100)` даёт значение; передайте его вторым аргументом `query_as!`. Затем снова
выполните `cargo sqlx prepare`: текст запроса изменился.

</details>

<details>
<summary>Ответ</summary>

```rust
// в ListFilter:      limit: Option<i64>,
// в list_tasks:
//     "SELECT id, title, done FROM tasks
//      WHERE ($1::boolean IS NULL OR done = $1)
//      ORDER BY id
//      LIMIT $2",
//     filter.done,
//     filter.limit.unwrap_or(50).clamp(1, 100)

    #[sqlx::test]
    async fn limit_returns_only_the_first_tasks(pool: PgPool) {
        let app = test_app(pool).await;
        send(app.clone(), post_json("/tasks", r#"{"title":"a"}"#)).await;
        send(app.clone(), post_json("/tasks", r#"{"title":"b"}"#)).await;
        send(app.clone(), post_json("/tasks", r#"{"title":"c"}"#)).await;

        let request = Request::get("/tasks?limit=2").body(Body::empty()).unwrap();
        let (status, body) = send(app, request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(
            body,
            r#"[{"id":1,"title":"a","done":false},{"id":2,"title":"b","done":false}]"#
        );
    }
```

Почему это работает: ограничивает база данных, поэтому по сети уходят только две строки, а не все. `clamp(1, 100)`
ещё и отклоняет `limit=0` и отрицательные значения, которые PostgreSQL обработал бы иначе.

</details>

### Упражнение 2 — Тест `PATCH` (средне)

Напишите `#[sqlx::test]` `patch_changes_only_the_given_fields`: создайте «Buy milk»; отправьте `PATCH`
`{"done":true}` и проверьте, что заголовок не изменился; отправьте `PATCH` `{"title":"Buy oat milk"}` и проверьте, что
`done` остался `true`; отправьте `PATCH` для задачи 99 и ожидайте `404`.

<details>
<summary>Подсказка</summary>

Замыкание `let patch = |json: &'static str| Request::patch("/tasks/1")...` строит запросы. Каждому `send`, кроме
последнего, нужен `app.clone()`.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[sqlx::test]
    async fn patch_changes_only_the_given_fields(pool: PgPool) {
        let app = test_app(pool).await;
        send(app.clone(), post_json("/tasks", r#"{"title":"Buy milk"}"#)).await;

        let patch = |json: &'static str| {
            Request::patch("/tasks/1")
                .header(header::CONTENT_TYPE, "application/json")
                .body(Body::from(json))
                .unwrap()
        };
        let (status, body) = send(app.clone(), patch(r#"{"done":true}"#)).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, r#"{"id":1,"title":"Buy milk","done":true}"#);

        let (_, body) = send(app.clone(), patch(r#"{"title":"Buy oat milk"}"#)).await;
        assert_eq!(body, r#"{"id":1,"title":"Buy oat milk","done":true}"#);

        let missing = Request::patch("/tasks/99")
            .header(header::CONTENT_TYPE, "application/json")
            .body(Body::from(r#"{"done":true}"#))
            .unwrap();
        let (status, _) = send(app, missing).await;
        assert_eq!(status, StatusCode::NOT_FOUND);
    }
```

Почему это работает: этот тест проверяет логику `COALESCE`, которая живёт в SQL, поэтому проверить её может только
тест на настоящей базе данных. Тест с фейковым хранилищем проверял бы фейк, а не запрос.

</details>

### Упражнение 3 — Комментарии задачи (сложно)

Добавьте `GET /tasks/{id}/comments`: он возвращает комментарии задачи в виде
`[{"id":1,"task_id":1,"text":"Two bottles"}]`, от старых к новым, и `404`, если не существует **задачи** (задача без
комментариев даёт `[]`). Напишите тест, который вставляет комментарий напрямую через `sqlx::query!` и проверяет оба
случая.

<details>
<summary>Подсказка</summary>

Два запроса: сначала `SELECT EXISTS (SELECT 1 FROM tasks WHERE id = $1)`, затем комментарии. Макрос не может знать,
что `EXISTS` никогда не возвращает `NULL`, поэтому выдал бы `Option<bool>`; напишите имя столбца как `AS "exists!"`
(`!` говорит sqlx «не NULL») и используйте `query_scalar!`, который возвращает единственное значение строки из одного
столбца. В тесте передайте `pool.clone()` в `test_app`, чтобы у вас остался пул для вставки.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, Serialize)]
struct Comment {
    id: i64,
    task_id: i64,
    text: String,
}

async fn list_comments(
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Json<Vec<Comment>>, ApiError> {
    let exists = sqlx::query_scalar!(
        r#"SELECT EXISTS (SELECT 1 FROM tasks WHERE id = $1) AS "exists!""#,
        id
    )
    .fetch_one(&state.pool)
    .await?;
    if !exists {
        return Err(ApiError::NotFound(id));
    }
    let comments = sqlx::query_as!(
        Comment,
        "SELECT id, task_id, text FROM comments WHERE task_id = $1 ORDER BY id",
        id
    )
    .fetch_all(&state.pool)
    .await?;
    Ok(Json(comments))
}

// в app():   .route("/tasks/{id}/comments", get(list_comments))

    #[sqlx::test]
    async fn comments_of_a_task(pool: PgPool) {
        let app = test_app(pool.clone()).await;
        send(app.clone(), post_json("/tasks", r#"{"title":"Buy milk"}"#)).await;
        sqlx::query!("INSERT INTO comments (task_id, text) VALUES (1, 'Two bottles')")
            .execute(&pool)
            .await
            .unwrap();

        let request = Request::get("/tasks/1/comments").body(Body::empty()).unwrap();
        let (status, body) = send(app.clone(), request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, r#"[{"id":1,"task_id":1,"text":"Two bottles"}]"#);

        let request = Request::get("/tasks/2/comments").body(Body::empty()).unwrap();
        let (status, _) = send(app, request).await;
        assert_eq!(status, StatusCode::NOT_FOUND);
    }
```

Со всеми тремя упражнениями `cargo test` запускает десять тестов:

```text
test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.35s
```

Почему это работает: `404` для отсутствующей задачи следует проекту из Урока 16.2 (упражнение 2): «нет комментариев»
и «нет такой задачи» — разные ответы. `r#"..."#` нужен, потому что в SQL есть двойные кавычки. Между двумя запросами
другой запрос мог бы удалить задачу; Урок 16.7 показывает, как транзакция убирает такие щели.

</details>

## Вопросы для самопроверки

1. Почему сервис использует пул соединений, а не открывает соединение на каждый запрос?

<details>
<summary>Ответ</summary>

Открытие соединения — дело медленное (сетевые обмены, аутентификация, новый серверный процесс), а база данных
допускает лишь ограниченное число соединений. Пул держит несколько соединений открытыми и одалживает их запросам.

</details>

2. Чем `sqlx::query_as(...)` отличается от `sqlx::query_as!(...)`?

<details>
<summary>Ответ</summary>

Функция проверяется только при выполнении; ошибки проявляются во время работы. Макрос проверяется при компиляции на
настоящей базе данных: синтаксис SQL, имена столбцов, типы параметров и результатов.

</details>

3. Как `$1` с `.bind(value)` (или аргументом макроса) защищает от SQL-внедрения?

<details>
<summary>Ответ</summary>

Текст SQL и значение передаются отдельно; база данных никогда не интерпретирует значение как SQL. Что бы ни было в
значении, оно остаётся значением.

</details>

4. Для чего нужна папка `.sqlx` и когда её нужно обновлять?

<details>
<summary>Ответ</summary>

Она хранит описания всех запросов, проверяемых при компиляции, чтобы сборки с `SQLX_OFFLINE=true` работали без базы
данных (CI, Docker). Выполняйте `cargo sqlx prepare` при каждом добавлении или изменении запроса и коммитьте результат.

</details>

5. Почему `ApiError::Database` показывает только `internal error`?

<details>
<summary>Ответ</summary>

Ошибки базы данных могут содержать внутренние подробности (SQL, имена таблиц, сведения о подключении). Клиент получает
общий `500`; подробности идут в лог.

</details>

6. Что `#[sqlx::test]` даёт каждому тесту?

<details>
<summary>Ответ</summary>

Пул, подключённый к новой пустой базе данных, созданной только для этого теста (и удаляемой после его прохождения),
поэтому тесты изолированы и могут идти параллельно.

</details>

## Итоги

- sqlx отправляет ваш собственный SQL с параметрами и превращает строки в значения Rust; это драйвер плюс помощники, а
  не ORM.
- `PgPool` (из `PgPoolOptions`) разделяет несколько соединений между всеми запросами; клонируйте его в состояние
  приложения.
- `fetch_one`, `fetch_optional`, `fetch_all` и `execute` покрывают все формы результатов.
- Макросы `query!`, `query_as!` и `query_scalar!` проверяют SQL и типы при компиляции через `DATABASE_URL`;
  `cargo sqlx prepare` и `SQLX_OFFLINE=true` позволяют собирать без базы данных.
- Превращайте `sqlx::Error` в общий `500` и пишите подробности в лог; `BIGINT` — это `i64`, столбцы с `NULL` — `Option`.
- `#[sqlx::test]` запускает каждый тест на собственной свежей базе данных.

## Источники и что почитать

- [Документация sqlx](https://docs.rs/sqlx/0.9) — обзор крейта, фичи и все типы.
- [sqlx `query!` macro](https://docs.rs/sqlx/0.9/sqlx/macro.query.html) — как работает проверка при компиляции, переопределения типов вроде `"name!"` и офлайн-режим.
- [sqlx-cli README](https://github.com/launchbadge/sqlx/blob/main/sqlx-cli/README.md) — команды `cargo sqlx prepare`, `database` и `migrate`.
- [sqlx `#[sqlx::test]`](https://docs.rs/sqlx/0.9/sqlx/attr.test.html) — тестовые базы данных, миграции и фикстуры.
- [PostgreSQL — Connection URIs](https://www.postgresql.org/docs/18/libpq-connect.html) — каждая часть строки подключения (раздел 32.1.1).
- [PostgreSQL — COALESCE](https://www.postgresql.org/docs/18/functions-conditional.html) — `COALESCE` и другие условные выражения.

---

[🏠 Начало](../START-HERE.md) · [← 16.5 Основы SQL и PostgreSQL](05-sql-and-postgresql-basics.md) · [📋 Этап 16](README.md) · [16.7 Миграции и транзакции →](07-migrations-and-transactions.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
