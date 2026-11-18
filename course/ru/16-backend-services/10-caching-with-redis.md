<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 16.9 gRPC на tonic](09-grpc-with-tonic.md) · [📋 Этап 16](README.md) · [16.11 Обмен сообщениями через NATS →](11-messaging-with-nats.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/16-backend-services/10-caching-with-redis.md) <!-- nav -->

# Урок 16.10 — Кэширование с Redis

> **Этап 16 — Бэкенд-сервисы** · Урок 10 из 16 · ⏱ около 120 минут

## Что вы узнаете

- Что такое кэш, зачем сервисам кэш и что с ним может пойти не так.
- Как запустить Redis в Docker и пользоваться его основными командами: `SET`, `GET`, `EX`, `TTL`, `INCR`, `DEL`.
- Паттерн cache-aside на Rust с крейтом `redis` и `ConnectionManager`.
- Как держать кэш верным с помощью времени жизни ключей и инвалидации и как пережить отказ Redis.
- Как построить небольшой ограничитель частоты запросов на счётчике Redis и как тестировать код, которому нужен Redis.

## Перед началом

- Вы прошли [Урок 16.9 — gRPC на tonic](09-grpc-with-tonic.md) и уроки о базах данных [16.5](05-sql-and-postgresql-basics.md) и [16.6](06-databases-with-sqlx.md).
- Docker запущен, и вы умеете запускать контейнер ([Урок 11.12 — Контейнеры и основы Docker](../11-linux-shell-and-containers/12-containers-and-docker-basics.md)).
- Вы знаете `async`/`await` с Tokio ([Урок 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)) и `serde` для JSON (JavaScript Object Notation, текстовый формат данных; Этап 05).

## Новые слова

| Слово | Значение |
|---|---|
| **кэш** (cache) | Быстрое хранилище с копиями данных, которые медленно или дорого получать, чтобы следующий запрос получил ответ быстро. |
| **попадание в кэш / промах кэша** (cache hit / cache miss) | Данные были в кэше / их не было, и пришлось спрашивать медленный источник. |
| **Redis** (REmote DIctionary Server, удалённый сервер-словарь) | Популярный сервер, который хранит данные в памяти как ключи и значения. Используется для кэшей, счётчиков, сессий и очередей. |
| **в памяти** (in-memory) | Данные живут в RAM (Random Access Memory, оперативная память), а не на диске, поэтому чтение и запись очень быстрые. Если сервер остановится, данные могут пропасть. |
| **ключ / значение** (key / value) | Имя, под которым хранятся данные (`price:v1:42`), и сами данные (`4299`). |
| **TTL** (Time To Live, время жизни) | Сколько секунд ещё живёт ключ, прежде чем Redis удалит его сам. |
| **cache-aside** | Паттерн: сервис сначала смотрит в кэш; при промахе читает базу данных и кладёт результат в кэш. |
| **инвалидация** (invalidation) | Удаление или обновление копии в кэше, потому что настоящие данные изменились. |
| **устаревшие данные** (stale data) | Копия в кэше, которая старше настоящих данных. |
| **менеджер соединения** (connection manager) | Соединение с Redis из крейта `redis`, которое само переподключается и которое можно делить, клонируя. |
| **ограничение частоты запросов** (rate limiting) | Разрешение клиенту только определённого числа запросов за окно времени, чтобы защитить сервис. |
| **атомарный** (atomic) | Выполняемый как один шаг: никакой другой клиент не может увидеть или изменить что-то посередине. |
| **конвейер команд** (pipeline) | Несколько команд, отправленных в Redis вместе; с `MULTI`/`EXEC` они выполняются атомарно. |

## 1. Зачем нужен кэш?

Представьте сервис магазина. Каждая страница товара спрашивает цену у базы данных. Запрос к базе длится 200 мс (миллисекунд), а одни и те же 1 000 популярных товаров запрашиваются тысячи раз в минуту. База снова и снова делает одну и ту же работу.

**Кэш** — маленькое быстрое хранилище перед медленным источником. Он хранит *копии*:

```text
              1. GET price:v1:42
  service ──────────────────────────► Redis (memory, ~0.2 ms)
     │        2a. hit: "4299" ◄──────
     │
     │ 2b. miss: ask the database (~200 ms)
     ▼
  PostgreSQL ──► 3. store the copy in Redis with a TTL of 60 s
```

