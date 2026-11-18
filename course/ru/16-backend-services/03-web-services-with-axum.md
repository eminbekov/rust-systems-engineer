<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 16.2 Проектирование HTTP API: REST](02-designing-http-apis.md) · [📋 Этап 16](README.md) · [16.4 Ошибки, проверка ввода и middleware →](04-errors-validation-and-middleware.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/16-backend-services/03-web-services-with-axum.md) <!-- nav -->

# Урок 16.3 — Веб-сервисы на axum

> **Этап 16 — Бэкенд-сервисы** · Урок 3 из 16 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как построить API (Application Programming Interface, программный интерфейс) задач из Урока 16.2 на `axum`: маршруты, обработчики и методы `get`, `post`,
  `patch`, `delete`.
- Как *экстракторы* (`Path`, `Query`, `Json`, `State`) превращают части запроса в типизированные значения Rust.
- Как обработчик строит ответ: код статуса, заголовки и JSON-тело (JavaScript Object Notation, нотация объектов JavaScript).
- Как разделять данные между запросами через `State` и `Arc<Mutex<…>>`.
- Как тестировать весь маршрутизатор в памяти, не открывая порт.

## Перед началом

- Вы прошли [Урок 16.2 — Проектирование HTTP API: REST](02-designing-http-apis.md). Держите открытой его
  таблицу контракта: этот урок её реализует.
- Вы знаете `Arc` и `Mutex` ([Урок 05.10 — Потоки, каналы и общее состояние](../05-rust-intermediate/10-threads-channels-and-shared-state.md))
  и почему блокировку нельзя держать через `.await` ([Урок 06.9](../06-rust-advanced/09-async-await-introduction.md), раздел 8).
- Вы знаете derive-макросы `serde` из [Урока 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md).

## Новые слова

| Слово | Значение |
|---|---|
| **экстрактор** (extractor) | Тип в списке параметров обработчика, который axum заполняет из запроса: `Path` — из пути, `Query` — из строки запроса, `Json` — из тела, `State` — из общего состояния. |
| **отказ** (rejection) | Ответ с ошибкой, который axum отправляет сам, когда экстрактор не справился, например `400` для сломанного JSON. Ваш обработчик не выполняется. |
| **`IntoResponse`** | Трейт axum «можно превратить в HTTP-ответ (HyperText Transfer Protocol, протокол передачи гипертекста)». Обработчики возвращают типы, которые его реализуют: текст, `Json`, `StatusCode`, кортежи, `Result`. |
| **состояние приложения** (application state) | Данные, общие для всех запросов, например хранилище задач или (позже) пул соединений с базой данных. В axum его подключают через `.with_state(...)` и читают через `State`. |
| **tower** | Крейт, который определяет `Service`: «нечто, что принимает запрос и асинхронно возвращает ответ». `Router` в axum — это `Service` из tower. |
| **`oneshot`** | Вспомогательный метод из tower, который отправляет сервису ровно один запрос и возвращает его ответ. Тесты используют его, чтобы вызвать маршрутизатор без сети. |

## 1. Пакет

Сервис задач живёт в своём пакете, который вы будете выращивать до Урока 16.8:

```bash
cd ~/projects/rust
cargo new code/p16-tasks
cd code/p16-tasks
cargo add axum@0.8
cargo add tokio@1 --features full
cargo add serde@1 --features derive
cargo add --dev tower@0.5 --features util
cargo add --dev http-body-util@0.1
```

Теперь в `Cargo.toml` есть:

```toml
[dependencies]
axum = "0.8"
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }

[dev-dependencies]
http-body-util = "0.1"
tower = { version = "0.5", features = ["util"] }
```

`tower` и `http-body-util` нужны только тестам (раздел 8), поэтому это **dev-зависимости**. Перенесите в эту
папку `API.md`, который вы написали в Уроке 16.2.

## 2. Весь сервис

Замените `src/main.rs` следующей программой. Прочитайте её один раз сверху вниз; следующие разделы объясняют
каждую часть.

