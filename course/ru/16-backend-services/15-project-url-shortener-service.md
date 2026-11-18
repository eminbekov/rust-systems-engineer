<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 16.14 Тестирование бэкенд-сервиса](14-testing-a-backend-service.md) · [📋 Этап 16](README.md) · [16.16 Контрольная точка →](16-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/16-backend-services/15-project-url-shortener-service.md) <!-- nav -->

# Урок 16.15 — Проект: сервис коротких ссылок

> **Этап 16 — Бэкенд-сервисы** · Урок 15 из 16 · ⏱ около 12 часов (6 занятий)

## Что вы узнаете

- Как соединить всё из этого этапа в один настоящий сервис: REST API, PostgreSQL с миграциями, кэш Redis, аутентификацию, ограничение частоты запросов, трассировки, образ контейнера и интеграционные тесты.
- Как работать по письменному контракту и доказывать каждую веху приёмочными тестами.
- Как принимать проектные решения и записывать их в README.

У этого урока нет полного решения. План даёт контракт, вехи, небольшие подсказки и точные команды, которые показывают, что веха готова. Рассчитывайте на несколько занятий; время выше — только на чтение плана.

## Перед началом

- Вы прошли [Урок 16.14 — Тестирование бэкенд-сервиса](14-testing-a-backend-service.md). Здесь используется каждый урок этапа; держите их открытыми.
- Docker запущен. Свободные локальные порты: 8080 (сервис), 15432 и 16379 (база и кэш для разработки), 16686 (Jaeger).
- Создайте проект и ветку Git:

```bash
cd ~/projects/rust
git switch main
git switch -c project/p16-shortener
cargo new code/p16-shortener
```

## Новые слова

| Слово | Значение |
|---|---|
| **сервис коротких ссылок** (URL shortener) | Сервис, который превращает длинный URL (Uniform Resource Locator, адрес в интернете) в короткий и отправляет посетителей короткого на длинный. |
| **короткий код** (short code) | Короткая часть ссылки, например `learn` в `http://127.0.0.1:8080/learn`. |
| **перенаправление** (redirect) | Ответ HTTP (Hypertext Transfer Protocol, протокол передачи гипертекста) со статусом `3xx` и заголовком `Location`: «иди по этому другому адресу». |
| **`302 Found`** | Временное перенаправление: браузеры его не запоминают, поэтому каждый визит возвращается в сервис и может быть посчитан. (`301`/`308` — постоянные, браузеры их кэшируют.) |
| **токен на предъявителя** (bearer token) | Секретная строка, которую клиент отправляет в заголовке `Authorization: Bearer <token>`, чтобы доказать право пользоваться эндпоинтом (Урок 16.8). |
| **сравнение за постоянное время** (constant-time comparison) | Сравнение двух секретов, которое всегда занимает одинаковое время, чтобы злоумышленник не мог угадать токен байт за байтом по времени ответа. |
| **запись проектного решения** (design decision record) | Короткая заметка в README: что вы решили, почему и от чего отказались. |

## Проект

### Что вы создадите

```text
              ┌──────────────────── p16-shortener (axum) ─────────────────────┐
 curl / browser│  rate limit (Redis INCR) → auth (admin routes) → handlers     │
   ───────────►│                                                               │
               │  GET /{code}: cache-aside ──► Redis   link:v1:{code}          │
               │                    miss ────► PostgreSQL  links table        │
               │  spans ──► OTLP ──► Collector ──► Jaeger (UI on 16686)        │
               └───────────────────────────────────────────────────────────────┘
```

Администратор создаёт короткие ссылки с секретным токеном. Любой может открыть короткую ссылку и будет перенаправлен. Сервис считает переходы, кэширует поиск в Redis, ограничивает, сколько запросов может отправить один клиент, отправляет трассировки в Jaeger, работает в Docker Compose и имеет интеграционные тесты, которые сами запускают свои базы данных.