Кэш ускоряет чтение и снимает нагрузку с базы данных. Но он приносит новую проблему: копия может быть **устаревшей**, старше правды. Любое решение с кэшем отвечает на два вопроса:

1. **Насколько старой может быть копия?** Вы выбираете **TTL** (Time To Live, время жизни): через 60 секунд копия исчезает сама. Короткий TTL — более свежие данные и больше чтений из базы.
2. **Что происходит, когда данные меняются?** Вы **инвалидируете** копию: удаляете её, чтобы следующее чтение получило новое значение.

> 💡 **Совет:** кэшируйте только данные, которые читают гораздо чаще, чем меняют, и для которых допустимо отставание на несколько секунд: цены на странице товара, профиль пользователя, результат медленного отчёта. Не кэшируйте данные, которые должны быть точными именно в этот момент, например баланс счёта перед платежом.

## 2. Redis в Docker

**Redis** (REmote DIctionary Server) — это сервер «ключ — значение» **в памяти**: все данные живут в RAM (Random Access Memory, оперативная память), поэтому команда занимает доли миллисекунды. Запустите его из официального образа:

```bash
docker run -d --name p16-redis -p 127.0.0.1:6379:6379 redis:8
```

**Что происходит:**

1. `-d` запускает контейнер в фоне, `--name p16-redis` даёт ему имя для следующих команд.
2. `-p 127.0.0.1:6379:6379` делает порт 6379 (стандартный порт Redis) доступным **только с вашего Mac**.
3. `redis:8` — официальный образ Redis версии 8.

> 🔒 **Безопасность:** у этого Redis нет пароля. Никогда не открывайте его на всех сетевых интерфейсах (`-p 6379:6379` без `127.0.0.1`) на сервере: злоумышленники сканируют интернет в поисках открытых портов Redis и могут прочитать или удалить всё. В продакшене у Redis есть пароль (или пользователи ACL, Access Control Lists, списки управления доступом), и он стоит в закрытой сети.

В образе есть клиент командной строки `redis-cli`. Откройте его внутри контейнера:

```bash
docker exec -it p16-redis redis-cli
```

Вводите команды после приглашения `127.0.0.1:6379>`:

```text
127.0.0.1:6379> PING
PONG
127.0.0.1:6379> SET greeting hello
OK
127.0.0.1:6379> GET greeting
"hello"
127.0.0.1:6379> GET missing
(nil)
127.0.0.1:6379> SET session:abc user42 EX 30
OK
127.0.0.1:6379> TTL session:abc
(integer) 30
127.0.0.1:6379> TTL greeting
(integer) -1
127.0.0.1:6379> INCR visits
(integer) 1
127.0.0.1:6379> INCR visits
(integer) 2
127.0.0.1:6379> DEL greeting
(integer) 1
127.0.0.1:6379> GET greeting
(nil)
127.0.0.1:6379> TTL nothing
(integer) -2
```

**Что происходит:**

1. `SET key value` сохраняет значение, `GET key` читает его. `(nil)` означает «такого ключа нет». Значения — строки (байты).
2. `SET ... EX 30` сохраняет значение, которое истечёт через 30 секунд. `TTL` показывает, сколько секунд осталось. `-1` значит «ключ никогда не истекает», `-2` — «такого ключа нет».
3. `INCR visits` прибавляет 1 к числу и возвращает новое значение. Отсутствующий ключ считается за 0. Redis выполняет по одной команде за раз, поэтому два клиента, вызвавшие `INCR` в один момент, никогда не теряют счёт: команда **атомарна**.
4. `DEL` удаляет ключи и возвращает, сколько удалил.

Наберите `exit`, чтобы выйти из `redis-cli`. Кроме строк, в Redis есть и другие типы данных; чаще всего вы встретите эти:

| Тип | Примеры команд | Типичное применение |
|---|---|---|
| Строка | `SET`, `GET`, `INCR` | Значения в кэше, счётчики |
| Хеш | `HSET user:1 name Ann`, `HGET` | Небольшая запись с полями |
| Список | `LPUSH`, `RPOP` | Простая очередь |
| Множество | `SADD`, `SISMEMBER` | «Голосовал ли этот пользователь?» |
| Сортированное множество | `ZADD`, `ZRANGE` | Таблицы лидеров, данные по времени |

