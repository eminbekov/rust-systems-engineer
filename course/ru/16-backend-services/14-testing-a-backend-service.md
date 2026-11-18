<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 16.13 Конфигурация, проверки работоспособности и контейнеры](13-configuration-health-checks-and-containers.md) · [📋 Этап 16](README.md) · [16.15 Проект: сервис коротких ссылок →](15-project-url-shortener-service.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/16-backend-services/14-testing-a-backend-service.md) <!-- nav -->

# Урок 16.14 — Тестирование бэкенд-сервиса

> **Этап 16 — Бэкенд-сервисы** · Урок 14 из 16 · ⏱ около 135 минут

## Что вы узнаете

- Какие виды тестов нужны бэкенд-сервису и как к нему применяется пирамида тестирования.
- Как отправлять HTTP-запросы (HTTP, Hypertext Transfer Protocol, протокол передачи гипертекста) в роутер axum внутри теста, без сети, с помощью `oneshot`.
- Как тест может сам запустить PostgreSQL в Docker через `testcontainers`, чтобы `cargo test` не требовал ручной подготовки.
- Как `#[sqlx::test]` даёт каждому тесту свежую базу данных с применёнными миграциями на одном общем сервере.
- Как держать тесты независимыми, быстрыми и честными.

## Перед началом

- Вы прошли [Урок 16.13 — Конфигурация, проверки работоспособности и контейнеры](13-configuration-health-checks-and-containers.md).
- Вы знаете пирамиду тестирования ([Урок 09.1](../09-testing-debugging-benchmarking/01-the-test-pyramid.md)) и тестовые двойники ([Урок 09.3](../09-testing-debugging-benchmarking/03-test-doubles.md)).
- Вы знаете обработчики и экстракторы axum ([Уроки 16.3](03-web-services-with-axum.md) и [16.4](04-errors-validation-and-middleware.md)), sqlx ([16.6](06-databases-with-sqlx.md)) и миграции ([16.7](07-migrations-and-transactions.md)).
- Docker запущен.

## Новые слова

| Слово | Значение |
|---|---|
| **тест внутри процесса** (in-process test) | Тест, который вызывает код приложения напрямую, в том же процессе, а не через настоящее сетевое соединение. |
| **`tower::Service`** | Трейт, на котором построен axum: «нечто, что принимает запрос и возвращает future с ответом». `Router` из axum — это `Service`. |
| **`oneshot`** | Метод из `tower::ServiceExt`: отправить сервису ровно один запрос и получить ответ. |
| **testcontainers** | Библиотека, которая запускает контейнеры Docker из кода теста и удаляет их, когда тест заканчивается. |
| **тестовые данные** (test fixture) | Всё, что нужно подготовить перед запуском теста: база данных, таблицы, несколько строк. |
| **изоляция тестов** (test isolation) | У каждого теста свои данные, поэтому тесты не могут менять результаты друг друга, даже когда идут параллельно. |
| **`#[sqlx::test]`** | Атрибут из sqlx, который создаёт новую базу данных для одного теста, применяет в ней ваши миграции и передаёт тесту пул. |
| **нестабильный тест** (flaky test) | Тест, который иногда проходит, а иногда падает без изменений кода (Урок 05.6). Обычные причины — общие данные и тайминги. |

## 1. Что тестировать в сервисе

Урок 09.1 познакомил вас с пирамидой: много маленьких быстрых тестов внизу, мало больших медленных наверху. Для бэкенд-сервиса уровни выглядят так:

| Уровень | Что проверяет | Что нужно | Скорость |
|---|---|---|---|
| **Модульные тесты** | Чистая логика: проверка ввода, разбор, вычисления, конфигурация (Урок 16.13) | Ничего | Микросекунды |
| **Тесты обработчиков внутри процесса** | Маршрутизация, экстракторы, коды статуса, форма JSON (JavaScript Object Notation) | `Router`, возможно фейковый репозиторий | Миллисекунды |
| **Интеграционные тесты с настоящей базой** | Запросы SQL (Structured Query Language, язык структурированных запросов), миграции, ограничения, транзакции | PostgreSQL (в Docker) | Десятки миллисекунд на тест плюс запуск |
| **Сквозные тесты** | Настоящий исполняемый файл через сеть со всеми зависимостями | Всё запущено | Секунды |

