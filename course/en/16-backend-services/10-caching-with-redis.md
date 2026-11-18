<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.9 gRPC with tonic](09-grpc-with-tonic.md) · [📋 Phase 16](README.md) · [16.11 Messaging with NATS →](11-messaging-with-nats.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/10-caching-with-redis.md) <!-- nav -->

# Lesson 16.10 — Caching with Redis

> **Phase 16 — Backend services** · Lesson 10 of 16 · ⏱ about 120 minutes

## What you will learn

- What a cache is, why services use one, and what can go wrong with it.
- How to run Redis in Docker and use its basic commands: `SET`, `GET`, `EX`, `TTL`, `INCR`, `DEL`.
- The cache-aside pattern in Rust with the `redis` crate and a `ConnectionManager`.
- How to keep the cache correct with expiry times and invalidation, and how to survive when Redis is down.
- How to build a small rate limiter on a Redis counter, and how to test code that needs Redis.

## Before you start

- You finished [Lesson 16.9 — gRPC with tonic](09-grpc-with-tonic.md) and the database lessons [16.5](05-sql-and-postgresql-basics.md) and [16.6](06-databases-with-sqlx.md).
- Docker is running, and you can start a container ([Lesson 11.12 — Containers and Docker basics](../11-linux-shell-and-containers/12-containers-and-docker-basics.md)).
- You know `async`/`await` with Tokio ([Lesson 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)) and `serde` for JSON (JavaScript Object Notation, Phase 05).

## New words

| Word | Meaning |
|---|---|
| **cache** | A fast store that keeps copies of data that is slow or expensive to get, so the next request is answered quickly. |
| **cache hit / cache miss** | The data was in the cache / it was not, so we had to ask the slow source. |
| **Redis** (REmote DIctionary Server) | A popular server that keeps data in memory as keys and values. Used for caches, counters, sessions and queues. |
| **in-memory** | The data lives in RAM (Random Access Memory), not on disk, so reads and writes are very fast. If the server stops, the data may be lost. |
| **key / value** | The name you store data under (`price:v1:42`) and the data itself (`4299`). |
| **TTL** (Time To Live) | How many seconds a key still lives before Redis deletes it by itself. |
| **cache-aside** | A pattern: the service first looks in the cache; on a miss it reads the database and puts the result in the cache. |
| **invalidation** | Removing or updating a cached copy because the real data changed. |
| **stale data** | A cached copy that is older than the real data. |
| **connection manager** | A Redis connection from the `redis` crate that reconnects by itself and can be shared by cloning. |
| **rate limiting** | Allowing a client only a certain number of requests in a time window, to protect the service. |
| **atomic** | Done as one step: no other client can see or change anything in the middle. |
| **pipeline** | Several commands sent to Redis together; with `MULTI`/`EXEC` they run atomically. |

## 1. Why a cache?

Imagine a shop service. Every product page asks the database for the price. The database query takes 200 ms (milliseconds), and the same 1,000 popular products are asked for thousands of times per minute. The database does the same work again and again.

A **cache** is a small, fast store in front of the slow source. It keeps *copies*:

```text
              1. GET price:v1:42
  service ──────────────────────────► Redis (memory, ~0.2 ms)
     │        2a. hit: "4299" ◄──────
     │
     │ 2b. miss: ask the database (~200 ms)
     ▼
  PostgreSQL ──► 3. store the copy in Redis with a TTL of 60 s
```

A cache makes reads faster and takes load off the database. But it adds a new problem: a copy can be **stale**, older than the truth. Every cache design answers two questions:

1. **How old may a copy be?** You choose a **TTL** (Time To Live): after 60 seconds the copy disappears by itself. A short TTL means fresher data and more database reads.
2. **What happens when the data changes?** You **invalidate** the copy: delete it, so the next read gets the new value.

> 💡 **Tip:** Cache only data that is read much more often than it changes, and where a few seconds of staleness is acceptable: prices on a product page, a user's profile, the result of a slow report. Do not cache data that must always be exact at that moment, such as a bank balance before a payment.