### Контракт

**Конфигурация** (переменные окружения, читаются один раз при запуске, Урок 16.13):

| Переменная | Обязательна | По умолчанию | Значение |
|---|---|---|---|
| `DATABASE_URL` | да | – | URL подключения к PostgreSQL |
| `REDIS_URL` | нет | `redis://127.0.0.1:6379/` | URL подключения к Redis |
| `ADMIN_TOKEN` | да | – | Токен администратора; не короче 32 символов |
| `PUBLIC_BASE_URL` | нет | `http://127.0.0.1:<APP_PORT>` | Используется, чтобы построить `short_url` |
| `APP_HOST` / `APP_PORT` | нет | `127.0.0.1` / `8080` | Адрес, который слушает сервис |
| `RATE_LIMIT_PER_MINUTE` | нет | `60` | Запросов в минуту с одного IP-адреса клиента (IP, Internet Protocol, межсетевой протокол) |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | нет | `http://localhost:4317` | Куда идут трассировки (Урок 16.12) |

Отсутствующее или неверное значение останавливает программу с кодом возврата 1 и сообщением `invalid configuration: …`. Секреты никогда не пишутся в лог.

**Эндпоинты:**

| Метод и путь | Аутентификация | Успех | Ошибки |
|---|---|---|---|
| `GET /healthz` | – | `200`, тело `ok` | – |
| `GET /readyz` | – | `200`, если PostgreSQL отвечает | `503` |
| `POST /api/links` | admin | `201` + созданная ссылка | `401`, `409`, `422`, `429` |
| `GET /api/links/{code}/stats` | admin | `200` + статистика | `401`, `404`, `429` |
| `GET /{code}` | – | `302` с `Location` и `x-cache` | `404`, `429` |

- **Запрос** для `POST /api/links`: `{"url": "<absolute http or https URL, at most 2048 bytes>", "code": "<optional custom code>"}` (абсолютный URL http или https не длиннее 2048 байт и необязательный свой код). Свой код — от 3 до 32 символов из `A–Z a–z 0–9 - _`, и это не `api`. Без `code` сервис генерирует 7 случайных букв и цифр и повторяет попытку при совпадении.
- **Созданная ссылка** (`201`): `{"code":"learn","url":"https://www.rust-lang.org/learn","short_url":"http://127.0.0.1:8080/learn"}`.
- **Статистика** (`200`): `{"code":"learn","url":"https://www.rust-lang.org/learn","clicks":3}`. Каждое успешное перенаправление добавляет один переход.
- **Ошибки** всегда имеют тело JSON (JavaScript Object Notation) `{"error":"<message>"}`. `401` также отправляет `WWW-Authenticate: Bearer`; `429` также отправляет `Retry-After: <seconds>`.
- **`x-cache`** у перенаправлений: `miss` (прочитано из PostgreSQL, теперь в кэше), `hit` (прочитано из Redis) или `unavailable` (Redis не ответил; использован PostgreSQL). Перенаправление должно работать во всех трёх случаях.
- **Ограничение частоты:** не больше `RATE_LIMIT_PER_MINUTE` запросов с одного IP-адреса клиента за фиксированное окно в одну минуту, для всех эндпоинтов, кроме `/healthz` и `/readyz`. Если Redis недоступен, запросы разрешаются (и в лог пишется предупреждение).

**База данных** (первая миграция, `migrations/20261118000000_create_links.sql`):

