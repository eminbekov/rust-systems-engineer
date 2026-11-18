<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.10 Caching with Redis](10-caching-with-redis.md) · [📋 Phase 16](README.md) · [16.12 Logs, metrics and traces with OpenTelemetry →](12-logs-metrics-and-traces.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/11-messaging-with-nats.md) <!-- nav -->

# Lesson 16.11 — Messaging with NATS

> **Phase 16 — Backend services** · Lesson 11 of 16 · ⏱ about 120 minutes

## What you will learn

- Why services send each other messages instead of always calling each other directly.
- The core ideas of NATS: subjects, wildcards, publish/subscribe, request/reply and queue groups.
- How to use NATS from Rust with the `async-nats` crate.
- What "at most once" and "at least once" delivery mean, and how JetStream stores messages so that nothing is lost.
- How to test code that talks to a message server.

## Before you start

- You finished [Lesson 16.10 — Caching with Redis](10-caching-with-redis.md) and can run a server in Docker.
- You know Tokio tasks and streams ([Lesson 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)); `StreamExt::next` is used here as in Lesson 16.9.

## New words

| Word | Meaning |
|---|---|
| **message** | A small piece of data (bytes) with a name (a subject) that one program sends and others receive. |
| **message broker** | A server that receives messages and passes them to the programs that want them. NATS is one. |
| **NATS** | A fast, simple, open-source message broker, written as one small server program. |
| **subject** | The name of a message, made of tokens with dots: `orders.created`. Receivers choose messages by subject. |
| **wildcard** | A special token in a subscription: `*` matches exactly one token, `>` matches one or more tokens at the end. |
| **publish / subscribe** (pub/sub) | One program publishes a message to a subject; every program subscribed to that subject gets a copy. |
| **request / reply** | A message that expects one answer, sent back to a private reply subject. |
| **queue group** | Subscribers with the same group name share the work: each message goes to only **one** of them. |
| **JetStream** | The part of NATS that stores messages on disk so that they can be delivered later and again. |
| **stream** (JetStream) | A stored, ordered log of all messages whose subjects match a pattern. |
| **consumer** (JetStream) | A reader of a stream that remembers which messages it has already processed. |
| **ack** (acknowledgement) | A message from the consumer to the server: "I processed this one, do not send it again". |
| **at-most-once / at-least-once** | Delivery promises: a message arrives zero or one time / one or more times. |
| **idempotent** | An operation that gives the same result when you do it twice, for example "set status to paid". |

## 1. Why messages?

So far, our services called each other directly: a REST call (Lesson 16.3) or gRPC (Lesson 16.9). The caller waits until the other service answers. That is right when the caller **needs** the answer now. But think about a shop:

```text
Direct calls:                          With messages:

orders ──► billing                     orders ──publish "orders.created"──► NATS
   │                                                                       │ │ │
   ├─────► email                                             billing ◄─────┘ │ │
   │                                                         email   ◄───────┘ │
   └─────► analytics                                         analytics ◄───────┘
```

With direct calls, the orders service must know every other service, wait for each of them, and fail when one of them is down. With messages, it publishes **one** event, "an order was created", and moves on. Each interested service subscribes. You can add a new service tomorrow without changing the orders service. This is called **loose coupling** (Lesson 08.2).

A **message broker** sits in the middle and passes messages on. We use **NATS**: one small server, very fast, easy to run.

## 2. NATS in Docker

```bash
docker run -d --name p16-nats -p 127.0.0.1:4222:4222 nats:2 -js
```

**What happens:**

1. `nats:2` is the official NATS server image, version 2. Port 4222 is where clients connect.
2. `-js` (after the image name, so it goes to the server program) turns on **JetStream**, which we need in Section 6.
3. `127.0.0.1` keeps the port private to your Mac. In production, NATS gets user names and passwords or keys, and TLS (Transport Layer Security).

Check that it started:

```bash
docker logs p16-nats 2>&1 | grep -E "Version|Server is ready"
```

```text
[1] 2026/11/18 07:15:20.382342 [INF]   Version:  2.15.3
[1] 2026/11/18 07:15:20.386422 [INF] Server is ready
```

Create the project. Each idea gets a small program in `src/bin/`:

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

Put a small helper in `src/lib.rs`, so that every program connects the same way:

