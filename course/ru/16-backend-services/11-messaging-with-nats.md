<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 16.10 Кэширование с Redis](10-caching-with-redis.md) · [📋 Этап 16](README.md) · [16.12 Логи, метрики и трассировки с OpenTelemetry →](12-logs-metrics-and-traces.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/16-backend-services/11-messaging-with-nats.md) <!-- nav -->

# Урок 16.11 — Обмен сообщениями через NATS

> **Этап 16 — Бэкенд-сервисы** · Урок 11 из 16 · ⏱ около 120 минут

## Что вы узнаете

- Почему сервисы отправляют друг другу сообщения, а не всегда вызывают друг друга напрямую.
- Основные идеи NATS: темы (subjects), шаблоны, публикация/подписка, запрос/ответ и группы очередей.
- Как работать с NATS из Rust с помощью крейта `async-nats`.
- Что значат доставка «не более одного раза» и «хотя бы один раз» и как JetStream хранит сообщения, чтобы ничего не потерялось.
- Как тестировать код, который общается с сервером сообщений.

## Перед началом

- Вы прошли [Урок 16.10 — Кэширование с Redis](10-caching-with-redis.md) и умеете запускать сервер в Docker.
- Вы знаете задачи и потоки Tokio ([Урок 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)); `StreamExt::next` используется здесь так же, как в Уроке 16.9.

## Новые слова

| Слово | Значение |
|---|---|
| **сообщение** (message) | Небольшой кусок данных (байты) с именем (темой), который одна программа отправляет, а другие получают. |
| **брокер сообщений** (message broker) | Сервер, который принимает сообщения и передаёт их программам, которым они нужны. NATS — один из них. |
| **NATS** | Быстрый, простой брокер сообщений с открытым исходным кодом: одна небольшая серверная программа. |
| **тема** (subject) | Имя сообщения из частей (токенов), разделённых точками: `orders.created`. Получатели выбирают сообщения по теме. |
| **шаблон** (wildcard) | Специальный токен в подписке: `*` совпадает ровно с одним токеном, `>` — с одним или несколькими токенами в конце. |
| **публикация / подписка** (publish / subscribe, pub/sub) | Одна программа публикует сообщение в тему; каждая программа, подписанная на эту тему, получает копию. |
| **запрос / ответ** (request / reply) | Сообщение, которое ждёт одного ответа, отправленного в личную тему для ответа. |
| **группа очереди** (queue group) | Подписчики с одним именем группы делят работу: каждое сообщение достаётся только **одному** из них. |
| **JetStream** | Часть NATS, которая хранит сообщения на диске, чтобы их можно было доставить позже и повторно. |
| **поток** (stream, JetStream) | Сохранённый упорядоченный журнал всех сообщений, темы которых совпадают с шаблоном. |
| **потребитель** (consumer, JetStream) | Читатель потока, который помнит, какие сообщения уже обработал. |
| **подтверждение** (ack, acknowledgement) | Сообщение от потребителя серверу: «я обработал это, не присылай снова». |
| **не более одного раза / хотя бы один раз** (at-most-once / at-least-once) | Обещания доставки: сообщение приходит ноль или один раз / один или больше раз. |
| **идемпотентный** (idempotent) | Об операции, которая даёт тот же результат, если выполнить её дважды, например «поставить статус „оплачен“». |

## 1. Зачем сообщения?

До сих пор наши сервисы вызывали друг друга напрямую: вызов REST (Урок 16.3) или gRPC (Урок 16.9). Вызывающий ждёт, пока другой сервис ответит. Это правильно, когда ответ **нужен** вызывающему прямо сейчас. Но подумайте о магазине:

```text
Direct calls:                          With messages:

orders ──► billing                     orders ──publish "orders.created"──► NATS
   │                                                                       │ │ │
   ├─────► email                                             billing ◄─────┘ │ │
   │                                                         email   ◄───────┘ │
   └─────► analytics                                         analytics ◄───────┘
```