```sql
CREATE TABLE links (
    code TEXT PRIMARY KEY,
    url TEXT NOT NULL,
    clicks BIGINT NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Ключи Redis:** `link:v1:{code}` → URL, TTL 3600 с; `rate:v1:{ip}` → счётчик запросов текущего окна, TTL 60 с.

**Трассировки:** имя сервиса `p16-shortener`. Перенаправление создаёт спаны `request` → `redirect` → `cache_get`, затем (при промахе) `db_find_link` и `cache_put`, затем `db_count_click`.

**Качество:** `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` и `cargo test` проходят; по одному коммиту Git на веху; `README.md` с тем, как запускать, как тестировать, и с вашими проектными решениями.

Предлагаемая структура:

```text
code/p16-shortener/
├── Cargo.toml
├── README.md
├── Dockerfile
├── compose.yaml
├── otel-collector.yaml
├── .env                  ← ADMIN_TOKEN для Compose; указан в .gitignore
├── migrations/
│   └── 20261118000000_create_links.sql
├── src/
│   ├── main.rs           ← конфигурация, подключения, запуск, остановка
│   ├── lib.rs            ← AppState, app(state) -> Router, обработчики
│   ├── config.rs
│   └── telemetry.rs
└── tests/
    └── api.rs            ← testcontainers + oneshot
```

> 🔒 **Безопасность:** сервис перенаправляет на любой URL, который сохранил администратор, поэтому создавать ссылки могут только администраторы. Никогда не разрешайте анонимное создание без дополнительных проверок: открытые перенаправители любят фишеры. Никогда не кладите `ADMIN_TOKEN` в `compose.yaml`, в Git или в логи.

### Пошаговый план

Для вех M1–M6 запускайте сервис на Mac через `cargo run`, а его базы данных — в Docker. Сначала создайте `compose.yaml` только с двумя базами:

```yaml
services:
  db:
    image: postgres:18
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev-only-password
      POSTGRES_DB: shortener
    ports:
      - "127.0.0.1:15432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d shortener"]
      interval: 5s
      retries: 10
  cache:
    image: redis:8
    ports:
      - "127.0.0.1:16379:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      retries: 10
```

Запустите их и задайте окружение в терминале, где будете запускать сервис:

```bash
docker compose up -d --wait
export DATABASE_URL=postgres://app:dev-only-password@127.0.0.1:15432/shortener
export REDIS_URL=redis://127.0.0.1:16379/
export ADMIN_TOKEN=dev-token-change-me-0123456789abcdef
export TOKEN=$ADMIN_TOKEN
```

`--wait` возвращается, когда обе проверки работоспособности прошли. Порты 15432 и 16379 не конфликтуют с контейнерами из прошлых уроков.

#### M1 — Каркас: конфигурация, проверки, остановка

**Цель:** сервис запускается, загружает и проверяет конфигурацию, отвечает на `/healthz` и `/readyz` и корректно останавливается по Ctrl+C и `SIGTERM`.

**Как действовать:** переиспользуйте `Config`, `from_lookup`, `healthz`, `readyz` и `shutdown_signal` из Урока 16.13 с переменными из контракта. Положите роутер в `pub fn app(state: AppState) -> Router` в `src/lib.rs`; пока `AppState` хранит `PgPool`. Применяйте миграции при запуске через `sqlx::migrate!()` (Урок 16.7). Напишите модульные тесты правил конфигурации (нет `ADMIN_TOKEN`, токен короче 32 символов).

<details>
<summary>Подсказка</summary>

Правило токена — ещё одна проверка в `from_lookup`: прочитав `ADMIN_TOKEN`, верните `ConfigError::Invalid`, если `token.len() < 32`. Не кладите сам токен в значение ошибки: его напечатают.

</details>

**Готово, когда:**

```bash
env -u ADMIN_TOKEN cargo run
```

```text
invalid configuration: environment variable ADMIN_TOKEN is not set
```

```bash
cargo run
```

```text
 INFO p16_shortener: configuration loaded config=Config { listen: 127.0.0.1:8080, public_base_url: "http://127.0.0.1:8080", rate_limit_per_minute: 60, database_url: "<redacted>", redis_url: "<redacted>", admin_token: "<redacted>" }
 INFO p16_shortener: listening on 127.0.0.1:8080