```rust
/// Connects to the NATS server from `NATS_URL` (default: the local one).
pub async fn connect() -> Result<async_nats::Client, async_nats::ConnectError> {
    let url = std::env::var("NATS_URL").unwrap_or("nats://127.0.0.1:4222".to_string());
    async_nats::connect(url).await
}
```

A `Client` holds one TCP (Transmission Control Protocol) connection to the server. It is cheap to clone, and all clones share the connection.

## 3. Publish and subscribe

Subjects are words with dots, from general to specific: `orders.created`, `orders.paid`, `users.created`. A subscriber can use **wildcards**:

| Subscription | Matches | Does not match |
|---|---|---|
| `orders.created` | `orders.created` | `orders.paid` |
| `orders.*` | `orders.created`, `orders.paid` | `orders.eu.created`, `users.created` |
| `orders.>` | `orders.created`, `orders.eu.created` | `orders`, `users.created` |

`*` is exactly one token; `>` is one or more tokens and may only be last. Create `src/bin/pubsub.rs`:

```rust
use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = p16_nats_messaging::connect().await?;

    // Subscribe first: core NATS delivers only to subscribers that exist right now.
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

**What happens:**

1. `subscribe("orders.*")` tells the server "send me every message whose subject matches". It returns a `Subscriber`, which is a stream of messages.
2. `publish(subject, payload)` sends a message. The payload is `Bytes` (a byte buffer); `"order 1".into()` converts the text.
3. `publish` only puts the message into the client's send buffer. `flush()` waits until everything was really sent to the server.
4. `users.created` does not match `orders.*`, so our subscriber never sees it. Nobody else subscribed to it, so the server simply drops it.
5. `orders.next().await` waits for the next message. `message.subject` tells us which subject matched the wildcard.

In real systems the publisher and the subscribers are different programs, often on different machines. Here they are in one program so that the output is easy to follow.

## 4. Request and reply

Sometimes you do want an answer, but you still want the flexibility of messages: the caller does not need to know the address of the service, only a subject. **Request/reply** gives you that. Create `src/bin/request.rs`:

```rust
use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = p16_nats_messaging::connect().await?;

    // The responder: a task that answers every request on "greet".
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

    // The requester.
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

**What happens:**

1. `request("greet", ...)` creates a private, random **reply subject** (an "inbox"), subscribes to it, and publishes the request with that reply subject attached.
2. The responder sees `request.reply == Some(inbox)` and publishes its answer to that inbox. Only the requester listens there.
3. `request` returns the first reply. If nothing comes, it fails with a timeout (10 seconds by default).
4. For `nobody.listens` there is no subscriber at all. The server knows that and answers at once with "no responders", so the caller does not wait for the timeout.

> 💡 **Tip:** Several copies of the responder can subscribe to the same subject in a queue group (next section). The caller does not change, and NATS spreads the requests over the copies: a simple load balancer.

## 5. Queue groups: sharing work

With normal subscriptions **every** subscriber gets **every** message. That is right for events ("billing and email both want to know"). For work ("resize this image") you want each job done **once**, by whichever worker is free. That is a **queue group**: subscribers that use the same group name share the messages. Create `src/bin/queue.rs`:

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

One run printed:

```text
worker-b took job 3
worker-b took job 5
worker-b took job 6
worker-a took job 1
worker-a took job 2
worker-a took job 4
```

**What happens:**

1. Both subscriptions use the group `"resizers"`. For each message, the server picks **one** member of the group at random.
2. Every job is taken exactly once. Which worker takes which job is different on every run; another run gave all six to `worker-a`. Do not depend on the order.
3. The `sleep` at the end gives the worker tasks time to print before `main` returns and the program ends.

> ⚡ **Performance:** To handle more work, start more workers in the same queue group, on more machines if needed. Nothing else changes: this is how message-based systems scale out.

## 6. Delivery promises and JetStream

Core NATS, everything above, gives **at-most-once** delivery. The server passes each message to the subscribers that are connected *right now*, and keeps nothing. If the billing service is restarting at that moment, it never sees the order. For many messages that is fine (live updates, metrics). For an order, it is not.

**JetStream** adds storage:

```text
publisher ──"orders.created"──► stream ORDERS  [1][2][3][4]...   (on disk)
                                     │
                                     └──► consumer "billing": acknowledged up to 2,
                                                              next to deliver: 3
```