Большая часть урока — о двух средних уровнях, потому что именно там сервисы ломаются чаще всего: неверный код статуса, запрос, который падает на настоящих данных, миграция, которая не выполняется. Мок базы данных спрятал бы как раз эти ошибки, поэтому мы тестируем с **настоящим** PostgreSQL, запускаемым автоматически.

## 2. Тестируемый сервис

Небольшой API заметок (API, Application Programming Interface, программный интерфейс): `POST /notes` создаёт заметку, `GET /notes/{id}` читает одну. Создайте его:

```bash
cd ~/projects/rust/code
cargo new p16-tested-notes
cd p16-tested-notes
cargo add axum@0.8 serde_json@1
cargo add serde@1 --features derive
cargo add sqlx@0.9 --features postgres,runtime-tokio
cargo add tokio@1 --features macros,rt-multi-thread
cargo add --dev tower@0.5 --features util
cargo add --dev testcontainers@0.28
mkdir migrations tests
```

Миграция, `migrations/20261118000000_create_notes.sql`:

```sql
CREATE TABLE notes (
    id BIGSERIAL PRIMARY KEY,
    text TEXT NOT NULL
);
```

Приложение живёт в `src/lib.rs`, чтобы тесты могли его собрать:

```rust
use axum::extract::{Path, State};
use axum::http::StatusCode;
use axum::routing::{get, post};
use axum::{Json, Router};
use serde::{Deserialize, Serialize};
use sqlx::PgPool;

#[derive(Debug, PartialEq, Serialize, Deserialize, sqlx::FromRow)]
pub struct Note {
    pub id: i64,
    pub text: String,
}

#[derive(Deserialize)]
pub struct NewNote {
    pub text: String,
}

/// Собирает всё HTTP-приложение. Тесты тоже вызывают эту функцию.
pub fn app(pool: PgPool) -> Router {
    Router::new()
        .route("/notes", post(create_note))
        .route("/notes/{id}", get(get_note))
        .with_state(pool)
}

async fn create_note(
    State(pool): State<PgPool>,
    Json(new): Json<NewNote>,
) -> Result<(StatusCode, Json<Note>), StatusCode> {
    if new.text.trim().is_empty() {
        return Err(StatusCode::UNPROCESSABLE_ENTITY);
    }
    let note = sqlx::query_as::<_, Note>("INSERT INTO notes (text) VALUES ($1) RETURNING id, text")
        .bind(&new.text)
        .fetch_one(&pool)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok((StatusCode::CREATED, Json(note)))
}

async fn get_note(
    State(pool): State<PgPool>,
    Path(id): Path<i64>,
) -> Result<Json<Note>, StatusCode> {
    let note = sqlx::query_as::<_, Note>("SELECT id, text FROM notes WHERE id = $1")
        .bind(id)
        .fetch_optional(&pool)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    note.map(Json).ok_or(StatusCode::NOT_FOUND)
}
```

И короткий `src/main.rs`, который подключается, применяет миграции и обслуживает запросы:

```rust
use sqlx::PgPool;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let url = std::env::var("DATABASE_URL")?;
    let pool = PgPool::connect(&url).await?;
    sqlx::migrate!().run(&pool).await?;

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await?;
    println!("listening on {}", listener.local_addr()?);
    axum::serve(listener, p16_tested_notes::app(pool)).await?;
    Ok(())
}
```

**Что происходит:**

1. Главное решение — `pub fn app(pool: PgPool) -> Router`. `main` и тесты собирают **один и тот же** роутер; отличается только пул. Всё остальное, что делает `main` (читает окружение, занимает порт), невелико и в тестах такого рода не нуждается.
2. Мы используем запросы, проверяемые во время выполнения (`query_as::<_, Note>`), с `#[derive(sqlx::FromRow)]`, чтобы проект компилировался без базы. Макросы `query!` с проверкой при компиляции из Урока 16.6 работают в тестах так же.
3. Обработчики возвращают простые ошибки `StatusCode`, чтобы пример был коротким; настоящий сервис использует тип ошибки из Урока 16.4.
4. `sqlx::migrate!()` встраивает файлы из `migrations/` в исполняемый файл при компиляции (Урок 16.7).