При прямых вызовах сервис заказов должен знать каждый другой сервис, ждать каждого из них и падать, когда один из них недоступен. С сообщениями он публикует **одно** событие «создан заказ» и идёт дальше. Каждый заинтересованный сервис подписывается. Завтра можно добавить новый сервис, не меняя сервис заказов. Это называется **слабым сопряжением** (Урок 08.2).

**Брокер сообщений** стоит посередине и передаёт сообщения дальше. Мы используем **NATS**: один маленький сервер, очень быстрый, его легко запустить.

## 2. NATS в Docker

```bash
docker run -d --name p16-nats -p 127.0.0.1:4222:4222 nats:2 -js
```

**Что происходит:**

1. `nats:2` — официальный образ сервера NATS версии 2. Порт 4222 — тот, к которому подключаются клиенты.
2. `-js` (после имени образа, поэтому параметр получает серверная программа) включает **JetStream**, он понадобится в разделе 6.
3. `127.0.0.1` оставляет порт доступным только с вашего Mac. В продакшене у NATS есть имена пользователей и пароли или ключи, а также TLS (Transport Layer Security, протокол защиты транспортного уровня).

Проверьте, что он запустился:

```bash
docker logs p16-nats 2>&1 | grep -E "Version|Server is ready"
```

```text
[1] 2026/11/18 07:15:20.382342 [INF]   Version:  2.15.3
[1] 2026/11/18 07:15:20.386422 [INF] Server is ready
```

Создайте проект. Каждая идея получает маленькую программу в `src/bin/`:

```bash
cd ~/projects/rust/code
cargo new p16-nats-messaging
cd p16-nats-messaging
rm src/main.rs
mkdir src/bin
cargo add async-nats@0.52
cargo add tokio@1 --features macros,rt-multi-thread,time
cargo add futures@0.3
```

Положите небольшой помощник в `src/lib.rs`, чтобы все программы подключались одинаково:

```rust
/// Подключается к серверу NATS из `NATS_URL` (по умолчанию — к локальному).
pub async fn connect() -> Result<async_nats::Client, async_nats::ConnectError> {
    let url = std::env::var("NATS_URL").unwrap_or("nats://127.0.0.1:4222".to_string());
    async_nats::connect(url).await
}
```

`Client` держит одно соединение TCP (Transmission Control Protocol, протокол управления передачей) с сервером. Его дёшево клонировать, и все клоны делят соединение.

## 3. Публикация и подписка

Темы — это слова с точками, от общего к частному: `orders.created`, `orders.paid`, `users.created`. Подписчик может использовать **шаблоны**:

| Подписка | Совпадает | Не совпадает |
|---|---|---|
| `orders.created` | `orders.created` | `orders.paid` |
| `orders.*` | `orders.created`, `orders.paid` | `orders.eu.created`, `users.created` |
| `orders.>` | `orders.created`, `orders.eu.created` | `orders`, `users.created` |

`*` — ровно один токен; `>` — один или несколько токенов, и он может стоять только последним. Создайте `src/bin/pubsub.rs`:

```rust
use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = p16_nats_messaging::connect().await?;

    // Сначала подписка: core NATS доставляет только тем подписчикам, которые есть прямо сейчас.
    let mut orders = client.subscribe("orders.*").await?;

    client.publish("orders.created", "order 1".into()).await?;
    client.publish("users.created", "user 7".into()).await?;
    client.publish("orders.paid", "order 1".into()).await?;
    client.flush().await?;

    for _ in 0..2 {
        if let Some(message) = orders.next().await {
            let text = String::from_utf8_lossy(&message.payload);
            println!("got {} -> {text}", message.subject);
        }
    }
    Ok(())
}
```

```bash
cargo run --bin pubsub
```

```text
got orders.created -> order 1
got orders.paid -> order 1
```

**Что происходит:**