```rust
use std::collections::BTreeMap;
use std::sync::{Arc, Mutex};

use axum::{
    Json, Router,
    extract::{Path, Query, State},
    http::{StatusCode, header},
    response::IntoResponse,
    routing::get,
};
use serde::{Deserialize, Serialize};

#[derive(Debug, Clone, Serialize)]
struct Task {
    id: u64,
    title: String,
    done: bool,
}

#[derive(Debug, Deserialize)]
struct NewTask {
    title: String,
}

#[derive(Debug, Deserialize)]
struct TaskChanges {
    title: Option<String>,
    done: Option<bool>,
}

#[derive(Debug, Deserialize)]
struct ListFilter {
    done: Option<bool>,
}

#[derive(Debug, Default)]
struct Store {
    last_id: u64,
    tasks: BTreeMap<u64, Task>,
}

type AppState = Arc<Mutex<Store>>;

async fn list_tasks(
    State(state): State<AppState>,
    Query(filter): Query<ListFilter>,
) -> Json<Vec<Task>> {
    let store = state.lock().unwrap();
    let tasks = store
        .tasks
        .values()
        .filter(|task| filter.done.is_none_or(|done| task.done == done))
        .cloned()
        .collect();
    Json(tasks)
}

async fn create_task(
    State(state): State<AppState>,
    Json(new_task): Json<NewTask>,
) -> impl IntoResponse {
    let mut store = state.lock().unwrap();
    store.last_id += 1;
    let task = Task {
        id: store.last_id,
        title: new_task.title,
        done: false,
    };
    store.tasks.insert(task.id, task.clone());
    let location = format!("/tasks/{}", task.id);
    (StatusCode::CREATED, [(header::LOCATION, location)], Json(task))
}

async fn get_task(
    State(state): State<AppState>,
    Path(id): Path<u64>,
) -> Result<Json<Task>, StatusCode> {
    let store = state.lock().unwrap();
    match store.tasks.get(&id) {
        Some(task) => Ok(Json(task.clone())),
        None => Err(StatusCode::NOT_FOUND),
    }
}

async fn update_task(
    State(state): State<AppState>,
    Path(id): Path<u64>,
    Json(changes): Json<TaskChanges>,
) -> Result<Json<Task>, StatusCode> {
    let mut store = state.lock().unwrap();
    let task = store.tasks.get_mut(&id).ok_or(StatusCode::NOT_FOUND)?;
    if let Some(title) = changes.title {
        task.title = title;
    }
    if let Some(done) = changes.done {
        task.done = done;
    }
    Ok(Json(task.clone()))
}

async fn delete_task(State(state): State<AppState>, Path(id): Path<u64>) -> StatusCode {
    let mut store = state.lock().unwrap();
    match store.tasks.remove(&id) {
        Some(_) => StatusCode::NO_CONTENT,
        None => StatusCode::NOT_FOUND,
    }
}

async fn health() -> &'static str {
    "ok\n"
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
}

#[tokio::main]
async fn main() {
    let state = AppState::default();
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on http://127.0.0.1:3000");
    axum::serve(listener, app(state)).await.unwrap();
}
```

**Что происходит:**

1. `Task` — то, что отправляет API (`Serialize`). `NewTask` — тело `POST /tasks`, `TaskChanges` — тело
   `PATCH` (оба `Deserialize`). В `TaskChanges` оба поля — `Option`: отсутствующее поле становится `None`, что
   означает «не менять».
2. `ListFilter` — строка запроса `GET /tasks`: `?done=false` превращается в `ListFilter { done: Some(false) }`,
   отсутствие строки запроса — в `ListFilter { done: None }`.
3. `Store` хранит задачи в `BTreeMap`, который держит ключи отсортированными, поэтому список выходит от старых к
   новым, как требует контракт. `last_id` даёт каждой новой задаче следующий номер.
4. `AppState` — короткое имя для `Arc<Mutex<Store>>`: одно хранилище, общее для всех запросов.
5. На каждую строку контракта приходится один обработчик. `app` соединяет их: `get(list_tasks).post(create_task)`
   вешает два метода на один путь. `{id}` в `"/tasks/{id}"` — параметр пути из Урока 16.2.
6. `app` — отдельная функция, чтобы `main` и тесты (раздел 8) строили ровно один и тот же маршрутизатор.