```

Во втором терминале (с экспортированным `TOKEN`, как выше):

```bash
curl -s -w ' %{http_code}\n' http://127.0.0.1:8080/healthz
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/readyz
```

```text
ok 200
200
```

Ctrl+C печатает `shutdown signal received` и `stopped cleanly`. `cargo test` проходит. Коммит: `M1: skeleton with configuration and health checks`.

#### M2 — Ссылки в PostgreSQL: создание и перенаправление

**Цель:** `POST /api/links` сохраняет ссылку (пока без аутентификации), `GET /{code}` перенаправляет со статусом `302`, и каждая ошибка имеет тело JSON из контракта.

**Как действовать:** напишите тип ошибки `ApiError` с вариантом на каждый статус ошибки и `impl IntoResponse` (Урок 16.4). Проверяйте URL крейтом `url` (`cargo add url@2`): разберите его, затем проверьте схему и наличие хоста. Генерируйте коды через `rand` (`cargo add rand@0.10`). Отображайте нарушение уникальности в `409` для своих кодов и повторяйте попытку с новым кодом для сгенерированных.

<details>
<summary>Подсказка</summary>

Случайный код из 7 букв и цифр в `rand` 0.10:

```rust
use rand::RngExt;

fn random_code() -> String {
    rand::rng()
        .sample_iter(rand::distr::Alphanumeric)
        .take(7)
        .map(char::from)
        .collect()
}

fn main() {
    println!("{}", random_code());
}
```

Для нарушения уникальности сопоставляйте `Err(sqlx::Error::Database(db)) if db.is_unique_violation()`, так же как нарушения проверки в Уроке 16.14.

</details>

**Готово, когда:**

```bash
curl -s -X POST http://127.0.0.1:8080/api/links -H 'content-type: application/json' \
  -d '{"url":"https://www.rust-lang.org/learn","code":"learn"}'
curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' http://127.0.0.1:8080/learn
curl -s -w ' %{http_code}\n' -X POST http://127.0.0.1:8080/api/links -H 'content-type: application/json' \
  -d '{"url":"ftp://example.com"}'
curl -s -w ' %{http_code}\n' -X POST http://127.0.0.1:8080/api/links -H 'content-type: application/json' \
  -d '{"url":"https://example.com","code":"learn"}'
curl -s -w ' %{http_code}\n' http://127.0.0.1:8080/nope-nope
```

```text
{"code":"learn","url":"https://www.rust-lang.org/learn","short_url":"http://127.0.0.1:8080/learn"}
302 https://www.rust-lang.org/learn
{"error":"url must be an absolute http or https URL"} 422
{"error":"code already taken"} 409
{"error":"no such link"} 404
```

Без `"code"` в ответе будет случайный код из 7 символов, например `{"code":"ucLz4gv","url":"https://doc.rust-lang.org/book/","short_url":"http://127.0.0.1:8080/ucLz4gv"}`. Коммит: `M2: create links and redirect`.

#### M3 — Аутентификация администратора и статистика переходов

**Цель:** `POST /api/links` и `GET /api/links/{code}/stats` требуют `Authorization: Bearer <ADMIN_TOKEN>`; каждое перенаправление увеличивает `clicks`.

**Как действовать:** достаточно небольшой функции `require_admin(&headers, &state) -> Result<(), ApiError>`, вызываемой в начале двух обработчиков администратора (хорошая альтернатива — экстрактор или middleware axum, Урок 16.8). Сравнивайте токен за постоянное время крейтом `subtle` (`cargo add subtle@2`). Считайте переход запросом `UPDATE links SET clicks = clicks + 1 WHERE code = $1` после успешного поиска.

<details>
<summary>Подсказка</summary>

`subtle::ConstantTimeEq` добавляет срезам байтов метод `ct_eq`: `given.as_bytes().ct_eq(expected.as_bytes()).into()` даёт `bool`. Обычное `==` останавливается на первом отличающемся байте, поэтому его время выдаёт, сколько начальных байтов было верно (Урок 10.9).

</details>

**Готово, когда:**

```bash
curl -s -i -X POST http://127.0.0.1:8080/api/links -H 'content-type: application/json' \
  -d '{"url":"https://example.com"}'
