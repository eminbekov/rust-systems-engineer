<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 20.9 Чек-лист готовности к продакшену](09-production-readiness-checklist.md) · [📋 Этап 20](README.md) · [20.11 Контрольная точка →](11-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/20-reliability-and-hardening/10-project-harden-a-service.md) <!-- nav -->

# Урок 20.10 — Проект: укрепить сервис

> **Этап 20 — Надёжность и защита** · Урок 10 из 11 · ⏱ около 10 часов (5 занятий)

## Что вы узнаете

- Как проверить работающий сервис по чек-листу готовности к продакшену и найти его слабые места.
- Как превратить паники в ответы и добавить дедлайн, лимиты размера и сброс нагрузки, покрытые тестами.
- Как применить Miri и фаззинг к тем частям сервиса, где они уместны, и превратить каждую находку в регрессионный тест.
- Как добавить проверки цепочки поставок и хаос-тест с toxiproxy и доказать числами, что сервис теперь быстро отказывает.

Это **урок-проект**. Он даёт вам небольшой сервис, который работает, но не готов к продакшену, цель (контракт), план с вехами, подсказки и команды, которые доказывают, что каждая веха выполнена. Укреплённого кода он не даёт.

## Перед началом

- Вы прошли [Урок 20.9 — Чек-лист готовности к продакшену](09-production-readiness-checklist.md) и предыдущие уроки этого этапа. Вам понадобятся паники и линты Clippy ([20.2](02-errors-and-panics-in-production.md)), фаззинг ([20.3](03-fuzzing-in-depth.md)), Miri ([20.4](04-miri-sanitizers-and-loom-in-ci.md)), toxiproxy ([20.5](05-fault-injection-and-chaos-testing.md)), инструменты цепочки поставок ([20.7](07-supply-chain-security-in-depth.md)), лимиты ([20.8](08-overload-limits-and-backpressure.md)) и чек-лист со скриптом `readiness-check.sh` ([20.9](09-production-readiness-checklist.md)).
- Вы знаете axum и Redis по Этапу 16 ([Урок 16.3](../16-backend-services/03-web-services-with-axum.md), [Урок 16.10](../16-backend-services/10-caching-with-redis.md)). Если вы построили сервис коротких ссылок (URL — Uniform Resource Locator, адрес в интернете) из [Урока 16.15](../16-backend-services/15-project-url-shortener-service.md), можете укреплять его вместо стартового кода ниже; вехи работают так же.
- Инструменты: ночной тулчейн (nightly) с Miri, `cargo-fuzz`, `cargo-audit`, `cargo-deny`, `cargo-vet`, `cargo-auditable`, `cargo-cyclonedx`, Docker, `curl` и `jq`.
- Git: проект лежит в `~/projects/rust/code/p20-shortener`. Работайте в ветке (`git switch -c project/20-hardening`) и делайте коммит после каждой вехи.

## Новые слова

| Слово | Значение |
|---|---|
| **стартовый код** (starter code) | Код, который урок даёт как отправную точку. Здесь он работает, но в нём намеренно оставлены слабые места. |
| **слабое место** (weak point) | Место, где сервис может упасть, зависнуть, подвергнуться злоупотреблению или отказать так, что никто не заметит. |
| **toxic** («токсин») | В toxiproxy — сбой, добавленный к соединению: лишняя задержка, разорванное соединение, медленная струйка байтов. |
| **дедлайн** (deadline) | Последний момент, к которому нужно ответить на весь запрос; здесь — 300 мс после его прихода. |
| **регрессионный тест** (regression test) | Тест, который повторяет вход, однажды нашедший ошибку, чтобы она не вернулась незаметно. |

## Проект

### Что вы создадите

Вы получаете **p20-shortener** — сервис коротких ссылок с тремя эндпоинтами. `POST /links` сохраняет URL в Redis и возвращает короткий код; `GET /{code}` перенаправляет на URL; `GET /health` отвечает `ok`. Тесты он проходит. Но при плохом вводе, медленном или остановленном Redis он паникует, и от перегрузки его ничто не защищает.

Вы будете укреплять его шаг за шагом, измерять каждое изменение и закончите заполненным `READINESS.md`, в котором у каждого пункта есть доказательство. Сервис обращается к Redis через **toxiproxy**, так что вы можете в любой момент сделать Redis медленным или «упавшим»:

```text
 curl / oha ──▶ p20-shortener ──▶ toxiproxy :26379 ──────▶ course-p20-redis :6379
 (your Mac)     127.0.0.1:3000    (Docker, API on :18474)    (Docker)
```

### Стартовый код

Создайте пакет и добавьте крейты:

```bash
cd ~/projects/rust
cargo new --lib code/p20-shortener
cd code/p20-shortener
cargo add axum@0.8
cargo add tokio@1 --features full
cargo add serde@1 --features derive
cargo add redis@1 --features tokio-comp,connection-manager
cargo add --dev tower@0.5 --features util
mkdir tests
```