1. `subscribe("orders.*")` говорит серверу: «присылай мне каждое сообщение, тема которого совпадает». Он возвращает `Subscriber` — поток сообщений.
2. `publish(subject, payload)` отправляет сообщение. Полезная нагрузка имеет тип `Bytes` (буфер байтов); `"order 1".into()` преобразует текст.
3. `publish` только кладёт сообщение в буфер отправки клиента. `flush()` ждёт, пока всё действительно уйдёт на сервер.
4. `users.created` не совпадает с `orders.*`, поэтому наш подписчик его не видит. Больше никто на него не подписан, и сервер просто отбрасывает его.
5. `orders.next().await` ждёт следующего сообщения. `message.subject` говорит, какая тема совпала с шаблоном.

В настоящих системах издатель и подписчики — разные программы, часто на разных машинах. Здесь они в одной программе, чтобы за выводом было легко следить.

## 4. Запрос и ответ

Иногда ответ всё-таки нужен, но хочется сохранить гибкость сообщений: вызывающему не нужно знать адрес сервиса, только тему. Это даёт **запрос/ответ**. Создайте `src/bin/request.rs`:

```rust
use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = p16_nats_messaging::connect().await?;

    // Отвечающий: задача, которая отвечает на каждый запрос в "greet".
    let mut requests = client.subscribe("greet").await?;
    let responder = client.clone();
    tokio::spawn(async move {
        while let Some(request) = requests.next().await {
            let name = String::from_utf8_lossy(&request.payload).to_string();
            if let Some(reply_to) = request.reply {
                let answer = format!("hello, {name}");
                responder.publish(reply_to, answer.into()).await.unwrap();
            }
        }
    });
    client.flush().await?;

    // Запрашивающий.
    let reply = client.request("greet", "Ann".into()).await?;
    println!("reply: {}", String::from_utf8_lossy(&reply.payload));

    match client.request("nobody.listens", "hi".into()).await {
        Ok(_) => println!("unexpected reply"),
        Err(err) => println!("error: {:?}", err.kind()),
    }
    Ok(())
}
```

```bash
cargo run --bin request
```

```text
reply: hello, Ann
error: NoResponders
```

**Что происходит:**

1. `request("greet", ...)` создаёт личную случайную **тему для ответа** («почтовый ящик», inbox), подписывается на неё и публикует запрос с этой темой.
2. Отвечающий видит `request.reply == Some(inbox)` и публикует ответ в этот ящик. Слушает там только запрашивающий.
3. `request` возвращает первый ответ. Если ничего не пришло, он завершается ошибкой по тайм-ауту (по умолчанию 10 секунд).
4. На `nobody.listens` нет ни одного подписчика. Сервер это знает и сразу отвечает «no responders» («нет отвечающих»), так что вызывающий не ждёт тайм-аута.

> 💡 **Совет:** несколько копий отвечающего могут подписаться на одну тему в группе очереди (следующий раздел). Вызывающий не меняется, а NATS распределяет запросы по копиям: простой балансировщик нагрузки.

## 5. Группы очередей: делим работу

При обычных подписках **каждый** подписчик получает **каждое** сообщение. Это правильно для событий («и биллингу, и почте нужно знать»). Для работы («уменьши эту картинку») нужно, чтобы каждое задание выполнилось **один** раз тем рабочим, кто свободен. Это **группа очереди**: подписчики с одним именем группы делят сообщения. Создайте `src/bin/queue.rs`:

```rust
use std::time::Duration;

use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = p16_nats_messaging::connect().await?;

    for worker in ["worker-a", "worker-b"] {
        let mut jobs = client.queue_subscribe("jobs", "resizers".to_string()).await?;
        tokio::spawn(async move {
            while let Some(job) = jobs.next().await {
                println!("{worker} took {}", String::from_utf8_lossy(&job.payload));
            }
        });
    }
    client.flush().await?;

    for i in 1..=6 {
        client.publish("jobs", format!("job {i}").into()).await?;
    }
    client.flush().await?;
    tokio::time::sleep(Duration::from_millis(200)).await;
    Ok(())
}
```