> 📏 **Соглашение:** называйте ключи `kind:version:id` (вид:версия:id), например `price:v1:42`. Префикс показывает, что лежит в ключе, а версия позволяет позже сменить формат: новый код читает `price:v2:...`, а старые ключи `v1` просто истекают.

## 3. Cache-aside на Rust

Создайте проект и добавьте крейты:

```bash
cd ~/projects/rust/code
cargo new p16-redis-cache
cd p16-redis-cache
cargo add redis@1 --features tokio-comp,connection-manager
cargo add tokio@1 --features macros,rt-multi-thread,time
```

```toml
[dependencies]
redis = { version = "1", features = ["tokio-comp", "connection-manager"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread", "time"] }
```

Фича `tokio-comp` позволяет `redis` работать на Tokio, а `connection-manager` даёт нам `ConnectionManager`: асинхронное соединение, которое **само переподключается** после сетевой ошибки и которое дёшево клонировать, так что его могут делить многие задачи.

Положите логику кэша в `src/lib.rs`, чтобы её могли использовать тесты:

```rust
use std::time::Duration;

use redis::AsyncTypedCommands;
use redis::aio::ConnectionManager;

/// Сколько времени цена в кэше остаётся действительной.
pub const PRICE_TTL_SECONDS: u64 = 60;

/// Ключ кэша для одного товара. `v1` позволяет позже сменить формат.
pub fn price_key(product_id: u32) -> String {
    format!("price:v1:{product_id}")
}

/// Ненастоящая база данных: специально медленная.
pub async fn load_price_from_db(product_id: u32) -> u32 {
    tokio::time::sleep(Duration::from_millis(200)).await;
    product_id * 100 + 99
}

/// Cache-aside: пробуем кэш, при промахе спрашиваем базу и заполняем кэш.
pub async fn get_price(cache: &mut ConnectionManager, product_id: u32) -> (u32, &'static str) {
    let key = price_key(product_id);
    match cache.get(&key).await {
        Ok(Some(text)) => {
            if let Ok(price) = text.parse() {
                return (price, "cache");
            }
        }
        Ok(None) => {}
        Err(err) => eprintln!("cache read failed: {err}"),
    }

    let price = load_price_from_db(product_id).await;
    if let Err(err) = cache.set_ex(&key, price, PRICE_TTL_SECONDS).await {
        eprintln!("cache write failed: {err}");
    }
    (price, "database")
}

/// Вызывайте после того, как цена изменилась в базе данных.
pub async fn invalidate_price(
    cache: &mut ConnectionManager,
    product_id: u32,
) -> redis::RedisResult<()> {
    cache.del(price_key(product_id)).await?;
    Ok(())
}
```

**Что происходит:**

1. `AsyncTypedCommands` — трейт, который добавляет соединению по асинхронному методу на каждую команду Redis с фиксированным типом результата: `get` возвращает `Option<String>`, `set_ex` — `()`, `del` — число удалённых ключей.
2. `load_price_from_db` заменяет настоящий запрос (Урок 16.6). Она спит 200 мс, чтобы была видна разница.
3. `get_price` — это **cache-aside**. Шаг первый: `GET`. Если значение есть и разбирается как число, это попадание, и мы сразу возвращаем его.
4. При промахе мы читаем «базу данных» и сохраняем значение через `set_ex` (в Redis это `SET key value EX 60`). Следующие 60 секунд запросы будут попаданиями.
5. **Ошибка кэша — не ошибка запроса.** Если Redis недоступен, `get` возвращает `Err`; мы печатаем предупреждение и идём в базу. Сервис становится медленнее, но продолжает работать. Кэш — это оптимизация, а не источник истины.
6. Если текст в кэше не разбирается (например, старый формат), мы тоже идём в базу и перезаписываем его.
7. `invalidate_price` удаляет копию. Вызывайте её сразу после успешной записи в базу.

Теперь `src/main.rs`:

```rust
use std::time::Instant;

use p16_redis_cache::{get_price, invalidate_price};

#[tokio::main]
async fn main() -> redis::RedisResult<()> {
    let url = std::env::var("REDIS_URL").unwrap_or("redis://127.0.0.1:6379/".to_string());
    let client = redis::Client::open(url)?;
    let mut cache = client.get_connection_manager().await?;

    for _ in 0..3 {
        let start = Instant::now();
        let (price, source) = get_price(&mut cache, 42).await;
        println!(
            "price={price} source={source} took={}ms",
            start.elapsed().as_millis()
        );
    }

    invalidate_price(&mut cache, 42).await?;
    let (price, source) = get_price(&mut cache, 42).await;
    println!("after invalidation: price={price} source={source}");
    Ok(())
}
```