```

```text
HTTP/1.1 401 Unauthorized
content-type: application/json
www-authenticate: Bearer
content-length: 36
date: Wed, 18 Nov 2026 08:18:31 GMT

{"error":"missing or invalid token"}
```

С заголовком `-H "authorization: Bearer $TOKEN"` тот же запрос возвращает `201`. После одного открытия `/learn` с момента её создания:

```bash
curl -s http://127.0.0.1:8080/api/links/learn/stats -H "authorization: Bearer $TOKEN"
```

```text
{"code":"learn","url":"https://www.rust-lang.org/learn","clicks":1}
```

Коммит: `M3: admin token and click statistics`.

#### M4 — Кэш Redis для перенаправлений

**Цель:** перенаправления используют cache-aside с `link:v1:{code}` (Урок 16.10) и сообщают `x-cache`; когда Redis недоступен, перенаправления всё равно работают.

**Как действовать:** добавьте `ConnectionManager` в `AppState`. Создайте его через `ConnectionManager::new_lazy_with_config`, чтобы сервис запускался, даже когда Redis недоступен, и дайте ему короткие тайм-ауты, чтобы мёртвый Redis стоил доли секунды, а не тридцать. Любая ошибка Redis означает `x-cache: unavailable` и поиск в базе. Уже сейчас пометьте вспомогательные функции `#[instrument(skip(state))]`; спаны понадобятся в M6.

<details>
<summary>Подсказка</summary>

Помощник для `src/main.rs`:

```rust
use std::time::Duration;

use redis::aio::{ConnectionManager, ConnectionManagerConfig};

fn cache(url: &str) -> redis::RedisResult<ConnectionManager> {
    let config = ConnectionManagerConfig::new()
        .set_connection_timeout(Some(Duration::from_millis(500)))
        .set_response_timeout(Some(Duration::from_millis(500)))
        .set_number_of_retries(1);
    ConnectionManager::new_lazy_with_config(redis::Client::open(url)?, config)
}
```

Вызывайте его из `run` внутри рантайма Tokio: менеджеру нужен рантайм для фоновой работы.

</details>

**Готово, когда:**

```bash
docker compose exec cache redis-cli FLUSHALL
for i in 1 2; do curl -s -o /dev/null -D - http://127.0.0.1:8080/learn | grep -i x-cache; done
docker compose stop cache
curl -s -o /dev/null -D - http://127.0.0.1:8080/learn | grep -iE '^HTTP|x-cache'
docker compose start cache
```

```text
OK
x-cache: miss
x-cache: hit
HTTP/1.1 302 Found
x-cache: unavailable
```

Запрос при остановленном Redis всё равно получил быстрый ответ (0,19 с на машине курса, далеко от 30-секундного тайм-аута по умолчанию), а сервис записал в лог два предупреждения: `rate limiter unavailable, allowing request: …` (из M5, когда он у вас будет) и `cache read failed: …`. Через несколько секунд после `docker compose start cache` перенаправления снова `hit` или `miss`, без перезапуска сервиса. Коммит: `M4: Redis cache for redirects`.

#### M5 — Ограничение частоты запросов

**Цель:** не больше `RATE_LIMIT_PER_MINUTE` запросов с одного IP клиента в минуту; следующие получают `429` с `Retry-After`.

**Как действовать:** напишите ограничитель как middleware через `axum::middleware::from_fn_with_state` и добавьте его через `route_layer` к роутеру со всеми маршрутами, кроме проверок работоспособности; затем подключите маршруты проверок через `merge`. Адрес клиента получайте экстрактором `ConnectInfo<SocketAddr>`; для этого запускайте сервер с `app(state).into_make_service_with_connect_info::<SocketAddr>()`. Используйте атомарный конвейер из Упражнения 3 Урока 16.10 и добавьте команду `TTL`, чтобы узнать секунды для `Retry-After`.