```bash
cargo run --bin queue
```

Один запуск напечатал:

```text
worker-b took job 3
worker-b took job 5
worker-b took job 6
worker-a took job 1
worker-a took job 2
worker-a took job 4
```

**Что происходит:**

1. Обе подписки используют группу `"resizers"`. Для каждого сообщения сервер выбирает случайно **одного** члена группы.
2. Каждое задание взято ровно один раз. Какой рабочий какое задание возьмёт, меняется при каждом запуске; другой запуск отдал все шесть `worker-a`. Не полагайтесь на порядок.
3. `sleep` в конце даёт задачам-рабочим время напечатать вывод, прежде чем `main` вернётся и программа завершится.

> ⚡ **Производительность:** чтобы обрабатывать больше работы, запустите больше рабочих в той же группе, при необходимости на других машинах. Больше ничего не меняется: так масштабируются системы на сообщениях.

## 6. Обещания доставки и JetStream

Core NATS, то есть всё, что выше, даёт доставку **не более одного раза**. Сервер передаёт каждое сообщение подписчикам, подключённым *прямо сейчас*, и ничего не хранит. Если сервис биллинга в этот момент перезапускается, он никогда не увидит заказ. Для многих сообщений это нормально (живые обновления, метрики). Для заказа — нет.

**JetStream** добавляет хранение:

```text
publisher ──"orders.created"──► stream ORDERS  [1][2][3][4]...   (on disk)
                                     │
                                     └──► consumer "billing": acknowledged up to 2,
                                                              next to deliver: 3
```

- **Поток** хранит каждое сообщение, тема которого совпадает с его шаблоном, по порядку, с порядковым номером.
- **Потребитель** читает поток и помнит свою позицию. **Долговечный** (durable) потребитель имеет имя и сохраняет позицию, даже когда использующая его программа останавливается.
- Программа **подтверждает** (ack) каждое сообщение после обработки. Сообщение без подтверждения вовремя доставляется снова: доставка **хотя бы один раз**.

Создайте `src/bin/jetstream.rs`:

```rust
use async_nats::jetstream::{self, consumer, stream};
use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = p16_nats_messaging::connect().await?;
    let js = jetstream::new(client);

    // Поток хранит каждое сообщение, тема которого совпадает с "orders.>".
    let orders = js
        .get_or_create_stream(stream::Config {
            name: "ORDERS".to_string(),
            subjects: vec!["orders.>".to_string()],
            ..Default::default()
        })
        .await?;

    for text in ["order A", "order B"] {
        let ack = js.publish("orders.created", text.into()).await?.await?;
        println!("stored {text} as message {}", ack.sequence);
    }

    // Долговечный потребитель помнит, какие сообщения он уже подтвердил.
    let billing = orders
        .get_or_create_consumer(
            "billing",
            consumer::pull::Config {
                durable_name: Some("billing".to_string()),
                ..Default::default()
            },
        )
        .await?;

    let mut batch = billing.fetch().max_messages(10).messages().await?;
    while let Some(message) = batch.next().await {
        let message = message?;
        let sequence = message.info()?.stream_sequence;
        let text = String::from_utf8_lossy(&message.payload);
        println!("billing got message {sequence}: {text}");
        message.ack().await?;
    }
    Ok(())
}
```

Запустите дважды:

```bash
cargo run --bin jetstream
cargo run --bin jetstream
```

```text
stored order A as message 1
stored order B as message 2
billing got message 1: order A
billing got message 2: order B
stored order A as message 3
stored order B as message 4
billing got message 3: order A
billing got message 4: order B
```

**Что происходит:**