Запустите через `cargo run`; программа печатает `listening on http://127.0.0.1:3000`. Раздел 6 с ней общается.

## 3. Экстракторы: от запроса к типизированным значениям

Посмотрите на параметры обработчика:

```rust
async fn update_task(
    State(state): State<AppState>,
    Path(id): Path<u64>,
    Json(changes): Json<TaskChanges>,
) -> Result<Json<Task>, StatusCode>
```

Вы ничего не разбираете сами. Тип каждого параметра — **экстрактор**, и axum заполняет его до запуска вашего
обработчика:

| Экстрактор | Что берёт из запроса | Пример |
|---|---|---|
| `Path<u64>` | часть `{id}` пути, разобранную как `u64` | `/tasks/7` → `7` |
| `Query<ListFilter>` | строку запроса, десериализованную через serde | `?done=false` → `ListFilter { done: Some(false) }` |
| `Json<TaskChanges>` | тело, проверенное на `content-type: application/json` и затем десериализованное | `{"done":true}` → `TaskChanges { title: None, done: Some(true) }` |
| `State<AppState>` | общее состояние, переданное в `.with_state(...)` | `Arc<Mutex<Store>>` |

`State(state): State<AppState>` — это образец (Урок 06.3): он распаковывает структуру-обёртку `State`, так что
переменная `state` — это `AppState` внутри неё.

Если экстрактор не может сделать свою работу (в пути `abc` вместо числа, JSON сломан), axum сразу отправляет
ответ-**отказ**, и ваш обработчик не выполняется. Это большой выигрыш: внутри обработчика `id` — **всегда**
корректный `u64`. Раздел 7 показывает отказы.

> ⚠️ **Частая ошибка:** тело можно прочитать только один раз, поэтому экстрактор, который его читает (`Json`,
> `String`, `Bytes`), должен быть **последним** параметром. Все остальные могут идти в любом порядке. Ошибка
> компилятора при неверном порядке запутанная; см. «Частые ошибки».

## 4. Ответы: всё, что реализует `IntoResponse`

Обработчик может вернуть любой тип, реализующий `IntoResponse`. Здесь используются такие:

| Возвращаемое значение | Ответ |
|---|---|
| `&'static str`, `String` | `200`, `content-type: text/plain; charset=utf-8`, текст |
| `Json(value)` | `200`, `content-type: application/json`, `value`, сериализованное через serde |
| `StatusCode::NO_CONTENT` | этот статус, пустое тело |
| `(StatusCode, [(header, value)], Json(task))` | статус, дополнительные заголовки, JSON-тело |
| `Result<A, B>`, где оба реализуют `IntoResponse` | `A` для `Ok`, `B` для `Err` |

`create_task` возвращает кортеж: `201 Created`, заголовок `location` и новую задачу. Его тип возврата —
`impl IntoResponse`, что означает «какой-то тип, реализующий `IntoResponse`; компилятор знает какой». Это
избавляет от написания длинного типа кортежа.

`get_task` возвращает `Result<Json<Task>, StatusCode>`. Поскольку `Result` реализует `IntoResponse`, оператор
`?` работает в обработчиках: в `update_task` выражение `.ok_or(StatusCode::NOT_FOUND)?` превращает
отсутствующую задачу в ответ `404` и досрочно возвращается. Пока ошибка — это только голый код статуса;
Урок 16.4 превратит ошибки в JSON-тело ошибки из контракта.

## 5. Общее состояние

Каждый запрос выполняется в своей задаче, возможно, на другом потоке (Урок 16.1). Все они должны видеть
**одно и то же** хранилище, поэтому хранилище лежит за `Arc` (общее владение между задачами) и `Mutex` (менять
его может только одна задача за раз). `.with_state(state)` отдаёт маршрутизатору копию `Arc`; на каждый запрос
axum клонирует `Arc` (дёшево: увеличивается только счётчик) и передаёт его экстрактору `State`.

```text
  request 1 ─▶ task ─▶ State(state) ─┐
  request 2 ─▶ task ─▶ State(state) ─┼─▶ Arc ─▶ Mutex ─▶ Store { last_id, tasks }
  request 3 ─▶ task ─▶ State(state) ─┘
```