`src/code.rs` превращает числа в короткие коды и обратно:

```rust
//! Short codes: a number written in base 62 (digits, then a-z, then A-Z).

const ALPHABET: &[u8; 62] = b"0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

/// The value of every ASCII character, or 255 for characters outside the alphabet.
const VALUES: [u8; 128] = {
    let mut table = [255u8; 128];
    let mut i = 0;
    while i < 62 {
        table[ALPHABET[i] as usize] = i as u8;
        i += 1;
    }
    table
};

/// Writes `n` in base 62: 0 → "0", 61 → "Z", 62 → "10".
pub fn encode(mut n: u64) -> String {
    if n == 0 {
        return "0".to_string();
    }
    let mut digits = Vec::new();
    while n > 0 {
        digits.push(ALPHABET[(n % 62) as usize]);
        n /= 62;
    }
    digits.reverse();
    String::from_utf8(digits).expect("the alphabet is ASCII")
}

/// Reads a base-62 code back into a number. `None` if it is not a valid code.
pub fn decode(code: &str) -> Option<u64> {
    if code.is_empty() {
        return None;
    }
    let mut n: u64 = 0;
    for byte in code.bytes() {
        // SAFETY: a byte is at most 255, and the table covers every byte. (Fast: no bounds check.)
        let value = unsafe { *VALUES.as_ptr().add(usize::from(byte)) };
        if value == 255 {
            return None;
        }
        n = n * 62 + u64::from(value);
    }
    Some(n)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn known_values_and_round_trip() {
        assert_eq!(encode(0), "0");
        assert_eq!(encode(62), "10");
        assert_eq!(decode("10"), Some(62));
        assert_eq!(decode("ab-c"), None);
        for n in [1, 7, 1_000, 123_456_789] {
            assert_eq!(decode(&encode(n)), Some(n));
        }
    }
}
```

`src/store.rs` прячет хранилище за трейтом, чтобы тесты могли использовать хранилище в памяти:

```rust
//! Where links are kept: Redis in production, memory in tests.

use std::collections::HashMap;
use std::sync::{Arc, Mutex};

use redis::AsyncCommands;
use redis::aio::ConnectionManager;

#[derive(Debug)]
pub struct StoreError(pub String);

impl From<redis::RedisError> for StoreError {
    fn from(err: redis::RedisError) -> Self {
        StoreError(err.to_string())
    }
}

/// Everything the handlers need from a storage.
pub trait Store: Clone + Send + Sync + 'static {
    /// Saves a URL and returns its new number.
    fn save(&self, url: &str) -> impl Future<Output = Result<u64, StoreError>> + Send;
    /// Finds the URL with this number.
    fn load(&self, id: u64) -> impl Future<Output = Result<Option<String>, StoreError>> + Send;
}

#[derive(Clone)]
pub struct RedisStore {
    conn: ConnectionManager,
}

impl RedisStore {
    pub async fn connect(url: &str) -> Result<RedisStore, StoreError> {
        let client = redis::Client::open(url)?;
        let conn = client.get_connection_manager().await?;
        Ok(RedisStore { conn })
    }
}

impl Store for RedisStore {
    fn save(&self, url: &str) -> impl Future<Output = Result<u64, StoreError>> + Send {
        let mut conn = self.conn.clone();
        let url = url.to_string();
        async move {
            let id: u64 = conn.incr("links:next_id", 1).await?;
            let () = conn.set(format!("link:{id}"), url).await?;
            Ok(id)
        }
    }

    fn load(&self, id: u64) -> impl Future<Output = Result<Option<String>, StoreError>> + Send {
        let mut conn = self.conn.clone();
        async move { Ok(conn.get(format!("link:{id}")).await?) }
    }
}

/// A store in memory, for tests.
#[derive(Clone, Default)]
pub struct MemoryStore {
    links: Arc<Mutex<HashMap<u64, String>>>,
}

impl Store for MemoryStore {
    fn save(&self, url: &str) -> impl Future<Output = Result<u64, StoreError>> + Send {
        let mut links = self.links.lock().unwrap();
        let id = links.len() as u64 + 1;
        links.insert(id, url.to_string());
        async move { Ok(id) }
    }

    fn load(&self, id: u64) -> impl Future<Output = Result<Option<String>, StoreError>> + Send {
        let url = self.links.lock().unwrap().get(&id).cloned();
        async move { Ok(url) }
    }
}
```

В `src/lib.rs` — обработчики:

```rust
//! A small URL shortener. It works, but it is not ready for production yet.

pub mod code;
pub mod store;

use axum::extract::{Path, State};
use axum::http::StatusCode;
use axum::response::{IntoResponse, Redirect, Response};
use axum::routing::{get, post};
use axum::{Json, Router};
use serde::{Deserialize, Serialize};

use crate::code::{decode, encode};
use crate::store::Store;

/// Accepts only absolute http and https addresses of at most 2,000 bytes.
pub fn validate_url(input: &str) -> Result<&str, &'static str> {
    let input = input.trim();
    if input.len() > 2000 {
        return Err("url is too long");
    }
    if &input[..7] == "http://" || &input[..8] == "https://" {
        Ok(input)
    } else {
        Err("url must start with http:// or https://")
    }
}

#[derive(Deserialize)]
struct NewLink {
    url: String,
}

#[derive(Serialize)]
struct Created {
    code: String,
}

async fn create<S: Store>(
    State(store): State<S>,
    Json(body): Json<NewLink>,
) -> Result<(StatusCode, Json<Created>), (StatusCode, &'static str)> {
    let url = validate_url(&body.url).map_err(|e| (StatusCode::BAD_REQUEST, e))?;
    let id = store.save(url).await.unwrap();
    Ok((StatusCode::CREATED, Json(Created { code: encode(id) })))
}

async fn follow<S: Store>(State(store): State<S>, Path(code): Path<String>) -> Response {
    let Some(id) = decode(&code) else {
        return (StatusCode::NOT_FOUND, "no such link\n").into_response();
    };
    match store.load(id).await.unwrap() {
        Some(url) => Redirect::temporary(&url).into_response(),
        None => (StatusCode::NOT_FOUND, "no such link\n").into_response(),
    }
}

async fn health() -> &'static str {
    "ok\n"
}

pub fn app<S: Store>(store: S) -> Router {
    Router::new()
        .route("/links", post(create::<S>))
        .route("/health", get(health))
        .route("/{code}", get(follow::<S>))
        .with_state(store)
}
```

`src/main.rs`:

```rust
use p20_shortener::app;
use p20_shortener::store::RedisStore;

#[tokio::main]
async fn main() {
    let redis_url = std::env::var("REDIS_URL").unwrap_or("redis://127.0.0.1:26379/".to_string());
    let store = RedisStore::connect(&redis_url).await.unwrap();
    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000")
        .await
        .unwrap();
    println!("listening on http://127.0.0.1:3000");
    axum::serve(listener, app(store)).await.unwrap();
}
```

и `tests/api.rs`:

```rust
use axum::body::Body;
use axum::http::{Request, StatusCode, header};
use p20_shortener::app;
use p20_shortener::store::MemoryStore;
use tower::ServiceExt;

#[tokio::test]
async fn create_then_follow() {
    let app = app(MemoryStore::default());

    let create = Request::post("/links")
        .header(header::CONTENT_TYPE, "application/json")
        .body(Body::from(r#"{"url":"https://www.rust-lang.org/"}"#))
        .unwrap();
    let response = app.clone().oneshot(create).await.unwrap();
    assert_eq!(response.status(), StatusCode::CREATED);

    // The first link gets number 1, which is the code "1".
    let follow = Request::get("/1").body(Body::empty()).unwrap();
    let response = app.oneshot(follow).await.unwrap();
    assert_eq!(response.status(), StatusCode::TEMPORARY_REDIRECT);
    assert_eq!(
        response.headers()[header::LOCATION],
        "https://www.rust-lang.org/"
    );
}

#[tokio::test]
async fn unknown_code_is_404() {
    let response = app(MemoryStore::default())
        .oneshot(Request::get("/zz").body(Body::empty()).unwrap())
        .await
        .unwrap();
    assert_eq!(response.status(), StatusCode::NOT_FOUND);
}
```

`cargo test` проходит: 1 модульный тест и 2 теста API (Application Programming Interface, программный интерфейс), которые вызывают маршрутизатор напрямую.

Запустите Redis и toxiproxy (образ публикуют авторы toxiproxy) в одной сети Docker и создайте прокси:

```bash
docker network create course-p20-net
docker run -d --name course-p20-redis --network course-p20-net redis:8
docker run -d --name course-p20-toxiproxy --network course-p20-net \
  -p 127.0.0.1:18474:8474 -p 127.0.0.1:26379:26379 ghcr.io/shopify/toxiproxy:2.13.0
curl -s -X POST http://127.0.0.1:18474/proxies \
  -d '{"name":"redis","listen":"0.0.0.0:26379","upstream":"course-p20-redis:6379"}'
```

```text
{"name":"redis","listen":"[::]:26379","upstream":"course-p20-redis:6379","enabled":true,"Logger":{},"toxics":[]}
```

Порт API — `18474` вместо обычного для toxiproxy `8474`, чтобы не конфликтовать с контейнером из Урока 20.5, если тот ещё работает. На вашем Mac опубликован только toxiproxy; Redis доступен только через него.

### Контракт

Когда вы закончите, сервис ведёт себя так:

| Ситуация | Ответ |
|---|---|
| `POST /links` с правильным URL `http://` или `https://` | `201` и `{"code":"…"}` |
| `POST /links` с любым другим текстом в `url` (также `""`, `"https:/"`, текст не в ASCII — American Standard Code for Information Interchange, американский стандартный код для обмена информацией) | `400`, никогда не паника |
| Тело запроса больше 4 КиБ (4096 байт) | `413` |
| `GET /{code}` для сохранённого кода | `307` с `location: <url>` |
| `GET /{code}` для любого другого текста | `404`, никогда не паника |
| Redis возвращает ошибку или соединение разорвано | `503` с `retry-after: 1`, никогда не паника |
| Запрос не завершён через 300 мс | `503` через 300 мс |
| Больше 64 запросов одновременно | лишние сразу получают `503` |
| `GET /livez` | `200`, пока процесс работает |
| `GET /readyz` | `200`, только если сервис готов **и** Redis отвечает на `PING` за 200 мс; иначе `503` |
| SIGTERM (сигнал остановки) | `/readyz` становится `503`, отвод трафика 2 с, открытые запросы завершаются, код возврата `0` |
| `REDIS_URL` не задан | `config error: REDIS_URL is not set` в стандартный поток ошибок, код возврата `2` |

И проходят такие проверки:

| Проверка | Команда |
|---|---|
| Тесты, форматирование, Clippy, нет `unwrap` вне тестов, нет бюллетеней | `./readiness-check.sh` (Урок 20.9) |
| Miri на тестах библиотеки | `cargo +nightly miri test --lib` |
| Две цели фаззинга, по 60 секунд каждая без падения | `cargo +nightly fuzz run decode` и `… validate_url` |
| Лицензии, запреты, источники, бюллетени | `cargo deny check` |
| Каждая зависимость прошла аудит или в исключениях | `cargo vet` |
| Релизный бинарный файл несёт список зависимостей; есть SBOM (Software Bill of Materials, спецификация состава ПО) | `cargo auditable build --release`, `cargo cyclonedx` |
| Redis медленный, Redis упал, Redis вернулся | `./chaos.sh` |

### Пошаговый план

Проходите вехи по порядку. Выполните команды из «Готово, когда», затем сделайте коммит. Сначала попробуйте сами, потом открывайте подсказку.

---

#### M1 — Запустить, сломать, записать найденное

**Цель:** стартовый код работает с Redis, и у вас есть список его слабых мест.

**Указания:**

1. `cargo build --release` и `./target/release/p20-shortener`. Во втором терминале создайте ссылку и перейдите по ней.
2. Отправьте URL `https:/` и посмотрите на оба терминала.
3. Скопируйте `readiness-check.sh` из Урока 20.9 в проект и запустите его.
4. Создайте `READINESS.md` с таблицами из Урока 20.9. Прочитайте стартовый код с чек-листом в руках и запишите каждый пробел в столбец доказательств («нет: …»). Прочитайте также каждый блок `unsafe` и каждое обращение к срезу по индексу.
5. Закоммитьте стартовый код и `READINESS.md` (`chore: starter code and readiness baseline`).

<details>
<summary>Подсказка — что искать</summary>

Вы должны найти не меньше восьми слабых мест: `unwrap()` на каждом результате хранилища; нет собственного дедлайна (только внутренний тайм-аут клиента Redis по умолчанию, 500 мс); нет явного лимита тела; нет лимита конкурентности; `/health` вместо отдельных живости и готовности; нет корректного завершения; молчаливое значение по умолчанию для `REDIS_URL`; срезы в `validate_url`; комментарий `SAFETY` в `decode` (правда ли в таблице 256 элементов?); `n * 62` в `decode`; нет `deny.toml`, нет `supply-chain/`.

</details>

**Готово, когда:** создание ссылки и переход по ней работают:

```bash
curl -s -X POST http://127.0.0.1:3000/links -H 'content-type: application/json' \
  -d '{"url":"https://www.rust-lang.org/"}'
curl -si http://127.0.0.1:3000/1 | head -2
```

```text
{"code":"1"}
HTTP/1.1 307 Temporary Redirect
location: https://www.rust-lang.org/
```

короткий URL обрушивает запрос (код возврата curl 52 означает «пустой ответ от сервера»):

```bash
curl -s -X POST http://127.0.0.1:3000/links -H 'content-type: application/json' -d '{"url":"https:/"}'
echo "curl exit code: $?"
```

```text
curl exit code: 52
```

а в терминале сервиса видна паника:

```text
thread 'tokio-rt-worker' (6318885) panicked at src/lib.rs:22:42:
end byte index 8 is out of bounds for string of length 7
```

и `./readiness-check.sh` показывает ровно один `FAIL`: `no unwrap outside tests`. В `READINESS.md` перечислены ваши слабые места.

---

#### M2 — Ответы вместо паник, дедлайн и лимиты

**Цель:** никакая проблема хранилища не может обрушить или замедлить запрос, и лимиты из контракта на месте, каждый с тестом.