<details>
<summary>Подсказка</summary>

В тестах настоящего соединения нет, поэтому `ConnectInfo` отсутствует, и middleware упало бы. У axum есть помощник ровно для этого: добавьте `.layer(axum::extract::connect_info::MockConnectInfo(SocketAddr::from(([10, 0, 0, 1], 5000))))` к роутеру в тестах.

</details>

**Готово, когда:** подождите минуту после последнего запроса (чтобы началось новое окно), затем:

```bash
for i in $(seq 65); do curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8080/learn; done | sort | uniq -c
curl -s -i http://127.0.0.1:8080/learn | grep -iE '^HTTP|retry-after'
```

```text
  60 302
   5 429
HTTP/1.1 429 Too Many Requests
retry-after: 59
```

(Значение `Retry-After` — число секунд, оставшихся в окне, поэтому у вас оно может быть чуть меньше.) Коммит: `M5: rate limiting per client`.

#### M6 — Трассировки

**Цель:** каждый запрос — трассировка в Jaeger со спанами из контракта.

**Как действовать:** скопируйте `telemetry.rs` из Урока 16.12 (трассировок достаточно; метрики — дополнительная цель) и добавьте `TraceLayer` со спанами уровня `INFO`. Добавьте сервисы коллектора и Jaeger из Урока 16.12 в `compose.yaml`, пока опубликуйте порт коллектора `127.0.0.1:4317:4317` и снова выполните `docker compose up -d --wait`. Вызовите `shutdown` у провайдера трассировщика перед возвратом из `main`.

<details>
<summary>Подсказка</summary>

Если спаны `cache_get` и `db_find_link` появляются отдельными трассировками, а не под `redirect`, функции вызываются вне спана `redirect`, например из задачи `tokio::spawn`: запущенная задача не наследует текущий спан, если не прикрепить его через `.instrument(Span::current())` (Урок 09.10).

</details>

**Готово, когда:** после одного перенаправления с пустым кэшем `curl -s http://127.0.0.1:16686/api/v3/services` показывает ваш сервис:

```text
{"services":["p16-shortener","jaeger"]}
```