Это стандартный `std::sync::Mutex`, а не асинхронный. Так можно, **потому что ни один обработчик не держит
блокировку через `.await`**: каждый блокирует, меняет словарь и отпускает guard в конце функции. Блокирующий
мьютекс, который удерживается несколько микросекунд, не вредит рантайму.

> ⚡ **Производительность:** делайте критические секции крошечными: заблокировать, скопировать или изменить
> нужное, разблокировать. Никогда не делайте медленную работу (сетевой вызов, запрос к базе данных), пока
> держите блокировку.

Вспомните Урок 16.1: это хранилище живёт в памяти процесса. Перезапустите сервис — и все задачи пропадут, а у
двух экземпляров было бы два разных хранилища. Уроки 16.5–16.7 переносят задачи в PostgreSQL.

## 6. Разговор с сервисом

Пока сервис работает, используйте второй терминал. Создайте задачу:

```bash
curl -i -H 'content-type: application/json' -d '{"title":"Buy milk"}' http://127.0.0.1:3000/tasks
```

```text
HTTP/1.1 201 Created
content-type: application/json
location: /tasks/1
content-length: 40
date: Wed, 18 Nov 2026 07:08:46 GMT

{"id":1,"title":"Buy milk","done":false}
```

Теперь пройдите остальной контракт (`-s` скрывает индикатор прогресса, `echo` добавляет перевод строки после
JSON):

```bash
curl -s -H 'content-type: application/json' -d '{"title":"Write lesson"}' http://127.0.0.1:3000/tasks; echo
curl -s http://127.0.0.1:3000/tasks; echo
curl -s -X PATCH -H 'content-type: application/json' -d '{"done":true}' http://127.0.0.1:3000/tasks/1; echo
curl -s 'http://127.0.0.1:3000/tasks?done=false'; echo
```

```text
{"id":2,"title":"Write lesson","done":false}
[{"id":1,"title":"Buy milk","done":false},{"id":2,"title":"Write lesson","done":false}]
{"id":1,"title":"Buy milk","done":true}
[{"id":2,"title":"Write lesson","done":false}]
```

URL (Uniform Resource Locator, единый указатель ресурса) с `?` взят в кавычки, потому что командная оболочка придаёт `?` особый смысл. Наконец, удалите задачу 1 и
попробуйте её прочитать:

```bash
curl -i -X DELETE http://127.0.0.1:3000/tasks/1
curl -i http://127.0.0.1:3000/tasks/1
```

```text
HTTP/1.1 204 No Content
date: Wed, 18 Nov 2026 07:08:46 GMT

HTTP/1.1 404 Not Found
content-length: 0
date: Wed, 18 Nov 2026 07:08:46 GMT
```

Это ровно разговор из Урока 16.2, раздел 5, только у `404` пока пустое тело.

## 7. Что axum делает с плохим вводом

Отправьте несколько сломанных запросов, как сделал бы злоумышленник или клиент с ошибкой:

```bash
curl -i -H 'content-type: application/json' -d '{"title":' http://127.0.0.1:3000/tasks
curl -i -d '{"title":"x"}' http://127.0.0.1:3000/tasks
curl -i -H 'content-type: application/json' -d '{"name":"x"}' http://127.0.0.1:3000/tasks
curl -i http://127.0.0.1:3000/tasks/abc
curl -i 'http://127.0.0.1:3000/tasks?done=maybe'
```

Ответы без строк `date` и `content-length`:

```text
HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8

Failed to parse the request body as JSON: title: EOF while parsing a value at line 1 column 9

HTTP/1.1 415 Unsupported Media Type
content-type: text/plain; charset=utf-8

Expected request with `Content-Type: application/json`

HTTP/1.1 422 Unprocessable Entity
content-type: text/plain; charset=utf-8

Failed to deserialize the JSON body into the target type: missing field `title` at line 1 column 12

HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8

Invalid URL: Cannot parse `abc` to a `u64`

HTTP/1.1 400 Bad Request
content-type: text/plain; charset=utf-8

Failed to deserialize query string: done: provided string was not `true` or `false`
```

**Что происходит:**