- A **stream** stores every message whose subject matches its pattern, in order, with a sequence number.
- A **consumer** reads a stream and remembers its position. A **durable** consumer has a name and keeps its position even when the program that uses it stops.
- The program **acks** (acknowledges) each message after it has processed it. A message that is not acked in time is delivered again: **at-least-once** delivery.

Create `src/bin/jetstream.rs`:

```rust
use async_nats::jetstream::{self, consumer, stream};
use futures::StreamExt;

#[tokio::main]
async fn main() -> Result<(), async_nats::Error> {
    let client = p16_nats_messaging::connect().await?;
    let js = jetstream::new(client);

    // A stream stores every message whose subject matches "orders.>".
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

    // A durable consumer remembers which messages it has already acknowledged.
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

Run it twice:

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

**What happens:**

1. `jetstream::new(client)` gives a JetStream *context*: the entry point for streams and consumers.
2. `get_or_create_stream` creates the stream `ORDERS` the first time and simply returns it later. Names of streams are usually written in capitals.
3. `js.publish(...)` has **two** `.await`s. The first sends the message; the second waits for the server's confirmation that the message is stored. `ack.sequence` is its number in the stream.
4. We published **before** the consumer existed in the first run, and it still got both messages. They were waiting in the stream. This is the big difference from core NATS.
5. `fetch().max_messages(10)` asks for up to 10 waiting messages and ends when there are no more. It is a **pull** consumer: the program asks when it is ready.
6. After processing, `message.ack()` tells the server "done". In the second run, the consumer `billing` delivered only the **new** messages 3 and 4, because 1 and 2 were acknowledged. It remembered its position between two separate program runs.

> ⚠️ **Common mistake:** Thinking at-least-once means exactly once. If your program processes a message and crashes before the ack, the message comes again. Make handlers **idempotent**: "mark order 17 as paid" is safe to do twice, "add 10 euros to the balance" is not. Store the ids of processed messages if needed.

## Common mistakes

> ⚠️ **Common mistake:** Subscribing after publishing in core NATS. If you move the `subscribe` line in `pubsub.rs` below the `flush`, the program waits forever: the two messages were delivered to nobody and are gone. Subscribe first, or use JetStream when messages must not be lost.

> ⚠️ **Common mistake:** The server is not running. `connect` fails:

```text
Error: Error { kind: Io, source: Some(Os { code: 61, kind: ConnectionRefused, message: "Connection refused" }) }
```

Fix: `docker start p16-nats`. Once connected, `async-nats` reconnects by itself if the server restarts.

> ⚠️ **Common mistake:** Forgetting `flush()` before the program ends. `publish` returns as soon as the message is in the client's buffer. If `main` returns at once, the message may never leave the process.

> 🔒 **Security:** Subjects are often built from data (`orders.{country}`). Validate such parts (Lesson 10.4): a value with a dot or `*` in it would create a different subject than you planned. And everyone who can connect to an open NATS server can subscribe to `>` and read **every** message. Use accounts and credentials in production.

## Try it yourself

1. Run `pubsub` again, but subscribe to `orders.>` and also publish to `orders.eu.created` with the payload `order 2`. Read three messages instead of two. You see:

```text
got orders.created -> order 1
got orders.paid -> order 1
got orders.eu.created -> order 2
```

2. Run `queue` three times and compare which worker takes which job.
3. Stop NATS with `docker stop p16-nats` and start it again with `docker start p16-nats`. Run `jetstream` once more: the sequence numbers continue (5 and 6), because the container kept its stored data.

## Exercises

### Exercise 1 — Which subscriptions match? (easy)

For each subject, say which of `orders.*`, `orders.>` and `*.created` receive it: `orders.created`, `orders.eu.created`, `users.created`, `orders`.

<details>
<summary>Hint</summary>

Count the tokens (the parts between the dots). `*` stands for exactly one token, `>` for one or more at the end.

</details>

<details>
<summary>Answer</summary>

| Subject | `orders.*` | `orders.>` | `*.created` |
|---|---|---|---|
| `orders.created` | yes | yes | yes |
| `orders.eu.created` | no (3 tokens) | yes | no (3 tokens) |
| `users.created` | no | no | yes |
| `orders` | no | no (`>` needs at least one token) | no |

Why it works: matching is token by token, and only `>` may cover several tokens.

</details>

### Exercise 2 — Test a request/reply service (medium)

Write an integration test in `tests/messaging.rs`, marked `#[ignore = "needs a running NATS server"]`. It starts a responder on `math.double` that answers with the number doubled, or `error: not a number`. Check `"21"` → `"42"` and `"abc"` → `"error: not a number"`.