а в интерфейсе Jaeger (<http://localhost:16686>, сервис `p16-shortener`) трассировка `GET /learn` выглядит так (длительности из одного запуска на машине курса):

```text
request             1.74 ms
  redirect          1.51 ms
    cache_get       0.12 ms
    db_find_link    0.74 ms
    cache_put       0.09 ms
    db_count_click  0.54 ms
```

Коммит: `M6: OpenTelemetry traces`.

#### M7 — Образ контейнера и весь стек в Compose

**Цель:** `docker compose up -d --build` запускает PostgreSQL, Redis, коллектор, Jaeger и сервис; сервис ждёт здоровых баз и сам здоров.

**Как действовать:** напишите `Dockerfile`, как в Уроке 16.13 (distroless `nonroot`, режим `healthcheck`, кэширование зависимостей из его Упражнения 3). Не забудьте скопировать `migrations/` в этап сборки: `sqlx::migrate!()` читает их при компиляции. В `compose.yaml` добавьте сервис `app` с `build: .`, окружением из контракта с именами хостов `db`, `cache` и `collector`, портом `127.0.0.1:8080:8080` и `depends_on` с `condition: service_healthy` для `db` и `cache`. Токен читайте из файла `.env` рядом с `compose.yaml` (`ADMIN_TOKEN: ${ADMIN_TOKEN:?set ADMIN_TOKEN in .env}`) и добавьте `.env` в `.gitignore`. Сначала остановите сервис из `cargo run`: обоим нужен порт 8080.

<details>
<summary>Подсказка</summary>

Внутри Compose адрес клиента, который видит ограничитель, — это адрес шлюза сети Docker, поэтому все запросы с вашего Mac делят один счётчик. Здесь это нормально; за настоящим балансировщиком нагрузки вы читали бы адрес клиента из доверенного заголовка `X-Forwarded-For` (Урок 15.14) и никогда — из недоверенного.

</details>

**Готово, когда:**

```bash
docker compose up -d --build
docker compose ps --format 'table {{.Service}}\t{{.Status}}'
```

```text
SERVICE     STATUS
app         Up 20 seconds (healthy)
cache       Up 25 seconds (healthy)
collector   Up 25 seconds
db          Up 25 seconds (healthy)
jaeger      Up 25 seconds
```

Проверки `curl` из M2–M6 дают те же результаты против контейнера. `docker compose stop app` заканчивается гораздо быстрее секунды, а `docker compose logs app` заканчивается строками `shutdown signal received` и `stopped cleanly`. Коммит: `M7: Docker image and Compose stack`.

#### M8 — Самодостаточные интеграционные тесты и README

**Цель:** `cargo test` сам запускает свои PostgreSQL и Redis через testcontainers (Урок 16.14) и покрывает каждую строку таблицы эндпоинтов; контроль качества проходит; README объясняет проект.

**Как действовать:** помощник `test_app(rate_limit)` запускает контейнеры `postgres:18` и `redis:8`, применяет миграции, собирает `AppState` и возвращает роутер (с `MockConnectInfo`) вместе с обоими контейнерами. Напишите как минимум такие тесты: нет токена или неверный токен → `401`; создание → перенаправление с `x-cache` сначала `miss`, потом `hit`; неверный ввод → `422`; свой код дважды → `201`, потом `409`; три перенаправления → `clicks == 3`; неизвестный код → `404`; с лимитом 3 четыре запроса → `404, 404, 404, 429`.

<details>
<summary>Подсказка</summary>

Redis печатает `Ready to accept connections` в стандартный вывод, когда запустился, поэтому ждите его через `WaitFor::message_on_stdout("Ready to accept connections")`. Возвращайте структуру с полями `_db` и `_cache` для контейнеров, чтобы они жили столько же, сколько тест.

</details>

**Готово, когда:**

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test --test api
```

```text
running 7 tests
test unknown_code_is_404 ... ok
test created_link_redirects_and_is_cached ... ok
test creating_needs_the_admin_token ... ok
test too_many_requests_get_429 ... ok
test stats_count_clicks ... ok
test custom_code_can_be_taken_only_once ... ok
test bad_input_is_rejected ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 7.74s
```

В README есть: что делает сервис; как его запустить (`.env`, `docker compose up -d --build`); как его тестировать; таблица эндпоинтов; и хотя бы три проектных решения, например «302 вместо 301, чтобы считать переходы», «кэш необязателен: ошибки Redis уходят в PostgreSQL», «ограничитель при сбое пропускает запросы». Коммит: `M8: integration tests and README`. Остановите всё командой `docker compose down`.

### Дополнительные цели

1. **Ссылки со сроком действия:** необязательное поле `expires_at` в запросе (новая миграция); просроченные ссылки отвечают `410 Gone`, а TTL кэша никогда не длиннее оставшегося срока.
2. **Более дешёвый подсчёт переходов:** считайте переходы через `INCR clicks:v1:{code}` в Redis, а фоновая задача пусть каждые несколько секунд добавляет счёт в PostgreSQL. Запишите, что теряется при падении Redis и почему это может быть допустимо.
3. **Метрики:** счётчик `shortener.redirects` с атрибутом `cache` (`hit`, `miss`, `unavailable`) и гистограмма длительности запросов (Урок 16.12), экспортируемые через коллектор.
4. **Кэширование отсутствия:** кэшируйте «такого кода нет» на 30 секунд, чтобы перебор случайных кодов не бил каждый раз в PostgreSQL. Убедитесь, что создание этого кода позже удаляет закэшированное «не найдено».
5. **API администратора на gRPC:** те же операции создания и статистики как сервис tonic на втором порту (Урок 16.9), с общим кодом работы с базой для обработчиков REST.
6. **CI (Continuous Integration, непрерывная интеграция):** рабочий процесс GitHub Actions, который запускает контроль качества и тесты testcontainers (Урок 09.12); проверьте его через `actionlint`.

## Вопросы для самопроверки

1. Почему сервис отвечает `302 Found`, а не `301 Moved Permanently`?

<details>
<summary>Ответ</summary>

Браузеры запоминают постоянные перенаправления и в следующий раз идут прямо на цель, так что сервис никогда не увидел бы эти переходы. `302` временный: каждый визит возвращается и может быть посчитан (и цель даже можно поменять).

</details>

2. Redis недоступен. Что происходит с перенаправлениями, с ограничителем частоты и с заголовком `x-cache`?

<details>
<summary>Ответ</summary>

Перенаправления работают: URL берётся из PostgreSQL, а `x-cache` равен `unavailable`. Ограничитель пропускает запрос и пишет предупреждение («отказ в открытую сторону», fail open). Сервис медленнее, но не падает.

</details>

3. Почему токен администратора нужно сравнивать за постоянное время?

<details>
<summary>Ответ</summary>

Обычное сравнение останавливается на первом отличающемся байте. Измеряя время ответов, злоумышленник мог бы узнать, сколько начальных байтов догадки верны, и найти токен байт за байтом. Сравнение за постоянное время всегда длится одинаково.

</details>

4. Почему проверки работоспособности находятся вне ограничителя частоты?

<details>
<summary>Ответ</summary>

Docker (а позже Kubernetes) вызывает их каждые несколько секунд. Если бы их ограничивали, в нагруженную минуту проверка могла бы не пройти, и оркестратор перезапустил бы совершенно здоровый сервис.

</details>

## Итоги

- Настоящий сервис соединяет все части этого этапа; письменный контракт делает их согласованными и проверяемыми.
- PostgreSQL — источник истины; Redis только ускоряет работу и защищает базу, поэтому у каждого сбоя Redis есть запасной путь.
- Аутентификация защищает эндпоинты, которые могут навредить; ограничение частоты защищает сервис от слишком большого числа запросов; ни то ни другое не касается проверок работоспособности.
- Трассировки показывают путь каждого запроса через кэш и базу; корректное завершение гарантирует отправку последних спанов.
- Интеграционные тесты с testcontainers доказывают весь контракт с настоящими базами на любой машине с Docker.
- Каждая веха заканчивается приёмочными тестами и коммитом; проектные решения идут в README.

## Источники и что почитать

- [MDN — Redirections in HTTP](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Redirections) — постоянные и временные перенаправления и как с ними обращаются браузеры.
- [RFC 6585 — Additional HTTP Status Codes](https://www.rfc-editor.org/rfc/rfc6585) — определение `429 Too Many Requests` и `Retry-After`.
- [OWASP — Unvalidated Redirects and Forwards Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Unvalidated_Redirects_and_Forwards_Cheat_Sheet.html) — почему открытые перенаправления опасны.
- [axum — документация middleware на docs.rs](https://docs.rs/axum/latest/axum/middleware/index.html) — `from_fn_with_state`, `route_layer` и порядок слоёв.
- [subtle на docs.rs](https://docs.rs/subtle) — сравнение секретов за постоянное время.
- [url на docs.rs](https://docs.rs/url) — разбор и проверка URL.

---

[🏠 Начало](../START-HERE.md) · [← 16.14 Тестирование бэкенд-сервиса](14-testing-a-backend-service.md) · [📋 Этап 16](README.md) · [16.16 Контрольная точка →](16-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