1. Сломанный JSON → экстрактор `Json` отказывает с `400`.
2. Без `-H 'content-type: application/json'` команда `curl -d` отправляет тип
   `application/x-www-form-urlencoded` (формат форм HTML — HyperText Markup Language, языка разметки веб-страниц), поэтому `Json` отказывает с `415`.
3. Корректный JSON, но без поля `title` → `422`. (hyper печатает старое название `422`, «Unprocessable Entity»;
   RFC (Request for Comments, запрос комментариев) 9110 называет его «Unprocessable Content». Значение имеет число.)
4. `abc` — не `u64` → экстрактор `Path` отказывает с `400`.
5. `maybe` — не `bool` → экстрактор `Query` отказывает с `400`.

Коды статуса уже совпадают с контрактом, и ни для одного из этих запросов не выполнился обработчик. Две вещи
ещё неправильны: тела — простой текст, а не наш JSON-формат ошибок, и ничто не мешает **пустому** заголовку
(`{"title":""}` создаёт задачу с заголовком `""`). Урок 16.4 исправляет и то и другое.

Остановите сервис клавишами `Ctrl+C`.

## 8. Тестирование маршрутизатора без сети

`Router` — это `Service` из tower: нечто, что принимает запрос и возвращает ответ. `ServiceExt::oneshot` из
tower отправляет один запрос **напрямую**, в памяти: без порта, без `curl`, очень быстро. Добавьте это в конец
`src/main.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use axum::body::Body;
    use axum::http::Request;
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    /// Отправляет маршрутизатору один запрос и возвращает статус и тело в виде текста.
    async fn send(app: Router, request: Request<Body>) -> (StatusCode, String) {
        let response = app.oneshot(request).await.unwrap();
        let status = response.status();
        let bytes = response.into_body().collect().await.unwrap().to_bytes();
        (status, String::from_utf8(bytes.to_vec()).unwrap())
    }

    fn post_json(uri: &str, json: &'static str) -> Request<Body> {
        Request::post(uri)
            .header(header::CONTENT_TYPE, "application/json")
            .body(Body::from(json))
            .unwrap()
    }

    #[tokio::test]
    async fn created_task_can_be_read_back() {
        let app = app(AppState::default());

        let (status, body) = send(app.clone(), post_json("/tasks", r#"{"title":"Buy milk"}"#)).await;
        assert_eq!(status, StatusCode::CREATED);
        assert_eq!(body, r#"{"id":1,"title":"Buy milk","done":false}"#);

        let request = Request::get("/tasks/1").body(Body::empty()).unwrap();
        let (status, body) = send(app, request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, r#"{"id":1,"title":"Buy milk","done":false}"#);
    }

    #[tokio::test]
    async fn unknown_task_is_not_found() {
        let app = app(AppState::default());
        let request = Request::get("/tasks/42").body(Body::empty()).unwrap();
        let (status, _body) = send(app, request).await;
        assert_eq!(status, StatusCode::NOT_FOUND);
    }
}
```

```text
running 2 tests
test tests::unknown_task_is_not_found ... ok
test tests::created_task_can_be_read_back ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Что происходит:**

1. `Request::post(uri).header(...).body(...)` строит HTTP-запрос как значение Rust (тип взят из крейта
   `http`, который axum реэкспортирует как `axum::http`).
2. `app.oneshot(request)` поглощает маршрутизатор и возвращает ответ. Поэтому первый вызов использует
   `app.clone()`: клон маршрутизатора разделяет тот же `Arc`, так что второй запрос видит задачу, созданную
   первым.
3. Тело ответа приходит потоком кусков. `collect()` (из `http_body_util::BodyExt`) ждёт их все, а `to_bytes()`
   их соединяет.
4. Каждый тест создаёт свой `AppState::default()`, поэтому тесты никогда не видят задач друг друга и могут идти
   параллельно.

> 🧪 **Тестирование:** эти тесты проходят через маршрутизацию, экстракторы, обработчики и сериализацию — через
> всё, кроме TCP-сокета (Transmission Control Protocol, протокол управления передачей). Это делает их хорошим средним уровнем пирамиды тестов (Урок 09.1). Пишите по одному на
> каждую строку таблицы контракта.

## Частые ошибки

**1. Экстрактор тела не последний.** Вы пишете `Json(new_task): Json<NewTask>` перед `State(...)`:

```text
error[E0277]: the trait bound `fn(Json<NewTask>, State<_>) -> _ {create_task}: Handler<_, _>` is not satisfied
   --> src/main.rs:116:47
    |