1. `jetstream::new(client)` даёт *контекст* JetStream: точку входа для потоков и потребителей.
2. `get_or_create_stream` создаёт поток `ORDERS` в первый раз, а потом просто возвращает его. Имена потоков обычно пишут заглавными буквами.
3. У `js.publish(...)` **два** `.await`. Первый отправляет сообщение; второй ждёт подтверждения сервера, что сообщение сохранено. `ack.sequence` — его номер в потоке.
4. В первом запуске мы опубликовали **до** того, как появился потребитель, и он всё равно получил оба сообщения. Они ждали в потоке. В этом большое отличие от core NATS.
5. `fetch().max_messages(10)` просит до 10 ожидающих сообщений и заканчивается, когда их больше нет. Это **pull**-потребитель: программа сама спрашивает, когда готова.
6. После обработки `message.ack()` говорит серверу «готово». Во втором запуске потребитель `billing` получил только **новые** сообщения 3 и 4, потому что 1 и 2 были подтверждены. Он помнил свою позицию между двумя отдельными запусками программы.

> ⚠️ **Частая ошибка:** думать, что «хотя бы один раз» значит «ровно один раз». Если программа обработала сообщение и упала до подтверждения, сообщение придёт снова. Делайте обработчики **идемпотентными**: «пометить заказ 17 оплаченным» безопасно выполнить дважды, а «добавить 10 евро к балансу» — нет. При необходимости храните id обработанных сообщений.

## Частые ошибки

> ⚠️ **Частая ошибка:** подписываться после публикации в core NATS. Если в `pubsub.rs` перенести строку `subscribe` ниже `flush`, программа будет ждать вечно: два сообщения не были доставлены никому и пропали. Подписывайтесь заранее или используйте JetStream, когда сообщения нельзя терять.

> ⚠️ **Частая ошибка:** сервер не запущен. `connect` завершается ошибкой:

```text
Error: Error { kind: Io, source: Some(Os { code: 61, kind: ConnectionRefused, message: "Connection refused" }) }
```

Исправление: `docker start p16-nats`. После подключения `async-nats` сам переподключается, если сервер перезапустился.

> ⚠️ **Частая ошибка:** забыть `flush()` перед завершением программы. `publish` возвращается, как только сообщение попало в буфер клиента. Если `main` сразу вернётся, сообщение может так и не покинуть процесс.

> 🔒 **Безопасность:** темы часто строятся из данных (`orders.{country}`). Проверяйте такие части (Урок 10.4): значение с точкой или `*` создаст другую тему, чем вы планировали. И любой, кто может подключиться к открытому серверу NATS, может подписаться на `>` и читать **все** сообщения. В продакшене используйте учётные записи (accounts) и учётные данные.

## Попробуйте сами

1. Снова запустите `pubsub`, но подпишитесь на `orders.>` и опубликуйте ещё и в `orders.eu.created` с нагрузкой `order 2`. Читайте три сообщения вместо двух. Вы увидите:

```text
got orders.created -> order 1
got orders.paid -> order 1
got orders.eu.created -> order 2
```

2. Запустите `queue` три раза и сравните, какой рабочий какое задание берёт.
3. Остановите NATS через `docker stop p16-nats` и снова запустите через `docker start p16-nats`. Ещё раз выполните `jetstream`: номера продолжаются (5 и 6), потому что контейнер сохранил данные.

## Упражнения

### Упражнение 1 — Какие подписки совпадают? (легко)

Для каждой темы скажите, какие из `orders.*`, `orders.>` и `*.created` её получат: `orders.created`, `orders.eu.created`, `users.created`, `orders`.

<details>
<summary>Подсказка</summary>

Посчитайте токены (части между точками). `*` означает ровно один токен, `>` — один или несколько в конце.

</details>

<details>
<summary>Ответ</summary>

| Тема | `orders.*` | `orders.>` | `*.created` |
|---|---|---|---|
| `orders.created` | да | да | да |
| `orders.eu.created` | нет (3 токена) | да | нет (3 токена) |
| `users.created` | нет | нет | да |
| `orders` | нет | нет (`>` нужен хотя бы один токен) | нет |

Почему это работает: сравнение идёт токен за токеном, и только `>` может покрыть несколько токенов.

</details>