```bash
cargo run
```

```text
price=4299 source=database took=203ms
price=4299 source=cache took=0ms
price=4299 source=cache took=0ms
after invalidation: price=4299 source=database
```

**Что происходит:**

1. Адрес Redis берётся из переменной окружения `REDIS_URL`, со значением по умолчанию для вашего ноутбука. Это правило двенадцати факторов «конфигурация в окружении» из [Урока 08.15](../08-software-design/15-the-twelve-factor-app.md); на нём строится Урок 16.13.
2. `Client::open` только проверяет URL (Uniform Resource Locator, адрес). `get_connection_manager().await` действительно подключается.
3. Первое чтение — промах, оно занимает около 200 мс. Следующие два — попадания, они занимают меньше миллисекунды (ваши времена будут немного другими).
4. После `invalidate_price` ключа нет, поэтому следующее чтение снова идёт в базу и снова заполняет кэш.

Посмотрите на ключ в кэше через `redis-cli` (сначала ещё раз выполните `cargo run`, чтобы он существовал):

```text
127.0.0.1:6379> GET price:v1:42
"4299"
127.0.0.1:6379> TTL price:v1:42
(integer) 60
```

> ⚡ **Производительность:** обращение к Redis на той же машине стоит примерно 0,1–0,5 мс; через сеть дата-центра — всё ещё около 1 мс. Кэш окупается, когда источник намного медленнее. Кэшировать то, что вычисляется за 0,3 мс, — только добавлять работу.

## 4. Что может пойти не так с кэшем

Кэш легко добавить и легко сделать неправильно. Классические проблемы:

| Проблема | Что происходит | Обычное решение |
|---|---|---|
| **Устаревшие данные** | База изменилась, а в кэше старая копия. | Инвалидировать при каждой записи; держать TTL достаточно коротким. |
| **Порядок записи** | Вы удаляете ключ кэша, затем другой запрос читает *старое* значение из базы и снова кэширует его, а потом происходит ваша запись в базу. | Сначала пишите в базу, **потом** удаляйте ключ. Оставшийся вред ограничивает TTL. |
| **Лавина запросов** (stampede) | Популярный ключ истёк; 500 запросов одновременно промахиваются и все идут в базу. | Добавляйте к каждому TTL немного случайного времени, чтобы ключи не истекали вместе; позволяйте заполнять ключ только одному запросу. |
| **Неограниченная память** | Каждый ключ хранится вечно, и у Redis кончается RAM. | Всегда задавайте TTL ключам кэша; в продакшене настройте `maxmemory` с политикой вытеснения. |
| **Кэширование ошибок** | Неудачный поиск («не найдено», потому что база была недоступна) кэшируется на 60 с. | Кэшируйте только успешные результаты или «не найдено» с гораздо более коротким TTL. |

> 🔒 **Безопасность:** никогда не кладите данные конкретного пользователя под ключ без id пользователя. Если два пользователя делят `profile:v1:me`, один из них увидит данные другого. Ключи, построенные из пользовательского ввода, тоже нужно проверять (Урок 10.4), чтобы никто не смог читать ключи `price:v1:*` других видов.

## 5. Счётчик с окном времени: ограничение частоты запросов

`INCR` и `EXPIRE` вместе дают простой **ограничитель частоты запросов**: «не больше 3 запросов за 10 секунд на клиента». Первый запрос в окне создаёт счётчик и задаёт ему TTL; когда TTL кончается, счётчик исчезает и начинается новое окно. Добавьте в `src/lib.rs`:

```rust
/// Ограничение с фиксированным окном: не больше `limit` вызовов за `window_seconds` на клиента.
pub async fn allow_request(
    cache: &mut ConnectionManager,
    client_id: &str,
    limit: u64,
    window_seconds: i64,
) -> redis::RedisResult<bool> {
    let key = format!("rate:v1:{client_id}");
    let count = cache.incr(&key, 1).await?;
    if count == 1 {
        cache.expire(&key, window_seconds).await?;
    }
    Ok(count as u64 <= limit)
}
```