116 |         .route("/tasks", get(list_tasks).post(create_task))
    |                                          ---- ^^^^^^^^^^^ unsatisfied trait bound
    = note: Consider using `#[axum::debug_handler]` to improve the error message
```

Сообщение говорит только «эта функция — не обработчик». Чтобы получить сообщение получше, включите фичу
`macros` (`cargo add axum@0.8 --features macros`) и поставьте `#[axum::debug_handler]` над функцией. Теперь
компилятор говорит точно, что не так:

```text
error: `Json<_>` consumes the request body and thus must be the last argument to the handler function
  --> src/main.rs:60:21
   |
60 |     Json(new_task): Json<NewTask>,
   |                     ^^^^
```

Исправление: перенесите параметр `Json` в конец. Та же ошибка «не обработчик» появляется и тогда, когда тип
параметра — не экстрактор, или когда future обработчика не `Send` (например, `MutexGuard` удерживается через
`.await`).

**2. Старый синтаксис путей.** Старые версии axum (и многие статьи в блогах) пишут `"/tasks/:id"`. axum 0.8
отвергает его при построении маршрутизатора, поэтому программа компилируется, но паникует при запуске:

```text
listening on http://127.0.0.1:3000

thread 'main' (5506020) panicked at src/main.rs:117:10:
Path segments must not start with `:`. For capture groups, use `{capture}`. If you meant to literally match a segment starting with a colon, call `without_v07_checks` on the router.
```

Исправление: `"/tasks/{id}"`.

**3. Забытый `.with_state(state)`.** Маршрутизатор, обработчики которого используют `State<AppState>`, — это
`Router<AppState>`: «маршрутизатор, которому ещё нужен `AppState`». Только после `.with_state(...)` он
становится простым `Router`, который принимает `axum::serve`:

```text
error[E0308]: mismatched types
   --> src/main.rs:114:5
    |
113 |   fn app(state: AppState) -> Router {
    |                              ------ expected `Router` because of return type
...
    = note: expected struct `Router<()>`
               found struct `Router<Arc<std::sync::Mutex<Store>>>`
```

## Попробуйте сами

1. Снова запустите сервис и создайте две задачи. Отметьте задачу 2 выполненной, затем отправьте `PATCH /tasks/2`
   только с `{"title":"Write lesson 16.3"}`. Убедитесь, что `done` по-прежнему `true`.
2. Перезапустите сервис и выполните `curl -s http://127.0.0.1:3000/tasks`. Список пуст (`[]`): состояние жило в
   памяти.
3. Запустите `cargo test` и `cargo clippy`, затем сделайте коммит: `git add code/p16-tasks && git commit -m "p16-tasks: in-memory tasks API"`.

## Упражнения

### Упражнение 1 — Тест удаления (легко)

Напишите тест `deleted_task_is_gone_and_delete_is_idempotent`: создайте задачу, `DELETE /tasks/1` (ожидается
`204`), `GET /tasks/1` (ожидается `404`) и ещё раз `DELETE /tasks/1` (ожидается `404`). Используйте помощники
`send` и `post_json`.

<details>
<summary>Подсказка</summary>

`Request::delete("/tasks/1").body(Body::empty()).unwrap()` строит запрос. Он нужен дважды; небольшое замыкание
`let delete = || Request::delete(...)...;` экономит набор.

</details>

<details>
<summary>Ответ</summary>

```rust
    #[tokio::test]
    async fn deleted_task_is_gone_and_delete_is_idempotent() {
        let app = app(AppState::default());
        send(app.clone(), post_json("/tasks", r#"{"title":"Buy milk"}"#)).await;

        let delete = || Request::delete("/tasks/1").body(Body::empty()).unwrap();
        let (status, _) = send(app.clone(), delete()).await;
        assert_eq!(status, StatusCode::NO_CONTENT);

        let get = Request::get("/tasks/1").body(Body::empty()).unwrap();
        let (status, _) = send(app.clone(), get).await;
        assert_eq!(status, StatusCode::NOT_FOUND);

        // Второй DELETE ничего не находит, но итоговое состояние то же: задачи 1 нет.
        let (status, _) = send(app, delete()).await;
        assert_eq!(status, StatusCode::NOT_FOUND);
    }
```