<details>
<summary>Hint</summary>

Copy the responder from `request.rs`. Parse the payload with `text.parse::<i64>()`. A `Bytes` value can be compared with a `&str` directly in `assert_eq!`.

</details>

<details>
<summary>Answer</summary>

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

With NATS running, `cargo test --test messaging -- --ignored` prints `test double_service_answers_requests ... ok`.

Why it works: the test is both the service and its client. `flush()` makes sure the server has registered the subscription. Here the request goes over the same connection, so the order is kept anyway; but when the responder and the requester are separate clients, a request sent too early can fail with "no responders".

</details>

### Exercise 3 — Watch a redelivery (hard)

Write `examples/redeliver.rs`: a stream `EMAILS` on `emails.>`, one message, and a durable pull consumer `sender` with `ack_wait: Duration::from_secs(1)`. Fetch the message, **do not** ack it, wait 2 seconds, fetch again and ack. Print how many times the message was delivered (`message.info()?.delivered`).

<details>
<summary>Hint</summary>

Use the `fetch()` builder twice with `.max_messages(1)` and `.expires(Duration::from_secs(3))`, so the second fetch waits long enough for the redelivery.

</details>

<details>
<summary>Answer</summary>

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

Why it works: after a delivery, the server waits `ack_wait` (1 second) for the ack. When it does not come, the server assumes the worker failed and delivers the same message again, with `delivered` increased. This is at-least-once delivery in action.

</details>

## Quiz

1. What is the difference between a normal subscription and a queue group?

<details>
<summary>Answer</summary>

With normal subscriptions every subscriber gets a copy of every message. In a queue group, each message goes to only one member of the group, so the members share the work.

</details>

2. Billing is down for one minute. Orders are published with core NATS `publish`. What happens to those orders, and how does JetStream change that?

<details>
<summary>Answer</summary>

Core NATS delivers only to connected subscribers, so billing never gets them (at-most-once). With JetStream the stream stores the messages, and billing's durable consumer receives them when it comes back.

</details>

3. Why does `js.publish(...)` have two `.await`s?

<details>
<summary>Answer</summary>

The first sends the message; the second waits for the server's acknowledgement that it was stored in the stream. Only after the second one do you know the message is safe.

</details>

4. What does `NoResponders` mean for a request?

<details>
<summary>Answer</summary>

Nobody is subscribed to that subject. The server answers immediately, so the caller does not wait for the timeout.

</details>

5. Why must a JetStream message handler be idempotent?

<details>
<summary>Answer</summary>

Delivery is at-least-once: if the handler crashes or is too slow before the ack, the same message is delivered again. An idempotent handler gives the same result when it runs twice.

</details>

## Summary

- Messages decouple services: the publisher does not know or wait for the receivers, and new receivers can be added later.
- NATS routes messages by subject; `*` matches one token, `>` matches the rest.
- Publish/subscribe sends a copy to every subscriber; request/reply gets one answer through a private inbox; queue groups share work between workers.
- Core NATS is at-most-once: subscribe before publishing and `flush()` before exiting.
- JetStream streams store messages; durable consumers remember their position; unacknowledged messages are delivered again (at-least-once), so handlers must be idempotent.

## Sources and further reading

- [NATS docs — Subject-based messaging](https://docs.nats.io/nats-concepts/subjects) — subjects, tokens and wildcards.
- [NATS docs — Core NATS](https://docs.nats.io/nats-concepts/core-nats) — publish/subscribe, request/reply and queue groups.
- [NATS docs — JetStream](https://docs.nats.io/nats-concepts/jetstream) — streams, consumers, acknowledgements and delivery guarantees.
- [async-nats on docs.rs](https://docs.rs/async-nats) — the Rust client used in this lesson.
- [Docker Hub — nats official image](https://hub.docker.com/_/nats) — the server image and its options.

---

[🏠 Start](../START-HERE.md) · [← 16.10 Caching with Redis](10-caching-with-redis.md) · [📋 Phase 16](README.md) · [16.12 Logs, metrics and traces with OpenTelemetry →](12-logs-metrics-and-traces.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