## 3. Вызов роутера без сети

`Router` из axum реализует трейт `tower::Service`: он принимает HTTP-запрос `Request` и возвращает future, который даёт `Response`. Трейт-расширение `tower::ServiceExt` добавляет `oneshot`, который отправляет **один** запрос и ждёт ответа. Никакого порта, никакого соединения TCP (Transmission Control Protocol, протокол управления передачей), никакого `reqwest`: тест вызывает роутер так же, как это делал бы сервер.

В тестовом файле нужны два маленьких помощника: `send` выполняет запрос и возвращает статус и тело как текст; `post_json` строит JSON-запрос. Вы увидите их в полном `tests/api.rs` в разделе 4; вот `send`:

```rust
/// Отправляет приложению один запрос, без всякой сети, и возвращает статус и тело.
async fn send(app: &Router, request: Request<Body>) -> (StatusCode, String) {
    let response = app.clone().oneshot(request).await.unwrap();
    let status = response.status();
    let bytes = axum::body::to_bytes(response.into_body(), usize::MAX)
        .await
        .unwrap();
    (status, String::from_utf8(bytes.to_vec()).unwrap())
}
```

**Что происходит:**

1. `oneshot` принимает сервис **по значению** (поглощает его), поэтому мы передаём клон. Клонировать `Router` дёшево: его маршруты общие.
2. В `post_json` (раздел 4) `Request::post(uri)` — построитель запросов из крейта `http` (реэкспортирован как `axum::http`). Не забудьте заголовок `content-type`: экстрактор `Json` из axum отвергает тело без него (Упражнение 1).
3. Тело ответа — поток байтов; `axum::body::to_bytes` собирает его целиком. `usize::MAX` означает «без ограничения размера», что в тесте допустимо.

## 4. Настоящий PostgreSQL для каждого теста

Обработчикам нужна база. **testcontainers** позволяет тесту запустить её в Docker, как `docker run`, и удаляет контейнер, когда тест закончился. Вот полный `tests/api.rs`:

```rust
use std::time::Duration;

use axum::Router;
use axum::body::Body;
use axum::http::{Request, StatusCode};
use p16_tested_notes::{Note, app};
use sqlx::PgPool;
use testcontainers::core::{IntoContainerPort, WaitFor};
use testcontainers::runners::AsyncRunner;
use testcontainers::{ContainerAsync, GenericImage, ImageExt};
use tower::ServiceExt; // ради `oneshot`

/// Запускает свежий PostgreSQL в Docker и возвращает приложение, подключённое к нему.
/// Держите значение контейнера живым: его удаление удаляет контейнер.
async fn test_app() -> (Router, ContainerAsync<GenericImage>) {
    let container = GenericImage::new("postgres", "18")
        .with_exposed_port(5432.tcp())
        .with_wait_for(WaitFor::message_on_stderr(
            "database system is ready to accept connections",
        ))
        .with_env_var("POSTGRES_PASSWORD", "test")
        .start()
        .await
        .expect("Docker must be running");
    let port = container.get_host_port_ipv4(5432).await.unwrap();
    let url = format!("postgres://postgres:test@127.0.0.1:{port}/postgres");

    // После первого запуска сервер может один раз перезапуститься, поэтому недолго повторяем.
    let mut attempts = 0;
    let pool = loop {
        match PgPool::connect(&url).await {
            Ok(pool) => break pool,
            Err(_) if attempts < 20 => {
                attempts += 1;
                tokio::time::sleep(Duration::from_millis(250)).await;
            }
            Err(err) => panic!("cannot connect to the test database: {err}"),
        }
    };
    sqlx::migrate!().run(&pool).await.unwrap();
    (app(pool), container)
}

/// Отправляет приложению один запрос, без всякой сети, и возвращает статус и тело.
async fn send(app: &Router, request: Request<Body>) -> (StatusCode, String) {
    let response = app.clone().oneshot(request).await.unwrap();
    let status = response.status();
    let bytes = axum::body::to_bytes(response.into_body(), usize::MAX)
        .await
        .unwrap();
    (status, String::from_utf8(bytes.to_vec()).unwrap())
}

fn post_json(uri: &str, json: &str) -> Request<Body> {
    Request::post(uri)
        .header("content-type", "application/json")
        .body(Body::from(json.to_string()))
        .unwrap()
}

#[tokio::test]
async fn created_note_can_be_read_back() {
    let (app, _db) = test_app().await;

    let (status, body) = send(&app, post_json("/notes", r#"{"text":"buy milk"}"#)).await;
    assert_eq!(status, StatusCode::CREATED);
    let created: Note = serde_json::from_str(&body).unwrap();
    assert_eq!(created.text, "buy milk");

    let uri = format!("/notes/{}", created.id);
    let (status, body) = send(&app, Request::get(uri).body(Body::empty()).unwrap()).await;
    assert_eq!(status, StatusCode::OK);
    assert_eq!(serde_json::from_str::<Note>(&body).unwrap(), created);
}

#[tokio::test]
async fn unknown_note_is_404() {
    let (app, _db) = test_app().await;
    let request = Request::get("/notes/999").body(Body::empty()).unwrap();
    let (status, _) = send(&app, request).await;
    assert_eq!(status, StatusCode::NOT_FOUND);
}
```