Почему это работает: клоны маршрутизатора разделяют одно хранилище, поэтому запросы видят результаты друг
друга. Статус второго `DELETE` другой (`404`), но **состояние** такое же, как после первого. Именно это и
значит «идемпотентный» (Урок 16.2).

</details>

### Упражнение 2 — `limit` для списка (средне)

Добавьте к `GET /tasks` необязательный параметр запроса `limit`: возвращать не больше `limit` задач, 50, если он
не указан, и никогда больше 100. Напишите тест, который создаёт три задачи и проверяет, что `?limit=2`
возвращает первые две.

<details>
<summary>Подсказка</summary>

Добавьте `limit: Option<usize>` в `ListFilter`. В цепочке итераторов `.take(n)` останавливается после `n`
элементов. `unwrap_or(50).min(100)` выражает правило одним выражением.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, Deserialize)]
struct ListFilter {
    done: Option<bool>,
    limit: Option<usize>,
}

// в list_tasks, после .filter(...): .take(filter.limit.unwrap_or(50).min(100))

    #[tokio::test]
    async fn limit_returns_only_the_first_tasks() {
        let app = app(AppState::default());
        for title in ["a", "b", "c"] {
            let body = format!(r#"{{"title":"{title}"}}"#);
            let request = Request::post("/tasks")
                .header(header::CONTENT_TYPE, "application/json")
                .body(Body::from(body))
                .unwrap();
            send(app.clone(), request).await;
        }
        let request = Request::get("/tasks?limit=2").body(Body::empty()).unwrap();
        let (status, body) = send(app, request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(
            body,
            r#"[{"id":1,"title":"a","done":false},{"id":2,"title":"b","done":false}]"#
        );
    }
```

Почему это работает: `Query` десериализует `limit=2` в `Some(2)`; отсутствующий `limit` — это `None`. Предел в
100 защищает сервис от `?limit=100000000` (Урок 10.4: ограничения на всё). В `format!` `{{` и `}}` — буквальные
фигурные скобки.

</details>

### Упражнение 3 — `PUT` заменяет задачу (сложно)

Добавьте `PUT /tasks/{id}` с телом `{"title": "...", "done": true}`: оба поля **обязательны**, и задача
заменяется полностью. Отвечайте `200` с задачей или `404`. Напишите тест, который дважды отправляет один и тот
же `PUT` и проверяет, что оба ответа равны: `PUT` идемпотентен.

<details>
<summary>Подсказка</summary>

Создайте новую структуру `ReplaceTask { title: String, done: bool }` (без `Option`, чтобы отсутствующее поле
давало `422`). Добавьте `.put(replace_task)` в цепочку методов для `"/tasks/{id}"`. Кортежи
`(StatusCode, String)` можно сравнивать через `assert_eq!`.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, Deserialize)]
struct ReplaceTask {
    title: String,
    done: bool,
}

async fn replace_task(
    State(state): State<AppState>,
    Path(id): Path<u64>,
    Json(new): Json<ReplaceTask>,
) -> Result<Json<Task>, StatusCode> {
    let mut store = state.lock().unwrap();
    let task = store.tasks.get_mut(&id).ok_or(StatusCode::NOT_FOUND)?;
    task.title = new.title;
    task.done = new.done;
    Ok(Json(task.clone()))
}

// в app(): get(get_task).put(replace_task).patch(update_task).delete(delete_task)

    #[tokio::test]
    async fn put_twice_gives_the_same_result() {
        let app = app(AppState::default());
        send(app.clone(), post_json("/tasks", r#"{"title":"Buy milk"}"#)).await;

        let put = || {
            Request::put("/tasks/1")
                .header(header::CONTENT_TYPE, "application/json")
                .body(Body::from(r#"{"title":"Buy oat milk","done":true}"#))
                .unwrap()
        };
        let first = send(app.clone(), put()).await;
        let second = send(app.clone(), put()).await;
        assert_eq!(first, second);
        assert_eq!(first.0, StatusCode::OK);
        assert_eq!(first.1, r#"{"id":1,"title":"Buy oat milk","done":true}"#);
    }
```

Все пять тестов (два из раздела 8 и три из упражнений) проходят:

```text
test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `PUT` присваивает каждому полю значение, заданное клиентом, поэтому результат не зависит от
того, сколько раз он выполнится. `PATCH` с полями `Option` и `PUT` с обязательными полями — два разных
контракта; типы делают разницу видимой.

</details>

## Вопросы для самопроверки

1. Что такое экстрактор? Назовите четыре.

<details>
<summary>Ответ</summary>

Тип параметра обработчика, который axum заполняет из запроса до запуска обработчика: `Path`, `Query`, `Json`,
`State` (а также `String`, `Bytes`, `HeaderMap` и другие).

</details>

2. Клиент отправляет `GET /tasks/abc`. Какой код выполняется и каков ответ?

<details>
<summary>Ответ</summary>

Никакой ваш код: экстрактор `Path<u64>` не справляется, и axum отвечает отказом, `400 Bad Request`,
``Invalid URL: Cannot parse `abc` to a `u64` ``. `get_task` не выполняется.

</details>

3. Почему `Json<T>` должен быть последним параметром обработчика?

<details>
<summary>Ответ</summary>

Он читает (поглощает) тело запроса, а тело можно прочитать только один раз. Экстракторы, которые смотрят
только на путь, строку запроса, заголовки или состояние, могут стоять перед ним.

</details>

4. Почему в этих обработчиках допустим `std::sync::Mutex`?

<details>
<summary>Ответ</summary>

Каждый обработчик держит блокировку очень недолго и никогда через `.await`. Короткая блокирующая блокировка не
останавливает рантайм, и guard никогда не хранится в future, который должен быть `Send`.

</details>

5. Что `.with_state(state)` меняет в типе маршрутизатора?

<details>
<summary>Ответ</summary>

Он превращает `Router<AppState>` (маршрутизатор, которому ещё нужно его состояние) в `Router<()>`, который
пишется просто `Router` и который можно запустить.

</details>

## Итоги

- `Router` сопоставляет путь и метод с обработчиком: `.route("/tasks", get(list_tasks).post(create_task))`;
  параметры пути пишутся как `{id}`.
- Экстракторы (`Path`, `Query`, `Json`, `State`) превращают запрос в типизированные значения; если один из них
  не справляется, axum отвечает отказом (`400`, `415`, `422`), и обработчик не выполняется.
- Экстрактор тела должен быть последним параметром; `#[axum::debug_handler]` объясняет ошибки обработчиков.
- Обработчики возвращают всё, что реализует `IntoResponse`: `Json`, `StatusCode`, кортежи с заголовками и
  `Result`, благодаря которому работает `?`.
- Общее состояние передаётся в `.with_state(...)`; для данных в памяти подходит `Arc<Mutex<…>>` с крошечными
  критическими секциями.
- `tower::ServiceExt::oneshot` тестирует весь маршрутизатор в памяти.

## Источники и что почитать

- [axum — документация крейта](https://docs.rs/axum/0.8) — маршрутизация, обработчики и обзор всех экстракторов.
- [axum::extract module](https://docs.rs/axum/0.8/axum/extract/index.html) — каждый экстрактор, правило порядка и как работают отказы.
- [axum::response module](https://docs.rs/axum/0.8/axum/response/index.html) — все способы построить ответ через `IntoResponse`.
- [axum examples](https://github.com/tokio-rs/axum/tree/main/examples) — официальные небольшие программы, включая пример «todos» и примеры тестов.
- [tower::ServiceExt](https://docs.rs/tower/0.5/tower/trait.ServiceExt.html) — `oneshot` и другие помощники, используемые в тестах.

---

[🏠 Начало](../START-HERE.md) · [← 16.2 Проектирование HTTP API: REST](02-designing-http-apis.md) · [📋 Этап 16](README.md) · [16.4 Ошибки, проверка ввода и middleware →](04-errors-validation-and-middleware.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