И вторая программа, `src/bin/ratelimit.rs`:

```rust
use p16_redis_cache::allow_request;

#[tokio::main]
async fn main() -> redis::RedisResult<()> {
    let url = std::env::var("REDIS_URL").unwrap_or("redis://127.0.0.1:6379/".to_string());
    let client = redis::Client::open(url)?;
    let mut cache = client.get_connection_manager().await?;

    for i in 1..=5 {
        let allowed = allow_request(&mut cache, "alice", 3, 10).await?;
        println!("request {i}: {}", if allowed { "allowed" } else { "blocked" });
    }
    Ok(())
}
```

```bash
cargo run --bin ratelimit
```

```text
request 1: allowed
request 2: allowed
request 3: allowed
request 4: blocked
request 5: blocked
```

**Что происходит:**

1. `incr` возвращает новое значение счётчика. Поскольку `INCR` атомарна, две копии вашего сервиса, работающие одновременно, всё равно считают верно. Поэтому счётчик живёт в Redis, а не в памяти одного процесса.
2. Только первый запрос окна задаёт TTL через `expire`.
3. Запросы 4 и 5 превышают лимит. В веб-сервисе вы ответили бы на них статусом HTTP (Hypertext Transfer Protocol, протокол передачи гипертекста) `429 Too Many Requests`. Запустите программу снова через 10 секунд — и первые три снова будут разрешены.

Есть одна слабость: если сервис упадёт между `incr` и `expire`, у ключа не будет TTL, и клиент останется заблокированным навсегда. Упражнение 3 это исправляет.

> 🧪 **Тестирование:** код, который общается с Redis, по-настоящему проверяется только с настоящим Redis. Держите такие тесты отдельно, как показано в «Попробуйте сами»; Урок 16.14 показывает, как тест может сам запустить контейнер Redis.

## Частые ошибки

> ⚠️ **Частая ошибка:** Redis не запущен. `get_connection_manager()` падает при старте:

```text
Error: Connection refused (os error 61)
```

Исправление: `docker start p16-redis` (или команда `docker run` из раздела 2). Упасть при старте — нормально; а вот отказывать в *запросах* позже из-за недоступного кэша — нет (раздел 3, пункт 5).

> ⚠️ **Частая ошибка:** `set` вместо `set_ex` для записей кэша. Ключ никогда не истекает (`TTL` показывает `-1`), старые данные остаются навсегда, и память растёт. Каждому ключу кэша нужен TTL.

> ⚠️ **Частая ошибка:** удалять ключ кэша *до* записи в базу. Запрос, пришедший в промежутке, снова закэширует старое значение. Сначала запись, потом инвалидация.

## Попробуйте сами

1. Запустите Redis и выполните `cargo run` дважды. Во втором запуске все три чтения — попадания. Почему? (Ключ из первого запуска ещё жив.)
2. В `redis-cli` выполните `TTL price:v1:42` несколько раз и посмотрите, как он уменьшается.
3. Положите тесты в `tests/cache.rs`: обычный тест формата ключа и тест, которому нужен Redis, помеченный `#[ignore]` с причиной:

```rust
use p16_redis_cache::{get_price, price_key};

#[test]
fn key_has_prefix_version_and_id() {
    assert_eq!(price_key(7), "price:v1:7");
}

#[tokio::test]
#[ignore = "needs a running Redis"]
async fn second_read_comes_from_cache() {
    let url = std::env::var("REDIS_URL").unwrap_or("redis://127.0.0.1:6379/".to_string());
    let client = redis::Client::open(url).unwrap();
    let mut cache = client.get_connection_manager().await.unwrap();
    let id = 900_001; // id, которого не использует ни один другой тест
    p16_redis_cache::invalidate_price(&mut cache, id).await.unwrap();

    let first = get_price(&mut cache, id).await;
    let second = get_price(&mut cache, id).await;

    assert_eq!(first.1, "database");
    assert_eq!(second, (first.0, "cache"));
}
```

`cargo test --test cache` пропускает тест с Redis:

```text
running 2 tests
test second_read_comes_from_cache ... ignored, needs a running Redis
test key_has_prefix_version_and_id ... ok

test result: ok. 1 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Когда Redis запущен, `cargo test --test cache -- --ignored` выполняет его:

```text
running 1 test
test second_read_comes_from_cache ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out; finished in 0.21s
```

Тест использует свой id товара и сначала удаляет ключ, поэтому даёт один и тот же результат при каждом запуске и не мешает другим тестам.

## Упражнения

### Упражнение 1 — Тест инвалидации (легко)

Напишите игнорируемый тест `invalidation_forces_a_database_read`: заполните кэш для одного id, инвалидируйте его, прочитайте снова и проверьте, что источник — `"database"`.

<details>
<summary>Подсказка</summary>

Скопируйте строки подключения из `second_read_comes_from_cache`. Используйте другой id, например `900_002`, чтобы два теста никогда не трогали один и тот же ключ.

</details>

<details>
<summary>Ответ</summary>

```rust
#[tokio::test]
#[ignore = "needs a running Redis"]
async fn invalidation_forces_a_database_read() {
    let url = std::env::var("REDIS_URL").unwrap_or("redis://127.0.0.1:6379/".to_string());
    let client = redis::Client::open(url).unwrap();
    let mut cache = client.get_connection_manager().await.unwrap();
    let id = 900_002;

    get_price(&mut cache, id).await; // заполняет кэш
    p16_redis_cache::invalidate_price(&mut cache, id)
        .await
        .unwrap();
    let (_, source) = get_price(&mut cache, id).await;

    assert_eq!(source, "database");
}
```

Почему это работает: после первого вызова ключ существует; `invalidate_price` выполняет `DEL`, поэтому третий шаг — промах. Если забыть `DEL` в `invalidate_price`, тест упадёт — именно от этого он и защищает.

</details>

### Упражнение 2 — Кэшировать целую структуру как JSON (средне)

Странице товара нужна не только цена. Добавьте `serde` (с `derive`) и `serde_json`, определите `Product { id: u32, name: String, price: u32 }` и напишите `get_product(cache, id) -> Product` по схеме cache-aside. Храните значение как строку JSON под ключом `product:v1:<id>`.

<details>
<summary>Подсказка</summary>

`serde_json::to_string(&product)` даёт `String`, которую можно передать в `set_ex`. При попадании `serde_json::from_str(&json)` превращает её обратно. Если декодирование не удалось, считайте это промахом.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(serde::Serialize, serde::Deserialize, Debug, Clone, PartialEq)]
pub struct Product {
    pub id: u32,
    pub name: String,
    pub price: u32,
}

pub async fn load_product_from_db(id: u32) -> Product {
    tokio::time::sleep(Duration::from_millis(200)).await;
    Product {
        id,
        name: format!("product {id}"),
        price: id * 100 + 99,
    }
}

pub async fn get_product(cache: &mut ConnectionManager, id: u32) -> Product {
    let key = format!("product:v1:{id}");
    if let Ok(Some(json)) = cache.get(&key).await
        && let Ok(product) = serde_json::from_str(&json)
    {
        return product;
    }
    let product = load_product_from_db(id).await;
    if let Ok(json) = serde_json::to_string(&product)
        && let Err(err) = cache.set_ex(&key, json, PRICE_TTL_SECONDS).await
    {
        eprintln!("cache write failed: {err}");
    }
    product
}
```

После одного вызова `GET product:v1:5` в `redis-cli` показывает `"{\"id\":5,\"name\":\"product 5\",\"price\":599}"`.

Почему это работает: Redis хранит байты, и ему всё равно, что они значат; JSON — просто удобный текстовый формат для структуры. Цепочки `let` (`if let ... && let ...`) делают путь «попадания» коротким, а всё неожиданное (ошибка Redis, старый или сломанный JSON) уходит в базу.

</details>

### Упражнение 3 — Ограничитель, который не может застрять (сложно)

Исправьте слабость `allow_request`: счётчик и его TTL должны задаваться одним атомарным шагом, а TTL — только когда у ключа его ещё нет. Используйте конвейер команд Redis с `atomic()` (он оборачивает команды в `MULTI` … `EXEC`) и команду `EXPIRE key seconds NX` (`NX`: только если у ключа ещё нет TTL).

<details>
<summary>Подсказка</summary>

`redis::pipe().atomic().incr(&key, 1)` начинает конвейер. Добавьте `EXPIRE` через `.cmd("EXPIRE").arg(&key).arg(window_seconds).arg("NX")` и пропустите его результат через `.ignore()`. Тогда `query_async` вернёт кортеж с одним значением: счётчиком.

</details>

<details>
<summary>Ответ</summary>