**Указания:**

1. Реализуйте для `StoreError` трейт `IntoResponse`, который пишет ошибку через `eprintln!` и отвечает `503` с `retry-after: 1`. Пусть обработчики возвращают `Result<Response, StoreError>` и используют `?` вместо `unwrap()`. `MemoryStore` тоже не должен использовать `unwrap()`: `lock().unwrap_or_else(PoisonError::into_inner)` берёт данные даже после паники.
2. Добавьте стек слоёв из Урока 20.8 к маршрутам `/links` и `/{code}`: `HandleErrorLayer`, сброс нагрузки, лимит конкурентности 64 и тайм-аут 300 мс. Поставьте `.with_state(...)` в самый конец `app`, чтобы слои строились один раз.
3. Добавьте к тем же маршрутам `DefaultBodyLimit::max(4 * 1024)` (из `axum::extract`).
4. Напишите в `tests/api.rs` тесты с двумя **фейковыми хранилищами**: `SlowStore` (каждый метод спит 10 с) и `BrokenStore` (каждый метод возвращает `Err`). Проверьте: `GET /1` со `SlowStore` отвечает `503` не позже чем через 300 мс остановленного времени; с `BrokenStore` — `503` с `retry-after: 1`; тело в 10 000 байт получает `413`.
5. Коммит (`feat: errors become 503, deadline, limits`).

<details>
<summary>Подсказка 1 — фейковое хранилище в несколько строк</summary>

В блоке `impl` метод трейта, объявленный как `fn … -> impl Future<Output = …> + Send`, можно реализовать через `async fn`:

```rust
#[derive(Clone)]
struct BrokenStore;

impl Store for BrokenStore {
    async fn save(&self, _url: &str) -> Result<u64, StoreError> {
        Err(StoreError("broken pipe".to_string()))
    }
    async fn load(&self, _id: u64) -> Result<Option<String>, StoreError> {
        Err(StoreError("broken pipe".to_string()))
    }
}
```

</details>

<details>
<summary>Подсказка 2 — измерение остановленного времени</summary>

Используйте `#[tokio::test(start_paused = true)]` и `let start = tokio::time::Instant::now();` перед запросом, затем `assert!(start.elapsed() <= Duration::from_millis(300));`. Добавьте `tokio` с фичей `test-util` как dev-зависимость.

</details>

**Готово, когда:** `cargo test --test api` показывает ваши новые тесты (имена ваши) и проходит:

```text
test broken_store_gives_503_not_a_panic ... ok
test big_body_is_413 ... ok
test unknown_code_is_404 ... ok
test create_then_follow ... ok
test slow_store_gives_503_within_the_deadline ... ok
```

`./readiness-check.sh` может ещё показывать `FAIL` для `no unwrap outside tests`, но только из-за `main.rs`, который вы исправите в M3.

---

#### M3 — Живость, готовность, конфигурация и остановка

**Цель:** строки контракта о жизненном цикле.

**Указания:**

1. Добавьте в трейт `Store` метод `fn ping(&self)` (Redis: команда `PING`; память: всегда `Ok`).
2. Замените `/health` на `/livez` и `/readyz`, как в Уроке 20.9. Держите флаг готовности и хранилище вместе в одном `AppState<S>`. `/readyz` даёт `ping()` не больше 200 мс.
3. В `main`: никакого значения по умолчанию для `REDIS_URL`, никаких `unwrap()` и остановка из Урока 20.9 (SIGTERM или Ctrl+C, «не готов», отвод трафика 2 с, завершение).
4. Расширьте тест с медленным хранилищем: `/readyz` отвечает `503`.
5. Коммит (`feat: liveness, readiness, drain on SIGTERM`).

<details>
<summary>Подсказка — PING с крейтом redis</summary>

`let _: String = redis::cmd("PING").query_async(&mut conn).await?;` отправляет любую команду и превращает ответ (`PONG`) в тот тип, который вы запросили.

</details>

**Готово, когда:**

```bash
env -u REDIS_URL cargo run -q
echo "exit code: $?"
```

```text
config error: REDIS_URL is not set
exit code: 2
```

и при работающем сервисе (`REDIS_URL=redis://127.0.0.1:26379/ ./target/release/p20-shortener`) примерно через 0,3 с после `kill -TERM <pid>` эндпоинты отвечают так, а процесс завершается в пределах 3 с:

```text
readyz 503  livez 200
```

`./readiness-check.sh` теперь показывает только строки `PASS`.

---

#### M4 — Miri на небезопасном коде

**Цель:** выяснить, действительно ли безопасен блок `unsafe` в `decode`, и сделать его безопасным.

**Указания:**