## 2. Redis in Docker

**Redis** (REmote DIctionary Server) is an **in-memory** key-value server: all data lives in RAM (Random Access Memory), so a command takes a fraction of a millisecond. Start it from the official image:

```bash
docker run -d --name p16-redis -p 127.0.0.1:6379:6379 redis:8
```

**What happens:**

1. `-d` runs the container in the background, `--name p16-redis` gives it a name for later commands.
2. `-p 127.0.0.1:6379:6379` makes port 6379 (the standard Redis port) reachable **only from your Mac**.
3. `redis:8` is the official Redis image, version 8.

> 🔒 **Security:** This Redis has no password. Never publish it on all network interfaces (`-p 6379:6379` without `127.0.0.1`) on a server: attackers scan the internet for open Redis ports and can read or delete everything. In production Redis gets a password (or ACL users, Access Control Lists) and sits in a private network.

The image contains the command-line client `redis-cli`. Open it inside the container:

```bash
docker exec -it p16-redis redis-cli
```

Type the commands after the prompt `127.0.0.1:6379>`:

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

**What happens:**

1. `SET key value` stores a value, `GET key` reads it. `(nil)` means "no such key". Values are strings (bytes).
2. `SET ... EX 30` stores a value that expires after 30 seconds. `TTL` shows the seconds left. `-1` means "this key never expires", `-2` means "no such key".
3. `INCR visits` adds 1 to a number and returns the new value. A missing key counts as 0. Redis runs one command at a time, so two clients that call `INCR` at the same moment never lose a count: the command is **atomic**.
4. `DEL` deletes keys and returns how many it deleted.

Type `exit` to leave `redis-cli`. Redis has more data types than strings; the ones you will meet most:

| Type | Example commands | Typical use |
|---|---|---|
| String | `SET`, `GET`, `INCR` | Cached values, counters |
| Hash | `HSET user:1 name Ann`, `HGET` | A small record with fields |
| List | `LPUSH`, `RPOP` | A simple queue |
| Set | `SADD`, `SISMEMBER` | "Has this user voted?" |
| Sorted set | `ZADD`, `ZRANGE` | Leaderboards, things ordered by time |

> 📏 **Convention:** Name keys `kind:version:id`, for example `price:v1:42`. The prefix shows what the key holds, and the version lets you change the format later: new code reads `price:v2:...` and the old `v1` keys simply expire.

## 3. Cache-aside in Rust

Create the project and add the crates:

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

The feature `tokio-comp` lets `redis` run on Tokio, and `connection-manager` gives us `ConnectionManager`: an async connection that **reconnects by itself** after a network problem and is cheap to clone, so many tasks can share it.

Put the cache logic in `src/lib.rs`, so tests can use it:

```rust
use std::time::Duration;

use redis::AsyncTypedCommands;
use redis::aio::ConnectionManager;

/// How long a cached price stays valid.
pub const PRICE_TTL_SECONDS: u64 = 60;

/// The cache key for one product. `v1` lets us change the format later.
pub fn price_key(product_id: u32) -> String {
    format!("price:v1:{product_id}")
}

/// Pretend database: slow on purpose.
pub async fn load_price_from_db(product_id: u32) -> u32 {
    tokio::time::sleep(Duration::from_millis(200)).await;
    product_id * 100 + 99
}

/// Cache-aside: try the cache, on a miss ask the database and fill the cache.
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

/// Call this after the price changed in the database.
pub async fn invalidate_price(
    cache: &mut ConnectionManager,
    product_id: u32,
) -> redis::RedisResult<()> {
    cache.del(price_key(product_id)).await?;
    Ok(())
}
```

**What happens:**