### Упражнение 2 — Тест сервиса запрос/ответ (средне)

Напишите интеграционный тест в `tests/messaging.rs` с пометкой `#[ignore = "needs a running NATS server"]`. Он запускает отвечающего на `math.double`, который отвечает удвоенным числом или `error: not a number`. Проверьте `"21"` → `"42"` и `"abc"` → `"error: not a number"`.

<details>
<summary>Подсказка</summary>

Скопируйте отвечающего из `request.rs`. Разберите нагрузку через `text.parse::<i64>()`. Значение `Bytes` можно сравнивать с `&str` прямо в `assert_eq!`.

</details>

<details>
<summary>Ответ</summary>

```rust
use futures::StreamExt;

#[tokio::test]
#[ignore = "needs a running NATS server"]
async fn double_service_answers_requests() {
    let client = p16_nats_messaging::connect().await.unwrap();
    let mut requests = client.subscribe("math.double").await.unwrap();
    let responder = client.clone();
    tokio::spawn(async move {
        while let Some(request) = requests.next().await {
            let text = String::from_utf8_lossy(&request.payload).to_string();
            let answer = match text.parse::<i64>() {
                Ok(n) => (n * 2).to_string(),
                Err(_) => "error: not a number".to_string(),
            };
            if let Some(reply_to) = request.reply {
                responder.publish(reply_to, answer.into()).await.unwrap();
            }
        }
    });
    client.flush().await.unwrap();

    let reply = client.request("math.double", "21".into()).await.unwrap();
    assert_eq!(reply.payload, "42");

    let reply = client.request("math.double", "abc".into()).await.unwrap();
    assert_eq!(reply.payload, "error: not a number");
}
```

Когда NATS запущен, `cargo test --test messaging -- --ignored` печатает `test double_service_answers_requests ... ok`.

Почему это работает: тест — одновременно и сервис, и его клиент. `flush()` гарантирует, что сервер зарегистрировал подписку. Здесь запрос идёт по тому же соединению, поэтому порядок и так сохраняется; но когда отвечающий и запрашивающий — разные клиенты, слишком ранний запрос может получить «no responders».

</details>

### Упражнение 3 — Посмотреть на повторную доставку (сложно)

Напишите `examples/redeliver.rs`: поток `EMAILS` на `emails.>`, одно сообщение и долговечный pull-потребитель `sender` с `ack_wait: Duration::from_secs(1)`. Получите сообщение, **не** подтверждайте его, подождите 2 секунды, получите снова и подтвердите. Печатайте, сколько раз сообщение было доставлено (`message.info()?.delivered`).

<details>
<summary>Подсказка</summary>

Используйте построитель `fetch()` дважды с `.max_messages(1)` и `.expires(Duration::from_secs(3))`, чтобы второй запрос ждал достаточно долго для повторной доставки.

</details>

<details>
<summary>Ответ</summary>

```rust
use std::time::Duration;

use async_nats::jetstream::{self, consumer, stream};
use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = p16_nats_messaging::connect().await?;
    let js = jetstream::new(client);
    let emails = js
        .get_or_create_stream(stream::Config {
            name: "EMAILS".to_string(),
            subjects: vec!["emails.>".to_string()],
            ..Default::default()
        })
        .await?;
    js.publish("emails.welcome", "to ann@example.com".into())
        .await?
        .await?;

    let sender = emails
        .get_or_create_consumer(
            "sender",
            consumer::pull::Config {
                durable_name: Some("sender".to_string()),
                ack_wait: Duration::from_secs(1),
                ..Default::default()
            },
        )
        .await?;

    for attempt in 1..=2 {
        let mut batch = sender
            .fetch()
            .max_messages(1)
            .expires(Duration::from_secs(3))
            .messages()
            .await?;
        if let Some(message) = batch.next().await {
            let message = message?;
            let info = message.info()?;
            println!(
                "attempt {attempt}: message {} delivered {} time(s)",
                info.stream_sequence, info.delivered
            );
            if attempt == 1 {
                println!("pretend the email server is down: no ack");
                tokio::time::sleep(Duration::from_secs(2)).await;
            } else {
                message.ack().await?;
                println!("sent and acknowledged");
            }
        }
    }
    Ok(())
}
```