1. Запустите `cargo +nightly miri test --lib`. Он проходит. Почему? Какие входные данные тесты дают `decode`?
2. Добавьте тест с символом не из ASCII, например `decode("é")`, и снова запустите Miri.
3. Исправьте `decode` без `unsafe` и исправьте комментарий. После этого Miri должен проходить.
4. Коммит (`fix(code): no out-of-bounds read in decode`).

<details>
<summary>Подсказка</summary>

`"é"` в UTF-8 (обычной кодировке строк Rust) занимает два байта, оба больше 127, а в таблице 128 элементов. `VALUES.get(usize::from(byte))` возвращает `None` для такого байта; с `?` функция тоже возвращает `None`. Обычный `cargo test` с этой ошибкой может даже пройти: чтение за пределами таблицы — неопределённое поведение, и программа может прочитать то, что лежит рядом.

</details>

**Готово, когда:** до исправления Miri сообщает об ошибке (сокращено; имя теста ваше):

```text
test code::more_tests::rejects_non_ascii ... error: Undefined Behavior: in-bounds pointer arithmetic failed: attempting to offset pointer by 195 bytes, but got alloc43 which is only 128 bytes from the end of the allocation
  --> src/code.rs:38:31
   |
38 |         let value = unsafe { *VALUES.as_ptr().add(usize::from(byte)) };
   |                               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ Undefined Behavior occurred here
```

а после исправления `cargo +nightly miri test --lib` заканчивается строкой `test result: ok.` без ошибок (здесь Miri запускает только модульные тесты библиотеки; тестам API нужны сетевые части Tokio, для которых Miri не нужен).

---

#### M5 — Фаззинг двух парсеров

**Цель:** `decode` и `validate_url` принимают **любой** текст без паники, а каждое падение, найденное фаззером, стало регрессионным тестом.

**Указания:**

1. `cargo fuzz init -t decode` и `cargo fuzz add validate_url`. Каждая цель принимает `text: &str`.
2. `decode`: вызовите её, и если она вернула `Some(n)`, проверьте свойство `decode(&encode(n)) == Some(n)`.
3. `validate_url`: вызовите её, и если она вернула `Ok(url)`, проверьте, что `url` начинается с `http://` или `https://` и не имеет пробелов по краям.
4. Запустите каждую цель с `-- -max_total_time=60`. Для каждого падения: прочитайте его (`cargo +nightly fuzz run <target> <artifact>`), исправьте код, добавьте регрессионный тест с этим входом, запустите снова.
5. Закоммитьте цели и исправления (`test: fuzz targets for decode and validate_url`).

<details>
<summary>Подсказка 1 — что находит фаззер</summary>

В `validate_url`: `end byte index 7 is out of bounds for string of length 0`. В `decode` (когда ошибка, найденная Miri, исправлена): `attempt to multiply with overflow`, для длинного кода вроде `"OOHttztOHtt"`. Если вы пропустили M4, `decode` сначала сообщит `AddressSanitizer: global-buffer-overflow` — это та же ошибка, которую нашёл Miri.

</details>

<details>
<summary>Подсказка 2 — исправления</summary>

`str::starts_with` никогда не паникует. Против переполнения `n.checked_mul(62)?.checked_add(u64::from(value))?` возвращает `None` вместо неверного числа. Добавьте `assert_eq!(decode(&encode(u64::MAX)), Some(u64::MAX));`, чтобы знать, что самый большой допустимый код всё ещё работает.

</details>

**Готово, когда:** обе цели работают 60 секунд без падения и заканчиваются строкой вроде:

```text
Done 1102458 runs in 61 second(s)
```

(число запусков у вас будет другим), а `cargo test` проходит с новыми регрессионными тестами.

---

#### M6 — Цепочка поставок

**Цель:** зависимости проверены, записаны и поставляются вместе с бинарным файлом.

**Указания:**

1. Напишите `deny.toml`, как в Уроке 10.13, и добавьте `publish = false` в `Cargo.toml`. Запустите `cargo deny check licenses` и решите, какие лицензии разрешить: прочитайте их, а не просто скопируйте список.
2. `cargo audit`, затем `cargo vet init` и `cargo vet`. Закоммитьте `supply-chain/`.
3. Соберите релиз через `cargo auditable build --release`, проверьте его через `cargo audit bin` и создайте SBOM командой `cargo cyclonedx --format json`.
4. Добавьте эти команды в `READINESS.md` как доказательства. Коммит (`chore: supply-chain checks`).

<details>
<summary>Подсказка — список лицензий</summary>

Если разрешены только `MIT` и `Apache-2.0`, отвергаются `redis` (BSD-3-Clause, лицензия Berkeley Software Distribution), `xxhash-rust` (BSL-1.0, Boost Software License) и крейты `icu_*` (Unicode-3.0). Все три — разрешительные лицензии; если ваша команда их принимает, добавьте в `allow` `"BSD-3-Clause"`, `"BSL-1.0"` и `"Unicode-3.0"`.

</details>

**Готово, когда** (ваши числа будут другими):