1. `AsyncTypedCommands` is a trait that adds one async method per Redis command to the connection, with a fixed result type: `get` returns `Option<String>`, `set_ex` returns `()`, `del` returns the number of deleted keys.
2. `load_price_from_db` stands in for a real query (Lesson 16.6). It sleeps 200 ms so that we can see the difference.
3. `get_price` is **cache-aside**. Step one: `GET`. If a value is there and parses as a number, it is a hit and we return at once.
4. On a miss we read the "database" and store the value with `set_ex` (Redis `SET key value EX 60`). The next request for 60 seconds is a hit.
5. **A cache error is not a request error.** If Redis is down, `get` returns `Err`; we print a warning and use the database. The service becomes slower, but it still works. The cache is an optimization, not the source of truth.
6. If the cached text does not parse (for example, an old format), we also fall through to the database and overwrite it.
7. `invalidate_price` deletes the copy. Call it right after the database write succeeds.

Now `src/main.rs`:

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

**What happens:**

1. The Redis address comes from the environment variable `REDIS_URL`, with a default for your laptop. This is the twelve-factor rule "configuration in the environment" from [Lesson 08.15](../08-software-design/15-the-twelve-factor-app.md); Lesson 16.13 builds on it.
2. `Client::open` only checks the URL (Uniform Resource Locator, the address). `get_connection_manager().await` really connects.
3. The first read is a miss and takes about 200 ms. The next two are hits and take less than a millisecond (your times will differ a little).
4. After `invalidate_price`, the key is gone, so the next read goes to the database again and refills the cache.

Look at the cached key from `redis-cli` (run `cargo run` again first, so that it exists):

```text
127.0.0.1:6379> GET price:v1:42
"4299"
127.0.0.1:6379> TTL price:v1:42
(integer) 60
```

> ⚡ **Performance:** A Redis round trip on the same machine costs roughly 0.1–0.5 ms; across a data-center network it is still about 1 ms. Caching pays off when the source is much slower than that. Caching something that takes 0.3 ms to compute only adds work.

## 4. What can go wrong with caches

Caches are simple to add and easy to get wrong. The classic problems:

| Problem | What happens | Usual fix |
|---|---|---|
| **Stale data** | The database changed, the cache still has the old copy. | Invalidate on every write; keep TTLs short enough. |
| **Write order** | You delete the cache key, then another request reads the *old* database value and caches it again, then your database write happens. | Write the database **first**, then delete the key. The TTL limits any damage that remains. |
| **Stampede** | A popular key expires; 500 requests miss at the same moment and all hit the database. | Add a little random time to each TTL so keys do not expire together; let only one request refill a key. |
| **Unbounded memory** | Every key is kept forever and Redis runs out of RAM. | Always set a TTL on cache keys; configure `maxmemory` with an eviction policy in production. |
| **Caching errors** | A failed lookup ("not found" because the database was down) is cached for 60 s. | Cache only successful results, or "not found" with a much shorter TTL. |

> 🔒 **Security:** Never put user-specific data under a key that does not contain the user id. If two users share `profile:v1:me`, one of them will see the other's data. Keys built from user input must also be checked (Lesson 10.4), so that nobody can read `price:v1:*` keys of other kinds.

## 5. A counter with a time window: rate limiting

`INCR` and `EXPIRE` together make a simple **rate limiter**: "at most 3 requests per 10 seconds per client". The first request in a window creates the counter and gives it a TTL; when the TTL ends, the counter disappears and a new window begins. Add to `src/lib.rs`:

```rust
/// Fixed-window rate limit: at most `limit` calls per `window_seconds` for one client.
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

And a second program, `src/bin/ratelimit.rs`:

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

**What happens:**

1. `incr` returns the new count. Because `INCR` is atomic, two copies of your service running at the same time still count correctly. This is why the counter lives in Redis and not in the memory of one process.
2. Only the first request of a window sets the TTL with `expire`.
3. Requests 4 and 5 are over the limit. In a web service you would answer them with HTTP (Hypertext Transfer Protocol) status `429 Too Many Requests`. Run the program again after 10 seconds and the first three are allowed again.

There is one weakness: if the service crashes between `incr` and `expire`, the key has no TTL and the client stays blocked forever. Exercise 3 fixes it.

> 🧪 **Testing:** Code that talks to Redis needs a real Redis to be tested properly. Keep such tests separate, as shown in "Try it yourself"; Lesson 16.14 shows how a test can start its own Redis container.

## Common mistakes

> ⚠️ **Common mistake:** Redis is not running. `get_connection_manager()` fails at start-up:

```text
Error: Connection refused (os error 61)
```

Fix: `docker start p16-redis` (or the `docker run` command from Section 2). Failing at start-up is fine; failing *requests* later because the cache is down is not (Section 3, point 5).

> ⚠️ **Common mistake:** Using `set` instead of `set_ex` for cache entries. The key never expires (`TTL` shows `-1`), old data stays forever, and memory grows. Every cache key needs a TTL.

> ⚠️ **Common mistake:** Deleting the cache key *before* writing the database. A request that comes in between caches the old value again. Write first, invalidate second.

## Try it yourself

1. Start Redis and run `cargo run` twice. On the second run, all three reads are hits. Why? (The key from the first run is still alive.)
2. In `redis-cli`, run `TTL price:v1:42` a few times and watch it count down.
3. Put tests in `tests/cache.rs`: a normal test for the key format, and a test that needs Redis, marked `#[ignore]` with a reason:

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
    let id = 900_001; // an id that no other test uses
    p16_redis_cache::invalidate_price(&mut cache, id).await.unwrap();

    let first = get_price(&mut cache, id).await;
    let second = get_price(&mut cache, id).await;

    assert_eq!(first.1, "database");
    assert_eq!(second, (first.0, "cache"));
}
```

`cargo test --test cache` skips the Redis test:

```text
running 2 tests
test second_read_comes_from_cache ... ignored, needs a running Redis
test key_has_prefix_version_and_id ... ok

test result: ok. 1 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

With Redis running, `cargo test --test cache -- --ignored` runs it:

```text
running 1 test
test second_read_comes_from_cache ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 1 filtered out; finished in 0.21s
```

The test uses its own product id and first deletes the key, so it gives the same result on every run and does not disturb other tests.

## Exercises

### Exercise 1 — A test for invalidation (easy)

Write an ignored test `invalidation_forces_a_database_read`: fill the cache for one id, invalidate it, read again, and check that the source is `"database"`.

<details>
<summary>Hint</summary>

Copy the connection lines from `second_read_comes_from_cache`. Use a different id, such as `900_002`, so the two tests never touch the same key.

</details>

<details>
<summary>Answer</summary>

```rust
#[tokio::test]
#[ignore = "needs a running Redis"]
async fn invalidation_forces_a_database_read() {
    let url = std::env::var("REDIS_URL").unwrap_or("redis://127.0.0.1:6379/".to_string());
    let client = redis::Client::open(url).unwrap();
    let mut cache = client.get_connection_manager().await.unwrap();
    let id = 900_002;

    get_price(&mut cache, id).await; // fills the cache
    p16_redis_cache::invalidate_price(&mut cache, id)
        .await
        .unwrap();
    let (_, source) = get_price(&mut cache, id).await;

    assert_eq!(source, "database");
}
```

Why it works: after the first call the key exists; `invalidate_price` runs `DEL`, so the third step is a miss. If you forget the `DEL` in `invalidate_price`, this test fails, which is exactly what it guards against.

</details>

### Exercise 2 — Cache a whole struct as JSON (medium)

A product page needs more than a price. Add `serde` (with `derive`) and `serde_json`, define `Product { id: u32, name: String, price: u32 }`, and write `get_product(cache, id) -> Product` with cache-aside. Store the value as a JSON string under `product:v1:<id>`.

<details>
<summary>Hint</summary>

`serde_json::to_string(&product)` gives a `String` you can pass to `set_ex`. On a hit, `serde_json::from_str(&json)` turns it back. If decoding fails, treat it as a miss.

</details>

<details>
<summary>Answer</summary>

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

After one call, `GET product:v1:5` in `redis-cli` shows `"{\"id\":5,\"name\":\"product 5\",\"price\":599}"`.

Why it works: Redis stores bytes and does not care what they mean; JSON is just a convenient text format for a struct. The `let` chains (`if let ... && let ...`) keep the "hit" path short, and anything unexpected (a Redis error, old or broken JSON) falls through to the database.