```bash
cargo run --example redeliver
```

```text
attempt 1: message 1 delivered 1 time(s)
pretend the email server is down: no ack
attempt 2: message 1 delivered 2 time(s)
sent and acknowledged
```

Почему это работает: после доставки сервер ждёт подтверждения `ack_wait` (1 секунду). Когда его нет, сервер считает, что рабочий сломался, и доставляет то же сообщение снова, увеличив `delivered`. Это доставка «хотя бы один раз» в действии.

</details>

## Вопросы для самопроверки

1. Чем обычная подписка отличается от группы очереди?

<details>
<summary>Ответ</summary>

При обычных подписках каждый подписчик получает копию каждого сообщения. В группе очереди каждое сообщение достаётся только одному члену группы, так что члены делят работу.

</details>

2. Биллинг недоступен одну минуту. Заказы публикуются через `publish` core NATS. Что происходит с этими заказами и как это меняет JetStream?

<details>
<summary>Ответ</summary>

Core NATS доставляет только подключённым подписчикам, поэтому биллинг их не получит (не более одного раза). С JetStream поток хранит сообщения, и долговечный потребитель биллинга получит их, когда вернётся.

</details>

3. Почему у `js.publish(...)` два `.await`?

<details>
<summary>Ответ</summary>

Первый отправляет сообщение; второй ждёт подтверждения сервера, что оно сохранено в потоке. Только после второго вы знаете, что сообщение в безопасности.

</details>

4. Что значит `NoResponders` для запроса?

<details>
<summary>Ответ</summary>

На эту тему никто не подписан. Сервер отвечает сразу, так что вызывающий не ждёт тайм-аута.

</details>

5. Почему обработчик сообщений JetStream должен быть идемпотентным?

<details>
<summary>Ответ</summary>

Доставка «хотя бы один раз»: если обработчик упал или слишком медлил до подтверждения, то же сообщение доставляется снова. Идемпотентный обработчик даёт тот же результат, если выполнится дважды.

</details>

## Итоги

- Сообщения ослабляют связь сервисов: издатель не знает получателей и не ждёт их, а новых получателей можно добавить позже.
- NATS направляет сообщения по темам; `*` совпадает с одним токеном, `>` — с остатком.
- Публикация/подписка отправляет копию каждому подписчику; запрос/ответ получает один ответ через личный ящик; группы очередей делят работу между рабочими.
- Core NATS доставляет не более одного раза: подписывайтесь до публикации и вызывайте `flush()` перед выходом.
- Потоки JetStream хранят сообщения; долговечные потребители помнят позицию; неподтверждённые сообщения доставляются снова (хотя бы один раз), поэтому обработчики должны быть идемпотентными.

## Источники и что почитать

- [NATS docs — Subject-based messaging](https://docs.nats.io/nats-concepts/subjects) — темы, токены и шаблоны.
- [NATS docs — Core NATS](https://docs.nats.io/nats-concepts/core-nats) — публикация/подписка, запрос/ответ и группы очередей.
- [NATS docs — JetStream](https://docs.nats.io/nats-concepts/jetstream) — потоки, потребители, подтверждения и гарантии доставки.
- [async-nats на docs.rs](https://docs.rs/async-nats) — клиент для Rust, используемый в этом уроке.
- [Docker Hub — официальный образ nats](https://hub.docker.com/_/nats) — образ сервера и его параметры.

---

[🏠 Начало](../START-HERE.md) · [← 16.10 Кэширование с Redis](10-caching-with-redis.md) · [📋 Этап 16](README.md) · [16.12 Логи, метрики и трассировки с OpenTelemetry →](12-logs-metrics-and-traces.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