```bash
cargo test --test api
```

```text
running 2 tests
test unknown_note_is_404 ... ok
test created_note_can_be_read_back ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.55s
```

**Что происходит:**

1. `GenericImage::new("postgres", "18")` описывает официальный образ, как `postgres:18` в командной строке. `with_env_var` задаёт пароль, как это сделал бы `-e`.
2. `with_exposed_port(5432.tcp())` публикует порт 5432 контейнера на **случайном свободном порту** вашего Mac; `get_host_port_ipv4(5432)` говорит, на каком. Поэтому тесты никогда не сталкиваются на порту, и ничто другое на вашей машине не страдает.
3. `with_wait_for(...)` заставляет `start()` ждать, пока PostgreSQL не напечатает, что принимает соединения. Официальный образ при первой настройке запускает временный сервер, а потом перезапускается, поэтому соединение ещё может на мгновение не удаться; это покрывает короткий цикл повторов.
4. `sqlx::migrate!().run(&pool)` создаёт таблицы. Тест работает с той же схемой, что и продакшен, потому что использует те же файлы миграций.
5. `test_app` возвращает контейнер вместе с роутером. Тест держит его в `_db`. Когда тест заканчивается, `_db` удаляется, и testcontainers удаляет контейнер. Имя, начинающееся с `_`, убирает предупреждение «неиспользуемая переменная», но всё равно держит значение живым до конца области видимости.
6. Каждый тест получает **свой** контейнер, поэтому тесты изолированы и могут идти параллельно. Первый запуск медленнее, потому что Docker скачивает образ; после этого два теста заняли вместе 1,55 с.

> 🧪 **Тестирование:** посмотрите, что проверяет первый тест: код статуса, форму JSON (через десериализацию в `Note`) и то, что чтение возвращает ровно то, что было создано. Он не проверяет точное значение id: id берутся из последовательности базы, и тест, ожидающий `id == 1`, может сломаться, когда тесты делят базу.

## 5. Быстрее: один сервер, новая база на каждый тест

Контейнер на тест стоит около секунды. Если у вас сотни тестов с базой, используйте **`#[sqlx::test]`**: вы сами запускаете **один** сервер PostgreSQL (или его запускает CI, Continuous Integration, непрерывная интеграция), а sqlx для каждого теста создаёт на нём свежую базу, применяет миграции из `migrations/`, даёт тесту `PgPool` и потом удаляет базу. Создайте `tests/with_sqlx_test.rs`:

```rust
use axum::body::Body;
use axum::http::{Request, StatusCode};
use sqlx::PgPool;
use tower::ServiceExt;

/// `#[sqlx::test]` создаёт пустую базу для этого теста, применяет в ней
/// `migrations/`, передаёт пул и потом удаляет базу.
#[sqlx::test]
async fn empty_text_is_rejected(pool: PgPool) {
    let app = p16_tested_notes::app(pool);
    let request = Request::post("/notes")
        .header("content-type", "application/json")
        .body(Body::from(r#"{"text":"   "}"#))
        .unwrap();

    let response = app.oneshot(request).await.unwrap();

    assert_eq!(response.status(), StatusCode::UNPROCESSABLE_ENTITY);
}

#[sqlx::test]
async fn ids_start_at_one_in_every_test_database(pool: PgPool) {
    let id: i64 = sqlx::query_scalar("INSERT INTO notes (text) VALUES ('a') RETURNING id")
        .fetch_one(&pool)
        .await
        .unwrap();
    assert_eq!(id, 1);
}
```

Запустите один сервер и скажите тестам, где он, через `DATABASE_URL`:

```bash
docker run -d --name p16-test-db -e POSTGRES_PASSWORD=test -p 127.0.0.1:5433:5432 postgres:18
DATABASE_URL=postgres://postgres:test@127.0.0.1:5433/postgres cargo test --test with_sqlx_test
```

```text
running 2 tests
test empty_text_is_rejected ... ok
test ids_start_at_one_in_every_test_database ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.23s
```

**Что происходит:**

1. Атрибут заменяет `#[tokio::test]` и добавляет аргумент `pool`. Пользователю из `DATABASE_URL` должно быть разрешено создавать базы (пользователю `postgres` из образа разрешено).
2. Второй тест показывает изоляцию: в каждой тестовой базе первый id равен 1, сколько бы тестов ни прошло до этого, потому что у каждого теста свои свежие таблицы.
3. Два теста вместе заняли 0,23 с вместо 1,55 с, потому что не пришлось запускать контейнер.
4. Порт 5433 позволяет не столкнуться с PostgreSQL, который у вас, возможно, уже работает на 5432 после предыдущих уроков. Когда закончите, удалите сервер командой `docker rm -f p16-test-db`.

Без `DATABASE_URL` каждый такой тест сразу падает (сокращено):

```text
thread 'empty_text_is_rejected' (6019166) panicked at .../sqlx-postgres-0.9.1/src/testing/mod.rs:93:44:
DATABASE_URL must be set: EnvVar(NotPresent)
```

Что выбрать?

| Инструмент | Для чего хорош | Цена |
|---|---|---|
| testcontainers | Самодостаточные тесты: `cargo test` нужен только Docker; любой образ (Redis, NATS, …) | ~1 с на запуск каждого контейнера |
| `#[sqlx::test]` | Много тестов с PostgreSQL, быстро | Кто-то должен запустить сервер и задать `DATABASE_URL` |
| Фейковый репозиторий (Урок 09.3) | Логика обработчиков без всякой базы | Совсем не проверяет SQL |

Многие проекты сочетают их: `#[sqlx::test]` для множества тестов с базой, где сервер запускает помощник на testcontainers или конвейер CI (Урок 09.12), и фейки там, где база не главное.

> 🔒 **Безопасность:** тестовые учётные данные вроде `POSTGRES_PASSWORD=test` допустимы, потому что контейнеры слушают только вашу машину и живут только во время теста. Никогда не направляйте `DATABASE_URL` для тестов на общую или продакшен-базу: `#[sqlx::test]` создаёт и удаляет там базы данных.

## Частые ошибки

> ⚠️ **Частая ошибка:** написать `let (app, _) = test_app().await;`. Простое `_` — не переменная: контейнер удаляется **сразу**, и каждый запрос затем падает. Тест прождал 30-секундный тайм-аут пула и упал:

```text
thread 'unknown_note_is_404' (6038758) panicked at tests/api.rs:81:5:
assertion `left == right` failed
  left: 500
```

Исправление: свяжите его с именем, `_db`, чтобы он жил до конца теста.

> ⚠️ **Частая ошибка:** Docker не запущен. testcontainers ничего не может запустить, а `expect("Docker must be running")` делает причину понятной. Здесь `DOCKER_HOST` указывал на несуществующий сокет:

```text
thread 'unknown_note_is_404' (6050912) panicked at tests/api.rs:24:10:
Docker must be running: Client(Init(SocketNotFoundError("/tmp/no-docker-here.sock")))
```

> ⚠️ **Частая ошибка:** тесты, которые делят одну базу и ждут точных данных, например «заметок ровно 2». По отдельности они проходят, а вместе или в другом порядке падают: **нестабильные тесты**. Дайте каждому тесту свою базу или проверяйте только данные, которые создал сам тест.

> ⚠️ **Частая ошибка:** тестировать только удачный путь. Большинство аварий сервисов в продакшене — на путях ошибок: неверный ввод, отсутствующие строки, нарушения ограничений. Каждый обработчик заслуживает хотя бы одного теста на каждый код статуса, который он может вернуть.

## Попробуйте сами

1. Запустите `cargo test` (все тестовые файлы), когда сервер из раздела 5 работает и `DATABASE_URL` задан. Посчитайте тесты: их четыре.
2. Пока выполняется `cargo test --test api`, посмотрите `docker ps` в другом терминале: вы увидите короткоживущие контейнеры `postgres:18` со случайными именами, которые исчезают, когда тесты заканчиваются.
3. Уберите из `test_app` строку `sqlx::migrate!().run(&pool).await.unwrap();` и снова запустите тесты. Теперь оба теста падают с `left: 500`, потому что таблицы нет. Верните строку.

## Упражнения

### Упражнение 1 — Что отвергает `Json`? (легко)

Предскажите код статуса для трёх плохих запросов к `POST /notes`, затем напишите один тест `#[sqlx::test]`, который проверяет все три: (a) правильное тело JSON, но **без** заголовка `content-type`; (b) `content-type: application/json` со сломанным телом `{"text":`; (c) корректный JSON с неверным полем, `{"title":"a"}`.

<details>
<summary>Подсказка</summary>

Эти запросы вообще не доходят до кода вашего обработчика: их раньше отвергает экстрактор `Json` (Урок 16.4). Небольшой помощник `status_for(pool, request)`, возвращающий только статус, делает тест коротким.

</details>

<details>
<summary>Ответ</summary>

(a) `415 Unsupported Media Type`, (b) `400 Bad Request`, (c) `422 Unprocessable Entity`. Положите тест в `tests/exercises.rs`:

```rust
use axum::body::Body;
use axum::http::{Request, StatusCode};
use p16_tested_notes::{Note, app};
use sqlx::PgPool;
use tower::ServiceExt;

async fn status_for(pool: PgPool, request: Request<Body>) -> StatusCode {
    app(pool).oneshot(request).await.unwrap().status()
}

#[sqlx::test]
async fn bad_bodies_are_rejected_by_the_json_extractor(pool: PgPool) {
    let no_type = Request::post("/notes")
        .body(Body::from(r#"{"text":"a"}"#))
        .unwrap();
    assert_eq!(
        status_for(pool.clone(), no_type).await,
        StatusCode::UNSUPPORTED_MEDIA_TYPE
    );

    let broken = Request::post("/notes")
        .header("content-type", "application/json")
        .body(Body::from(r#"{"text":"#))
        .unwrap();
    assert_eq!(
        status_for(pool.clone(), broken).await,
        StatusCode::BAD_REQUEST
    );

    let wrong_field = Request::post("/notes")
        .header("content-type", "application/json")
        .body(Body::from(r#"{"title":"a"}"#))
        .unwrap();
    assert_eq!(
        status_for(pool, wrong_field).await,
        StatusCode::UNPROCESSABLE_ENTITY
    );
}
```

(`Note` используется в следующем упражнении; если делаете только это, уберите его из строки `use`.)

Почему это работает: `Json` из axum сначала проверяет заголовок (415), потом синтаксис JSON (400), потом подходят ли данные к `NewNote` (422). Знание этих кодов помогает писать верную документацию API, а тест гарантирует, что обновление axum не изменит их незаметно.

</details>

### Упражнение 2 — Список всех заметок (средне)

Добавьте `GET /notes`, который возвращает все заметки по порядку id как массив JSON. Напишите тест `#[sqlx::test]`, который вставляет две заметки напрямую через SQL и проверяет, что эндпоинт возвращает их тексты по порядку.

<details>
<summary>Подсказка</summary>

У маршрута может быть несколько методов: `.route("/notes", post(create_note).get(list_notes))`. Используйте `fetch_all` и возвращайте `Json<Vec<Note>>`. В тесте десериализуйте тело через `serde_json::from_slice::<Vec<Note>>`.

</details>

<details>
<summary>Ответ</summary>

В `src/lib.rs`:

```rust
async fn list_notes(State(pool): State<PgPool>) -> Result<Json<Vec<Note>>, StatusCode> {
    let notes = sqlx::query_as::<_, Note>("SELECT id, text FROM notes ORDER BY id")
        .fetch_all(&pool)
        .await
        .map_err(|_| StatusCode::INTERNAL_SERVER_ERROR)?;
    Ok(Json(notes))
}
```

и маршрут `.route("/notes", post(create_note).get(list_notes))`. Тест:

```rust
#[sqlx::test]
async fn list_returns_all_notes_in_order(pool: PgPool) {
    for text in ["first", "second"] {
        sqlx::query("INSERT INTO notes (text) VALUES ($1)")
            .bind(text)
            .execute(&pool)
            .await
            .unwrap();
    }
    let response = app(pool)
        .oneshot(Request::get("/notes").body(Body::empty()).unwrap())
        .await
        .unwrap();
    assert_eq!(response.status(), StatusCode::OK);
    let bytes = axum::body::to_bytes(response.into_body(), usize::MAX)
        .await
        .unwrap();
    let notes: Vec<Note> = serde_json::from_slice(&bytes).unwrap();
    let texts: Vec<&str> = notes.iter().map(|n| n.text.as_str()).collect();
    assert_eq!(texts, ["first", "second"]);
}
```

Почему это работает: вставка тестовых строк через SQL держит тест сосредоточенным на эндпоинте списка (он не зависит от того, работает ли `POST`). `ORDER BY id` делает порядок определённым; без него PostgreSQL может вернуть строки в любом порядке, и тест в принципе был бы нестабильным.

</details>

### Упражнение 3 — Правило базы данных становится 422 (сложно)

Добавьте вторую миграцию, которая ограничивает заметки 200 символами с помощью ограничения `CHECK`. Сейчас более длинная заметка заставила бы `INSERT` упасть, а обработчик — вернуть `500`. Измените обработчик так, чтобы нарушение проверки превращалось в `422`, и сначала напишите тест (до исправления он должен падать с 500).

<details>
<summary>Подсказка</summary>

Миграция: `ALTER TABLE notes ADD CONSTRAINT text_not_too_long CHECK (length(text) <= 200);`. В `map_err` сопоставьте `sqlx::Error::Database(db)` и спросите `db.is_check_violation()`. Длинный текст постройте через `"x".repeat(201)` и `serde_json::json!`.

</details>

<details>
<summary>Ответ</summary>

`migrations/20261118000100_limit_note_length.sql`:

```sql
ALTER TABLE notes ADD CONSTRAINT text_not_too_long CHECK (length(text) <= 200);
```

В `create_note` замените `map_err` у `INSERT`:

```rust
        .map_err(|err| match err {
            sqlx::Error::Database(db) if db.is_check_violation() => {
                StatusCode::UNPROCESSABLE_ENTITY
            }
            _ => StatusCode::INTERNAL_SERVER_ERROR,
        })?;
```

Тест:

```rust
#[sqlx::test]
async fn too_long_text_is_422_not_500(pool: PgPool) {
    let json = serde_json::json!({ "text": "x".repeat(201) }).to_string();
    let request = Request::post("/notes")
        .header("content-type", "application/json")
        .body(Body::from(json))
        .unwrap();
    assert_eq!(
        status_for(pool, request).await,
        StatusCode::UNPROCESSABLE_ENTITY
    );
}
```

Почему это работает: ограничение делает базу последним стражем правила, даже для данных, пришедших не через этот обработчик. PostgreSQL сообщает класс ошибки `23514` (нарушение проверки); sqlx превращает его в `is_check_violation() == true`, а обработчик отображает это в ошибку клиента вместо «сервер упал». Поймать это может только тест с настоящей базой; фейковый репозиторий ничего не знает об ограничениях.

</details>

## Вопросы для самопроверки

1. Почему урок тестирует с настоящим PostgreSQL, а не с мок-базой?

<details>
<summary>Ответ</summary>

Многие ошибки сервисов живут в SQL, миграциях и ограничениях. Мок принял бы любой запрос и спрятал их. Настоящая база в контейнере всё равно быстрая и полностью автоматическая.

</details>

2. Что делает `oneshot` и почему мы вызываем его на `app.clone()`?

<details>
<summary>Ответ</summary>

Он отправляет один запрос в `tower::Service` (наш `Router`) и возвращает ответ, без всякой сети. Он поглощает сервис, поэтому мы передаём клон; клонировать `Router` дёшево.

</details>

3. Почему `test_app` возвращает контейнер и что будет, если тест его не сохранит?

<details>
<summary>Ответ</summary>

Удаление значения `ContainerAsync` удаляет контейнер. Если тест его не сохранит (например, через `let (app, _) = ...`), база сразу исчезнет, и каждый запрос будет падать.

</details>

4. Как `#[sqlx::test]` изолирует тесты и что ему нужно?

<details>
<summary>Ответ</summary>

Он создаёт новую базу для каждого теста на сервере из `DATABASE_URL`, применяет там миграции и затем удаляет её. Ему нужен работающий сервер и пользователь с правом создавать базы.

</details>

5. Тест ожидает `id == 1` и проходит в одиночку, но падает вместе с другими тестами. Что не так?

<details>
<summary>Ответ</summary>

Тесты делят базу, поэтому другой тест уже использовал id 1: нестабильный тест из-за общих данных. Дайте каждому тесту свою базу или не проверяйте значения, зависящие от других тестов.

</details>

## Итоги

- Тестируйте сервис на нескольких уровнях: чистую логику — модульными тестами, маршрутизацию и коды статуса — внутри процесса, SQL — с настоящей базой, и немного сквозных тестов.
- Собирайте роутер в одной функции, `app(pool)`, которую используют и `main`, и тесты.
- `tower::ServiceExt::oneshot` отправляет запрос в роутер без сети; тело собирайте через `axum::body::to_bytes`.
- testcontainers запускает настоящие контейнеры из теста на случайных портах и удаляет их при удалении значения; держите значение контейнера живым.
- `#[sqlx::test]` даёт каждому тесту свежую базу с миграциями на одном сервере: быстро и изолированно.
- Изолированные тесты проверяют только свои данные; тестируйте пути ошибок, а не только удачный путь.

## Источники и что почитать

- [axum — testing example](https://github.com/tokio-rs/axum/tree/main/examples/testing) — официальные примеры тестов с `oneshot` и тестов через настоящий сокет.
- [tower::ServiceExt на docs.rs](https://docs.rs/tower/latest/tower/trait.ServiceExt.html) — `oneshot` и другие вспомогательные методы.
- [testcontainers for Rust](https://rust.testcontainers.org/) — официальное руководство: образы, условия ожидания, порты, очистка.
- [sqlx::test на docs.rs](https://docs.rs/sqlx/latest/sqlx/attr.test.html) — все параметры атрибута, включая тестовые данные и свои миграции.
- [PostgreSQL — Error codes (Appendix A)](https://www.postgresql.org/docs/current/errcodes-appendix.html) — классы ошибок, такие как `23514` (нарушение проверки).

---

[🏠 Начало](../START-HERE.md) · [← 16.13 Конфигурация, проверки работоспособности и контейнеры](13-configuration-health-checks-and-containers.md) · [📋 Этап 16](README.md) · [16.15 Проект: сервис коротких ссылок →](15-project-url-shortener-service.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