```rust
pub async fn allow_request_atomic(
    cache: &mut ConnectionManager,
    client_id: &str,
    limit: u64,
    window_seconds: i64,
) -> redis::RedisResult<bool> {
    let key = format!("rate:v1:{client_id}");
    let (count,): (u64,) = redis::pipe()
        .atomic()
        .incr(&key, 1)
        .cmd("EXPIRE")
        .arg(&key)
        .arg(window_seconds)
        .arg("NX")
        .ignore()
        .query_async(cache)
        .await?;
    Ok(count <= limit)
}
```

Пять вызовов с лимитом 3 дают `true, true, true, false, false`, а `TTL rate:v1:<client>` — от 1 до 10.

Почему это работает: `MULTI`/`EXEC` заставляет Redis выполнить обе команды вместе, без других команд между ними, и падение вашего сервиса уже не может их разделить. `NX` сохраняет первый TTL окна, поэтому следующие запросы не отодвигают конец окна.

</details>

## Вопросы для самопроверки

1. Чем попадание в кэш отличается от промаха и что делает cache-aside при промахе?

<details>
<summary>Ответ</summary>

Попадание означает, что значение было в кэше; промах — что его не было. При промахе cache-aside читает настоящий источник (базу данных), сохраняет результат в кэш с TTL и возвращает его.

</details>

2. Redis недоступен. Что должна делать `get_price` и почему?

<details>
<summary>Ответ</summary>

Записать предупреждение в лог и прочитать из базы. Кэш — только оптимизация; без него сервис должен продолжать работать (медленнее).

</details>

3. Зачем каждому ключу кэша TTL?

<details>
<summary>Ответ</summary>

Чтобы устаревшие копии исчезали сами, даже если инвалидацию пропустили, и чтобы память не росла бесконечно.

</details>

4. Цена обновляется. В каком порядке писать в базу и удалять ключ кэша?

<details>
<summary>Ответ</summary>

Сначала база, потом удаление ключа. Если удалить сначала, запрос в промежутке может прочитать старое значение из базы и положить его обратно в кэш.

</details>

5. Почему счётчик ограничения частоты должен жить в Redis, а не в `HashMap` внутри сервиса?

<details>
<summary>Ответ</summary>

Сервис обычно работает в нескольких копиях (процессах или контейнерах). `HashMap` считал бы отдельно в каждой копии. Redis общий для всех копий, а `INCR` атомарна, поэтому счёт верный.

</details>

## Итоги

- Кэш хранит быстрые копии медленных данных. Он ускоряет чтение и защищает базу ценой возможного устаревания данных.
- Redis — сервер «ключ — значение» в памяти. `SET key value EX seconds`, `GET`, `DEL`, `TTL` и атомарная `INCR` покрывают большую часть работы с кэшем.
- Cache-aside: читаем кэш; при промахе читаем базу и заполняем кэш с TTL. Инвалидируем после каждой записи, сначала база.
- Ошибка Redis не должна ронять запрос: запишите её в лог и обратитесь к источнику истины.
- `ConnectionManager` — общее асинхронное соединение, которое само переподключается; адрес берётся из `REDIS_URL`.
- `INCR` плюс TTL дают ограничитель с фиксированным окном; атомарный конвейер с `EXPIRE ... NX` делает его надёжным.

## Источники и что почитать

- [Redis — Commands](https://redis.io/docs/latest/commands/) — все команды, включая параметры `SET`, `INCR`, `EXPIRE` и `TTL`.
- [Redis — Understand Redis data types](https://redis.io/docs/latest/develop/data-types/) — строки, хеши, списки, множества и сортированные множества с примерами.
- [Redis — Transactions](https://redis.io/docs/latest/develop/using-commands/transactions/) — как работает `MULTI`/`EXEC`.
- [Крейт redis на docs.rs](https://docs.rs/redis) — клиент для Rust: `Client`, `ConnectionManager`, `AsyncTypedCommands`, конвейеры команд.
- [Docker Hub — официальный образ redis](https://hub.docker.com/_/redis) — образ, который используется здесь, и как его настраивать.

---

[🏠 Начало](../START-HERE.md) · [← 16.9 gRPC на tonic](09-grpc-with-tonic.md) · [📋 Этап 16](README.md) · [16.11 Обмен сообщениями через NATS →](11-messaging-with-nats.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