</details>

### Exercise 3 — A rate limiter that cannot get stuck (hard)

Fix the weakness of `allow_request`: the counter and its TTL must be set in one atomic step, and the TTL must only be set when the key has none yet. Use a Redis pipeline with `atomic()` (it wraps the commands in `MULTI` … `EXEC`) and the command `EXPIRE key seconds NX` (`NX`: only if the key has no TTL yet).

<details>
<summary>Hint</summary>

`redis::pipe().atomic().incr(&key, 1)` starts the pipeline. Add the `EXPIRE` with `.cmd("EXPIRE").arg(&key).arg(window_seconds).arg("NX")` and `.ignore()` its result. `query_async` then returns a tuple with one value: the count.

</details>

<details>
<summary>Answer</summary>

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

Five calls with limit 3 give `true, true, true, false, false`, and `TTL rate:v1:<client>` is between 1 and 10.

Why it works: `MULTI`/`EXEC` makes Redis run both commands together, with no other command in between, and a crash of your service can no longer separate them. `NX` keeps the first TTL of the window, so later requests do not push the end of the window further away.

</details>

## Quiz

1. What is the difference between a cache hit and a cache miss, and what does cache-aside do on a miss?

<details>
<summary>Answer</summary>

A hit means the value was in the cache; a miss means it was not. On a miss, cache-aside reads the real source (the database), stores the result in the cache with a TTL, and returns it.

</details>

2. Redis is down. What should `get_price` do, and why?

<details>
<summary>Answer</summary>

Log a warning and read from the database. The cache is only an optimization; the service must keep working (slower) without it.

</details>

3. Why should every cache key have a TTL?

<details>
<summary>Answer</summary>

So that stale copies disappear by themselves even if an invalidation was missed, and so that memory does not grow forever.

</details>

4. A price is updated. In which order do you write the database and delete the cache key?

<details>
<summary>Answer</summary>

Database first, then delete the key. If you delete first, a request in between can read the old value from the database and put it back into the cache.

</details>

5. Why does a rate-limit counter belong in Redis and not in a `HashMap` inside the service?

<details>
<summary>Answer</summary>

A service usually runs as several copies (processes or containers). A `HashMap` would count separately in each copy. Redis is shared by all copies, and `INCR` is atomic, so the count is correct.

</details>

## Summary

- A cache keeps fast copies of slow data. It speeds up reads and protects the database, at the price of possibly stale data.
- Redis is an in-memory key-value server. `SET key value EX seconds`, `GET`, `DEL`, `TTL` and the atomic `INCR` cover most cache work.
- Cache-aside: read the cache; on a miss read the database and fill the cache with a TTL. Invalidate after every write, database first.
- A Redis error must not fail the request: log it and fall back to the source of truth.
- `ConnectionManager` is a shared, self-reconnecting async connection; the address comes from `REDIS_URL`.
- `INCR` plus a TTL makes a fixed-window rate limiter; an atomic pipeline with `EXPIRE ... NX` makes it safe.

## Sources and further reading

- [Redis — Commands](https://redis.io/docs/latest/commands/) — every command, including `SET` options, `INCR`, `EXPIRE` and `TTL`.
- [Redis — Understand Redis data types](https://redis.io/docs/latest/develop/data-types/) — strings, hashes, lists, sets and sorted sets with examples.
- [Redis — Transactions](https://redis.io/docs/latest/develop/using-commands/transactions/) — how `MULTI`/`EXEC` works.
- [redis crate on docs.rs](https://docs.rs/redis) — the Rust client: `Client`, `ConnectionManager`, `AsyncTypedCommands`, pipelines.
- [Docker Hub — redis official image](https://hub.docker.com/_/redis) — the image used here and how to configure it.

---

[🏠 Start](../START-HERE.md) · [← 16.9 gRPC with tonic](09-grpc-with-tonic.md) · [📋 Phase 16](README.md) · [16.11 Messaging with NATS →](11-messaging-with-nats.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