```text
advisories ok, bans ok, licenses ok, sources ok
Vetting Succeeded (106 exempted)
       Found 'cargo auditable' data in target/release/p20-shortener (102 dependencies)
```

`cargo audit` и `cargo audit bin` ничего не находят, а `jq '.components | length' p20-shortener.cdx.json` печатает число (здесь `101`).

---

#### M7 — Хаос-тест

**Цель:** доказать числами, что сервис отказывает **быстро**, когда Redis медленный или упал, и восстанавливается сам.

**Указания:** сохраните этот скрипт как `chaos.sh` и сделайте его исполняемым. Для каждого состояния прокси он запрашивает `/1` и `/readyz` и печатает коды ответа и время запроса `/1`.

```bash
#!/usr/bin/env bash
# Breaks the connection to Redis in three ways and checks that the service answers fast every time.
set -u
API=http://127.0.0.1:18474
probe() {
    local label="$1"
    local link ready
    link=$(curl -s -o /dev/null -w '%{http_code} in %{time_total}s' http://127.0.0.1:3000/1)
    ready=$(curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1:3000/readyz)
    echo "$label  GET /1: $link   readyz: $ready"
}

probe "normal          "
curl -s -o /dev/null -X POST "$API/proxies/redis/toxics" \
    -d '{"name":"slow","type":"latency","stream":"downstream","attributes":{"latency":1000}}'
probe "Redis 1 s slow  "
curl -s -o /dev/null -X DELETE "$API/proxies/redis/toxics/slow"
curl -s -o /dev/null -X POST "$API/proxies/redis" -d '{"enabled":false}'
probe "Redis down      "
curl -s -o /dev/null -X POST "$API/proxies/redis" -d '{"enabled":true}'
sleep 2
probe "Redis back      "
```

Вот что он напечатал для **стартового кода** — картина «до» (вы можете увидеть её сами, если запустите скрипт сразу после M1; заглядывать вперёд разрешено):

```text
normal            GET /1: 307 in 0.002004s   readyz: 404
Redis 1 s slow    GET /1: 000 in 0.506946s   readyz: 000
Redis down        GET /1: 000 in 0.002065s   readyz: 000
Redis back        GET /1: 307 in 0.002589s   readyz: 404
```

`000` означает «вообще нет HTTP-ответа» (HTTP — HyperText Transfer Protocol, протокол передачи гипертекста): запрос запаниковал. (В стартовом коде нет `/readyz`, поэтому этот путь считается кодом.) Медленный случай занимает 0,5 с, потому что его остановил только внутренний тайм-аут клиента Redis.

<details>
<summary>Подсказка — если результат не тот, что вы ждали</summary>

Если медленный случай занимает 1 с, ваш дедлайн не покрывает вызов хранилища: проверьте, что слой тайм-аута стоит на маршруте и что `.with_state(...)` идёт последним. Если «Redis back» всё ещё показывает `503`, подождите чуть дольше: `ConnectionManager` переподключается с небольшой задержкой.

</details>

**Готово, когда:** для вашего укреплённого сервиса (сначала создайте ссылку `1`):

```text
normal            GET /1: 307 in 0.002501s   readyz: 200
Redis 1 s slow    GET /1: 503 in 0.302653s   readyz: 503
Redis down        GET /1: 503 in 0.000754s   readyz: 503
Redis back        GET /1: 307 in 0.001027s   readyz: 200
```

Ни одна строка не показывает `000`, ни один ответ не занимает больше примерно 0,31 с, и сервис восстановился без перезапуска. Закоммитьте `chaos.sh` (`test: chaos test with toxiproxy`).

---

#### M8 — Проверка готовности

**Цель:** законченный `READINESS.md` и проект, который может запустить кто-то другой.

**Указания:**

1. Ещё раз пройдите по `READINESS.md`. У каждого пункта есть доказательство (имя теста, команда, файл), или «не нужно» с причиной, или «нет» с планом (пункты Этапа 21, такие как откаты и дашборды, пока остаются «нет»).
2. Напишите `README.md`: что делает сервис, как запустить Redis и toxiproxy, как запускать тесты, фаззеры, Miri и `chaos.sh`.
3. `./readiness-check.sh` должен показывать только `PASS`.
4. Когда закончите, уберите за собой в Docker: `docker rm -f course-p20-toxiproxy course-p20-redis` и `docker network rm course-p20-net`.
5. Коммит (`docs: readiness review and README`) и слияние ветки.

**Готово, когда:** `./readiness-check.sh` печатает шесть строк `PASS` и завершается с `0`, а однокурсник (или вы завтра) может пройти по `README.md` из свежего клона.

### Дополнительные цели

- **Ограничение частоты для каждого клиента** на `POST /links` с token bucket из Урока 20.8: `429` с `retry-after` и тест.
- **До и после под нагрузкой:** запустите `oha -z 10s -c 200 http://127.0.0.1:3000/1` (Урок 19.2) против стартового и укреплённого сервиса, пока Redis медленный. Сравните перцентили задержки и коды ответа.
- **Другие toxics:** `timeout` (соединение перестаёт отвечать) и `slicer` (ответ приходит крошечными кусками). Отвечает ли сервис по-прежнему за 300 мс?
- **Workflow CI** (Continuous Integration, непрерывная интеграция) с тремя заданиями: проверка готовности при каждой отправке, фаззеры и Miri по расписанию (Уроки 20.3 и 20.4) и шаги релиза из Урока 20.7.

## Частые ошибки

**1. Укрепление без измерений.** Тайм-аут, не покрытый тестом или хаос-скриптом, может стоять не там (внутри лимита или не на маршруте). У каждого изменения должен быть тест или число.

**2. Заглушить Miri или фаззер.** Пометить падающий тест `#[cfg_attr(miri, ignore)]` или сократить прогон фаззера — значит спрятать ошибку. Исправляйте код; игнорируйте только тесты, которые Miri вообще не может выполнить (сеть, файлы), с комментарием.

**3. Проверка Redis в живости.** Тогда отказ Redis перезапускает каждую копию сервиса по кругу. Redis место только в готовности.

## Вопросы для самопроверки

1. В M1 запрос с `https:/` закончился кодом возврата curl 52, но сервис продолжил работать. Почему, и почему это всё равно серьёзная проблема?

<details>
<summary>Ответ</summary>

axum выполняет каждое соединение в отдельной задаче Tokio; паника раскрутила только эту задачу, поэтому сервер выжил. Но клиент не получил вообще никакого ответа, сообщение о панике — единственный след, а любой пользователь может повторять это сколько угодно: дешёвый способ тратить ресурсы сервера и засорять логи (отказ в обслуживании).

</details>

2. Почему `cargo +nightly miri test --lib` проходил на стартовом коде, хотя в `decode` было неопределённое поведение?

<details>
<summary>Ответ</summary>

Miri проверяет только тот код, который выполняется. Тесты стартового кода вызывали `decode` только с текстом в ASCII, поэтому чтение за границей не происходило. Miri нужны тесты (или входы фаззера), которые доходят до опасного пути.

</details>

3. С toxic задержки в 1 с стартовый код ответил через 0,5 с, а укреплённый сервис — через 0,3 с. Откуда берутся эти два числа?

<details>
<summary>Ответ</summary>

0,5 с — тайм-аут ответа по умолчанию внутри `ConnectionManager` из Redis; после этой ошибки стартовый код запаниковал. 0,3 с — собственный дедлайн сервиса (слой тайм-аута), который превращает медленный запрос в `503` раньше, чем истечёт тайм-аут клиента.

</details>

## Итоги

- Начинайте с исходной картины: запустите сервис, намеренно сломайте его и запишите каждый пробел в чек-лист, прежде чем менять код.
- Превращайте каждую ошибку хранилища в `503` с `retry-after`, давайте каждому запросу один дедлайн и добавляйте лимиты тела, конкурентности и сброс нагрузки, каждый с тестом на фейковом хранилище.
- Разделяйте живость и готовность, проверяйте конфигурацию при запуске и отводите трафик по SIGTERM.
- Miri находит неопределённое поведение только на путях, до которых доходят тесты; фаззер находит входы, о которых никто не подумал. Каждая находка становится регрессионным тестом.
- `cargo deny`, `cargo audit`, `cargo vet`, `cargo auditable` и SBOM делают зависимости видимыми и проверенными.
- Хаос-тест с toxiproxy превращает «должен быстро отказывать» в числа: ни одного ответа медленнее дедлайна, ни одной паники, восстановление без перезапуска.

## Источники и что почитать

- [toxiproxy на GitHub](https://github.com/Shopify/toxiproxy) — HTTP API, все toxics и их атрибуты.
- [axum: `DefaultBodyLimit`](https://docs.rs/axum/latest/axum/extract/struct.DefaultBodyLimit.html) — лимит размера тела для экстракторов.
- [redis: `ConnectionManager`](https://docs.rs/redis/latest/redis/aio/struct.ConnectionManager.html) — переподключение и тайм-ауты по умолчанию.
- [Rust Fuzz Book](https://rust-fuzz.github.io/book/) — цели `cargo fuzz`, корпус и файлы падений.
- [Miri на GitHub](https://github.com/rust-lang/miri) — что проверяет Miri и его ограничения.
- [cargo-deny book](https://embarkstudios.github.io/cargo-deny/) и [cargo-vet book](https://mozilla.github.io/cargo-vet/) — политики цепочки поставок.

---

[🏠 Начало](../START-HERE.md) · [← 20.9 Чек-лист готовности к продакшену](09-production-readiness-checklist.md) · [📋 Этап 20](README.md) · [20.11 Контрольная точка →](11-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
