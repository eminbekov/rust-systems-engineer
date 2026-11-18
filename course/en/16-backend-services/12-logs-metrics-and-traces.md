<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.11 Messaging with NATS](11-messaging-with-nats.md) · [📋 Phase 16](README.md) · [16.13 Configuration, health checks and containers →](13-configuration-health-checks-and-containers.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/12-logs-metrics-and-traces.md) <!-- nav -->

# Lesson 16.12 — Logs, metrics and traces with OpenTelemetry

> **Phase 16 — Backend services** · Lesson 12 of 16 · ⏱ about 135 minutes

## What you will learn

- What observability means, and which question each of the three signals answers: logs, metrics and traces.
- What OpenTelemetry is: an API, an SDK, the OTLP protocol and the Collector.
- How to send the spans of the `tracing` crate as OpenTelemetry traces, and how to record counters and histograms.
- How to run an OpenTelemetry Collector and Jaeger with Docker Compose and look at a trace.
- How a trace continues from one service to the next with the `traceparent` header, and how to test your instrumentation.

## Before you start

- You finished [Lesson 16.11 — Messaging with NATS](11-messaging-with-nats.md).
- You know the `tracing` crate: events, fields, spans, `#[instrument]`, subscribers and `RUST_LOG` from [Lesson 09.10 — Logging and tracing](../09-testing-debugging-benchmarking/10-logging-and-tracing.md). This lesson builds directly on it.
- You can write an axum service ([Lesson 16.3](03-web-services-with-axum.md)) and use Docker Compose ([Lesson 11.14](../11-linux-shell-and-containers/14-docker-compose.md)).

## New words

| Word | Meaning |
|---|---|
| **observability** | How well you can understand what a running system is doing, from the data it sends out, without changing its code. |
| **telemetry** | The data a program sends about itself: logs, metrics and traces. |
| **signal** | One kind of telemetry. OpenTelemetry has three main signals: traces, metrics, logs. |
| **metric** | A number measured over time, such as "requests per second" or "memory used". |
| **counter** | A metric that only goes up: requests served, errors seen. |
| **histogram** | A metric that records many measured values (for example durations) and groups them into ranges ("buckets"). |
| **attribute** | A key-value pair attached to a span or a metric value, such as `outcome=found`. |
| **trace** | The whole story of one request: a tree of spans, possibly across several services, sharing one trace id. |
| **trace id / span id** | A 16-byte id shared by all spans of one trace / an 8-byte id of one span. |
| **OpenTelemetry** (OTel) | An open standard and a set of libraries for creating and sending telemetry, independent of any vendor. |
| **SDK** (Software Development Kit) | The part of OpenTelemetry that collects, batches and exports the data; the API is the part your code calls. |
| **OTLP** (OpenTelemetry Protocol) | The protocol that carries telemetry from programs to collectors, over gRPC (port 4317) or HTTP (Hypertext Transfer Protocol, port 4318). |
| **exporter** | The component that sends telemetry somewhere: to a collector, to the terminal, to a file. |
| **Collector** | A separate program from the OpenTelemetry project that receives telemetry, processes it and passes it on to back ends. |
| **Jaeger** | An open-source tool that stores traces and shows them in a web UI (User Interface). |
| **context propagation** | Passing the trace id and the current span id to the next service (for example in an HTTP header), so its spans join the same trace. |

## 1. Three questions, three signals

Your service is slow tonight. To find out why, you need data, and different questions need different data:

| Signal | Question it answers | Example |
|---|---|---|
| **Logs** | *What exactly happened* in this one case? | `WARN order not found order_id=500` |
| **Metrics** | *How much, how often*, over time? Is it getting worse? | 3 requests, 1 of them `not_found`; 95% of loads take < 75 ms |
| **Traces** | *Where did the time go* in this one request, across services? | 54 ms total: 53 ms in `load_order`, 21 ms of that in `check_stock` |

They work best together: a metric shows *that* the error rate went up at 21:05, a trace shows *which step* is slow in a failing request, and the logs of that trace show *what* went wrong.

**Observability** is the ability to answer such questions from outside, from the data the system sends. The data itself is **telemetry**.

You already have logs and spans from the `tracing` crate (Lesson 09.10). What is missing: sending spans to a place where you can see a request as a tree, measuring numbers, and doing both in a standard way that every tool understands. That standard is **OpenTelemetry**.

## 2. OpenTelemetry in one picture

```text
  your Rust service                                 Docker Compose
 ┌──────────────────────────────────┐            ┌──────────────────────────┐
 │ tracing spans & events           │            │  OpenTelemetry Collector │
 │   │ tracing-opentelemetry layer  │   OTLP     │  receive → batch → send  │
 │   ▼                              │  (gRPC,    │      │             │     │
 │ OpenTelemetry SDK ──► OTLP       │───4317────►│      ▼             ▼     │
 │ counters, histograms   exporter  │            │   debug        Jaeger    │
 └──────────────────────────────────┘            │  (its log)   (UI 16686)  │
                                                 └──────────────────────────┘
```

- The **API** (Application Programming Interface, crate `opentelemetry`) is what your code calls: create a counter, add 1.
- The **SDK** (Software Development Kit, crate `opentelemetry_sdk`) collects the data, groups it into batches and hands it to an exporter.
- The **exporter** (crate `opentelemetry-otlp`) sends the batches with **OTLP** (OpenTelemetry Protocol), here over gRPC, which you know from Lesson 16.9.
- The **Collector** is a separate program. Your service only knows the collector's address. The collector decides where the data goes: Jaeger for traces, a metrics database, the terminal. You can change that without touching the service.
- **`tracing-opentelemetry`** is the bridge: it turns every `tracing` span into an OpenTelemetry span. Your `#[instrument]` code stays as it is.

## 3. The collector and Jaeger

Create the project folder and two files for Docker:

```bash
cd ~/projects/rust/code
cargo new p16-otel-demo
cd p16-otel-demo
```

`otel-collector.yaml`, the collector's configuration:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317

processors:
  batch: {}

exporters:
  debug:
    verbosity: normal
  otlp_grpc/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug, otlp_grpc/jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
```

**What happens:**

1. **Receivers** take data in. `otlp` listens for OTLP over gRPC on port 4317 inside the container.
2. **Processors** change data on the way. `batch` groups items to send fewer, bigger requests.
3. **Exporters** send data out. `debug` prints a summary into the collector's own log, so we can see what arrived. `otlp_grpc/jaeger` sends OTLP to the container named `jaeger`; the part after `/` is just a name. `insecure: true` means no TLS (Transport Layer Security), which is fine inside one Docker network on your laptop only.
4. **Pipelines** connect them, one per signal: traces go to both exporters, metrics only to `debug`.

`compose.yaml`:

```yaml
services:
  collector:
    image: otel/opentelemetry-collector-contrib:0.165.0
    command: ["--config=/etc/otelcol/config.yaml"]
    volumes:
      - ./otel-collector.yaml:/etc/otelcol/config.yaml:ro
    ports:
      - "127.0.0.1:4317:4317"
    depends_on:
      - jaeger

  jaeger:
    image: jaegertracing/jaeger:2.23.0
    ports:
      - "127.0.0.1:16686:16686"
```

Both images are published by the projects themselves (OpenTelemetry and Jaeger). The collector gets our file as a read-only volume; Jaeger's web UI is on port 16686, and inside the network Jaeger also accepts OTLP on 4317. Start them:

```bash
docker compose up -d
docker compose ps --format 'table {{.Service}}\t{{.Image}}\t{{.Status}}'
```

```text
SERVICE     IMAGE                                          STATUS
collector   otel/opentelemetry-collector-contrib:0.165.0   Up 5 seconds
jaeger      jaegertracing/jaeger:2.23.0                    Up 5 seconds
```

## 4. Sending traces and metrics from Rust

Add the crates:

```bash
cargo add opentelemetry@0.33 opentelemetry_sdk@0.33
cargo add opentelemetry-otlp@0.33 --no-default-features --features grpc-tonic,trace,metrics
cargo add tracing@0.1 tracing-opentelemetry@0.34 axum@0.8
cargo add tracing-subscriber@0.3 --features env-filter
cargo add tower-http@0.7 --features trace
cargo add tokio@1 --features macros,rt-multi-thread,signal,time
```

`--no-default-features --features grpc-tonic,trace,metrics` picks the gRPC transport and only the signals we use. Put the set-up into its own module, `src/telemetry.rs`:

```rust
use opentelemetry::global;
use opentelemetry::trace::TracerProvider as _;
use opentelemetry_sdk::Resource;
use opentelemetry_sdk::metrics::SdkMeterProvider;
use opentelemetry_sdk::trace::SdkTracerProvider;
use tracing_subscriber::EnvFilter;
use tracing_subscriber::layer::SubscriberExt;
use tracing_subscriber::util::SubscriberInitExt;

/// Keeps the providers alive; call `shutdown` before the program ends.
pub struct Telemetry {
    tracer_provider: SdkTracerProvider,
    meter_provider: SdkMeterProvider,
}

pub fn init(service_name: &'static str) -> Result<Telemetry, Box<dyn std::error::Error>> {
    let resource = Resource::builder().with_service_name(service_name).build();

    // Traces: tracing spans -> OpenTelemetry spans -> OTLP over gRPC.
    let span_exporter = opentelemetry_otlp::SpanExporter::builder()
        .with_tonic()
        .build()?;
    let tracer_provider = SdkTracerProvider::builder()
        .with_resource(resource.clone())
        .with_batch_exporter(span_exporter)
        .build();
    let tracer = tracer_provider.tracer(service_name);

    // Metrics: counters and histograms -> OTLP over gRPC, sent periodically.
    let metric_exporter = opentelemetry_otlp::MetricExporter::builder()
        .with_tonic()
        .build()?;
    let meter_provider = SdkMeterProvider::builder()
        .with_resource(resource)
        .with_periodic_exporter(metric_exporter)
        .build();
    global::set_meter_provider(meter_provider.clone());

    // Logs: one `tracing` subscriber with three layers.
    tracing_subscriber::registry()
        .with(EnvFilter::try_from_default_env().unwrap_or_else(|_| "info".into()))
        .with(tracing_subscriber::fmt::layer().with_ansi(false).without_time())
        .with(tracing_opentelemetry::layer().with_tracer(tracer))
        .init();

    Ok(Telemetry {
        tracer_provider,
        meter_provider,
    })
}

impl Telemetry {
    /// Sends everything that is still buffered.
    pub fn shutdown(self) {
        if let Err(err) = self.tracer_provider.shutdown() {
            eprintln!("trace shutdown failed: {err}");
        }
        if let Err(err) = self.meter_provider.shutdown() {
            eprintln!("metrics shutdown failed: {err}");
        }
    }
}
```

**What happens:**

1. A **resource** describes *who* sends the data. `service.name` is the most important attribute: Jaeger lists traces by service name.
2. `SpanExporter::builder().with_tonic()` creates an OTLP exporter over gRPC. With no address in the code, it uses the environment variable `OTEL_EXPORTER_OTLP_ENDPOINT`, or `http://localhost:4317` if it is not set. Configuration from the environment, as in Lesson 08.15.
3. The **tracer provider** owns the pipeline for spans. `with_batch_exporter` collects finished spans in memory and sends them in batches from a background thread, so a request never waits for the network.
4. The **meter provider** does the same for metrics. A *periodic* exporter sends the current values every 60 seconds by default (`OTEL_METRIC_EXPORT_INTERVAL` changes that, in milliseconds). `global::set_meter_provider` lets any code get a meter with `global::meter(...)`.
5. The subscriber has three **layers**, all receiving the same spans and events: the filter from `RUST_LOG` (Lesson 09.10), the `fmt` layer that prints log lines to standard output, and the OpenTelemetry layer that exports spans.
6. `shutdown` flushes what is still in the buffers. Without it, the last batches are lost when the program exits.

We keep **logs** as lines on standard output: the twelve-factor way, and Docker collects them. OpenTelemetry can also export logs (crate `opentelemetry-appender-tracing`); the set-up is the same pattern with a third provider.

Now the service, `src/main.rs`:

```rust
mod telemetry;

use std::time::{Duration, Instant};

use axum::{Router, extract::Path, http::StatusCode, routing::get};
use opentelemetry::{KeyValue, global};
use tower_http::trace::{DefaultMakeSpan, DefaultOnResponse, TraceLayer};
use tracing::{Level, info, instrument, warn};

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let telemetry = telemetry::init("p16-otel-demo")?;

    let app = Router::new().route("/orders/{id}", get(get_order)).layer(
        TraceLayer::new_for_http()
            .make_span_with(DefaultMakeSpan::new().level(Level::INFO))
            .on_response(DefaultOnResponse::new().level(Level::INFO)),
    );

    let listener = tokio::net::TcpListener::bind("127.0.0.1:3000").await?;
    info!("listening on {}", listener.local_addr()?);
    axum::serve(listener, app)
        .with_graceful_shutdown(async {
            tokio::signal::ctrl_c().await.ok();
        })
        .await?;

    info!("shutting down, sending the last telemetry");
    telemetry.shutdown();
    Ok(())
}

#[instrument]
async fn get_order(Path(id): Path<u32>) -> Result<String, StatusCode> {
    let meter = global::meter("p16-otel-demo");
    let requests = meter.u64_counter("orders.requests").build();
    let duration = meter
        .f64_histogram("orders.load.duration")
        .with_unit("ms")
        .build();

    let start = Instant::now();
    let result = load_order(id).await;
    duration.record(start.elapsed().as_secs_f64() * 1000.0, &[]);

    match result {
        Some(order) => {
            requests.add(1, &[KeyValue::new("outcome", "found")]);
            info!(order_id = id, "order sent");
            Ok(order)
        }
        None => {
            requests.add(1, &[KeyValue::new("outcome", "not_found")]);
            warn!(order_id = id, "order not found");
            Err(StatusCode::NOT_FOUND)
        }
    }
}

#[instrument]
async fn load_order(id: u32) -> Option<String> {
    tokio::time::sleep(Duration::from_millis(30)).await; // pretend database query
    if id > 100 {
        return None;
    }
    check_stock(id).await;
    Some(format!("order {id}: 2 books\n"))
}

#[instrument]
async fn check_stock(id: u32) {
    tokio::time::sleep(Duration::from_millis(20)).await; // pretend call to another service
}
```

**What happens:**

1. `TraceLayer` from `tower-http` is middleware (Lesson 16.4) that opens a span named `request` for every HTTP request, with the method and the URI (Uniform Resource Identifier, the path of the request). Its spans are at `DEBUG` level by default; we raise them to `INFO` so our filter keeps them. `on_response` logs the status and the latency of each response.
2. `#[instrument]` on the three functions creates the spans `get_order`, `load_order` and `check_stock`, with the arguments as fields. Because each one is called inside the previous one, they form a **tree**.
3. `global::meter(...)` gives a meter. `u64_counter` creates a **counter**; `add(1, &[KeyValue::new("outcome", "found")])` adds 1 with an **attribute**, so you can count found and not-found requests separately.
4. `f64_histogram` creates a **histogram** with the unit `ms`. `record` adds one measured value. The SDK keeps a count, a sum, the minimum, the maximum and how many values fell into each bucket.
5. `with_graceful_shutdown` (Lesson 14.14) stops the server on Ctrl+C and lets `main` continue, so `telemetry.shutdown()` can send the last data.

> ⚡ **Performance:** Creating an instrument is cheap, because the SDK reuses one with the same name, but not free. In a bigger service, create counters and histograms once at start-up and keep them in your application state (Lesson 16.3). Recording a value is very cheap: no network call happens in the request.

## 5. Looking at the data

Run the service (from `code/p16-otel-demo`, with the Compose services running):

```bash
cargo run
```

In a second terminal, send three requests:

```bash
curl http://127.0.0.1:3000/orders/7
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3000/orders/500
curl http://127.0.0.1:3000/orders/8
```

```text
order 7: 2 books
404
order 8: 2 books
```

Then press Ctrl+C in the first terminal. The service printed:

```text
 INFO p16_otel_demo: listening on 127.0.0.1:3000
 INFO request{method=GET uri=/orders/7 version=HTTP/1.1}:get_order{id=7}: p16_otel_demo: order sent order_id=7
 INFO request{method=GET uri=/orders/7 version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=54 ms status=200
 WARN request{method=GET uri=/orders/500 version=HTTP/1.1}:get_order{id=500}: p16_otel_demo: order not found order_id=500
 INFO request{method=GET uri=/orders/500 version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=32 ms status=404
 INFO request{method=GET uri=/orders/8 version=HTTP/1.1}:get_order{id=8}: p16_otel_demo: order sent order_id=8
 INFO request{method=GET uri=/orders/8 version=HTTP/1.1}: tower_http::trace::on_response: finished processing request latency=55 ms status=200
 INFO p16_otel_demo: shutting down, sending the last telemetry
```

These are the **logs**: each line shows the span path (`request{...}:get_order{...}`) and the fields. Now look at what the collector received:

```bash
docker compose logs --no-log-prefix collector
```

Shortened output (we removed the collector's own start-up lines and, from each span line, the `code.*` and `thread.*` attributes):

```text
ResourceMetrics #0 telemetry.sdk.language=rust telemetry.sdk.name=opentelemetry service.name=p16-otel-demo telemetry.sdk.version=0.33.1
ScopeMetrics #0 p16-otel-demo
orders.requests{outcome=not_found} 1
orders.requests{outcome=found} 2
orders.load.duration{} count=3 sum=140.451458 min=32.127708 max=54.997667 le0=0 le5=0 le10=0 le25=0 le50=1 le75=2 le100=0 le250=0 ...
ResourceTraces #0 telemetry.sdk.language=rust telemetry.sdk.name=opentelemetry service.name=p16-otel-demo telemetry.sdk.version=0.33.1
ScopeTraces #0 p16-otel-demo
check_stock 4e19c4516badf9e33c74ddecc59b582b 23f59c7df0870165 id=7 busy_ns=20958 idle_ns=21047292
load_order 4e19c4516badf9e33c74ddecc59b582b 1cd88f0fc10f8f70 id=7 busy_ns=165708 idle_ns=53145459
get_order 4e19c4516badf9e33c74ddecc59b582b 9d40e1a887e2e25b id=7 busy_ns=700958 idle_ns=53142458
request 4e19c4516badf9e33c74ddecc59b582b 63344e7b0de51f55
load_order 77c66c55ca8ba9d1e9127af8ee78dd3a 4187533278f90f65 id=500 busy_ns=17500 idle_ns=32083250
get_order 77c66c55ca8ba9d1e9127af8ee78dd3a e854eefe9de4b429 id=500 busy_ns=163625 idle_ns=32082292
request 77c66c55ca8ba9d1e9127af8ee78dd3a 6b1a9846f97bbef7
...
```

**What happens:**

1. **Metrics:** the counter has one value per attribute set, 1 not found and 2 found. The histogram shows 3 values with a sum of about 140 ms; `le50=1 le75=2` means one value was ≤ 50 ms and two were between 50 and 75 ms.
2. **Traces:** each line is a span: name, **trace id** (32 hex digits, 16 bytes), **span id** (16 hex digits, 8 bytes), attributes. All four spans of the first request share the trace id `4e19c4…`. The request for order 500 has no `check_stock` span, because `load_order` returned early.
3. `busy_ns` and `idle_ns` come from `tracing-opentelemetry`: time spent running versus waiting (our `sleep`s) in nanoseconds.
4. Children finish before their parents, so they appear first.

Now open **Jaeger** at <http://localhost:16686>. Choose the service `p16-otel-demo` and click **Find Traces**. Open the trace for `/orders/7`. Jaeger draws the spans as a timeline; for our trace it shows:

```text
request          ████████████████████████████████████████  54.0 ms
  get_order      ████████████████████████████████████████  53.8 ms
    load_order   ███████████████████████████████████████   53.3 ms
      check_stock                       ████████████████   21.1 ms
```

At one glance you see where the time goes: 30 ms in the "database" part of `load_order`, then 21 ms in `check_stock`. In a real system, `check_stock` could be in another service, and its spans would still appear in this tree (Exercise 3 shows how).

> 🔒 **Security:** Telemetry leaves your service and is read by many people. Never put passwords, tokens, card numbers or other personal data into span fields, log fields or metric attributes (Lesson 09.10). `#[instrument]` records **all** arguments by default; use `#[instrument(skip(password))]` for secret ones.

> ⚡ **Performance:** Metric attributes must have few possible values (`outcome=found|not_found`). Each new combination creates a new time series in the metrics database. An attribute like `user_id` with millions of values (high *cardinality*) can overload it. Put such ids into spans and logs instead.

## Common mistakes

> ⚠️ **Common mistake:** The collector is not running. The service still works, which is right, but the SDK reports the failed exports:

```text
ERROR opentelemetry_sdk: name="BatchSpanProcessor.ExportError" error="Operation failed: TonicTracesClient export failed with gRPC code: Unavailable: transport error: tcp connect error: tcp connect error: Connection refused (os error 61)"
```

Fix: `docker compose up -d`. Telemetry must never break the service itself.

> ⚠️ **Common mistake:** Ending `main` without `telemetry.shutdown()`. The batch exporter sends every few seconds; the last spans and all metrics since the last periodic export are lost. Short programs especially show nothing in Jaeger.

> ⚠️ **Common mistake:** No `request` span in Jaeger, only `get_order`. `TraceLayer` creates its spans at `DEBUG` level, and the filter `info` drops them. Raise the level with `DefaultMakeSpan::new().level(Level::INFO)`, or set `RUST_LOG=info,tower_http=debug`.

## Try it yourself

1. Send ten requests with `for i in 1 2 3 4 5 6 7 8 9 10; do curl -s http://127.0.0.1:3000/orders/$i > /dev/null; done`, stop the service and find the counter value in the collector log.
2. Run the service with `RUST_LOG=warn cargo run`. The `info` lines disappear from the terminal. Do the spans still reach Jaeger? (No: the filter applies to all layers of this subscriber.)
3. When you are done with this lesson, stop everything with `docker compose down`.

## Exercises

### Exercise 1 — Choose the signal (easy)

For each question, say which signal answers it best: (a) "How many requests per minute failed during the last 24 hours?" (b) "Why did the request with trace id `77c66c…` take 3 seconds?" (c) "What was the exact error message when order 500 failed?"

<details>
<summary>Hint</summary>

Metrics are numbers over time, traces are the timing of one request, logs are the details of one event.

</details>

<details>
<summary>Answer</summary>

(a) Metrics: a counter of failed requests, shown as a rate over time. (b) A trace: its spans show which step took the time. (c) Logs: the `warn!` event with its fields, ideally found through the same trace id.

Why it works: each signal is cheap for one kind of question. Counting in logs or reading timings from metrics is possible but slow and imprecise.

</details>

### Exercise 2 — Test your spans (medium)

Write a test in `src/main.rs` that checks that `check_stock` is a **child** of `load_order`. Use the in-memory exporter of the SDK: add `opentelemetry_sdk` as a dev-dependency with the feature `testing` (`cargo add --dev opentelemetry_sdk@0.33 --features testing`).

<details>
<summary>Hint</summary>

Build an `SdkTracerProvider` with `.with_simple_exporter(InMemorySpanExporter::default())` (keep a clone of the exporter), make a subscriber with only the OpenTelemetry layer, and install it for this test with `tracing::subscriber::set_default`. After `load_order(7).await`, `exporter.get_finished_spans()` returns the spans. Each has `name`, `span_context.span_id()` and `parent_span_id`.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use opentelemetry::trace::TracerProvider as _;
    use opentelemetry_sdk::trace::{InMemorySpanExporter, SdkTracerProvider};
    use tracing_subscriber::layer::SubscriberExt;

    #[tokio::test]
    async fn stock_check_is_a_child_of_load_order() {
        let exporter = InMemorySpanExporter::default();
        let provider = SdkTracerProvider::builder()
            .with_simple_exporter(exporter.clone())
            .build();
        let subscriber = tracing_subscriber::registry()
            .with(tracing_opentelemetry::layer().with_tracer(provider.tracer("test")));
        let _guard = tracing::subscriber::set_default(subscriber);

        super::load_order(7).await;

        let spans = exporter.get_finished_spans().unwrap();
        let names: Vec<&str> = spans.iter().map(|span| span.name.as_ref()).collect();
        assert_eq!(names, ["check_stock", "load_order"]);
        assert_eq!(spans[0].parent_span_id, spans[1].span_context.span_id());
    }
}
```

```text
test tests::stock_check_is_a_child_of_load_order ... ok
```

Why it works: the *simple* exporter exports each span as soon as it ends, into a list in memory. `set_default` installs the subscriber only for the current thread until `_guard` is dropped, and `#[tokio::test]` runs the test on one thread, so other tests are not affected. The child ends first, so it is element 0, and its `parent_span_id` is the id of `load_order`.

</details>

### Exercise 3 — Continue a trace from a caller (hard)

When service A calls service B, A sends the W3C (World Wide Web Consortium) **Trace Context** header:

```text
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
```

It has four parts: version `00`, the trace id, the caller's span id, and flags (`01` = sampled, "record this trace"). Make the service continue such a trace: its `request` span must get the trace id from the header and the caller's span as its parent. Test it with `curl -H 'traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01' http://127.0.0.1:3000/orders/9` and find trace `0af7651916cd43dd8448eb211c80319c` in Jaeger.

<details>
<summary>Hint</summary>

Three pieces: (1) in `telemetry::init`, register the propagator with `global::set_text_map_propagator(TraceContextPropagator::new())` (from `opentelemetry_sdk::propagation`); (2) a small type that implements `opentelemetry::propagation::Extractor` for `&HeaderMap`; (3) your own `make_span` function for `TraceLayer::make_span_with` that extracts the context and calls `span.set_parent(context)` (trait `tracing_opentelemetry::OpenTelemetrySpanExt`).

</details>

<details>
<summary>Answer</summary>

In `src/telemetry.rs`, add `use opentelemetry_sdk::propagation::TraceContextPropagator;` and, after `global::set_meter_provider(...)`:

```rust
    global::set_text_map_propagator(TraceContextPropagator::new());
```

In `src/main.rs`, change the imports to `use opentelemetry::propagation::Extractor;`, `use tower_http::trace::{DefaultOnResponse, TraceLayer};` and `use tracing_opentelemetry::OpenTelemetrySpanExt;`, use `.make_span_with(make_span)` in the `TraceLayer`, and add:

```rust
/// Lets the OpenTelemetry propagator read HTTP headers.
struct HeaderExtractor<'a>(&'a axum::http::HeaderMap);

impl Extractor for HeaderExtractor<'_> {
    fn get(&self, key: &str) -> Option<&str> {
        self.0.get(key).and_then(|value| value.to_str().ok())
    }

    fn keys(&self) -> Vec<&str> {
        self.0.keys().map(|name| name.as_str()).collect()
    }
}

/// Creates the request span and continues the caller's trace, if any.
fn make_span(request: &axum::http::Request<axum::body::Body>) -> tracing::Span {
    let parent = global::get_text_map_propagator(|propagator| {
        propagator.extract(&HeaderExtractor(request.headers()))
    });
    let span = tracing::info_span!(
        "request",
        method = %request.method(),
        uri = %request.uri(),
    );
    if let Err(err) = span.set_parent(parent) {
        warn!("could not set the parent span: {err}");
    }
    span
}
```

After the `curl`, Jaeger has trace `0af7651916cd43dd8448eb211c80319c`, and its `request` span refers to the parent span `b7ad6b7169203331`, which lives in the (imaginary) caller.

Why it works: the propagator knows the `traceparent` format. `extract` reads the header through `HeaderExtractor` and builds a context with the remote span. `set_parent` makes our `request` span a child of it, so it takes the same trace id, and all our spans below follow. Without a header, `extract` returns an empty context and a new trace begins, as before. The calling side does the opposite with an `Injector` that writes the header into the outgoing request.

</details>

## Quiz

1. Which signal would you use to see that the error rate doubled at 21:05?

<details>
<summary>Answer</summary>

Metrics: a counter of errors (with an attribute such as `outcome`), viewed as a rate over time.

</details>

2. What do the collector's receivers, processors and exporters do?

<details>
<summary>Answer</summary>

Receivers accept telemetry (here OTLP on port 4317), processors change it on the way (here `batch` groups it), exporters send it on (to the debug log, to Jaeger). Pipelines connect them per signal.

</details>

3. How do the spans of one request in Jaeger know that they belong together?

<details>
<summary>Answer</summary>

They share one trace id, and each span (except the root) stores the span id of its parent. Jaeger builds the tree from these parent links.

</details>

4. Why does the service call `telemetry.shutdown()` before exiting?

<details>
<summary>Answer</summary>

The SDK sends data in batches and periodically. `shutdown` flushes the buffers; without it, the last spans and metric values are lost.

</details>

5. Why is `user_id` a bad metric attribute but fine as a span field?

<details>
<summary>Answer</summary>

Each distinct attribute value creates a new time series; millions of users mean millions of series and an overloaded metrics database (high cardinality). A span is one record of one request, so an id there costs nothing extra (as long as it is not secret personal data).

</details>

6. What does the `traceparent` header contain, and what is it for?

<details>
<summary>Answer</summary>

A version, the trace id, the caller's span id and flags. It carries the trace context to the next service, so the next service's spans join the same trace.

</details>

## Summary

- Logs tell what happened, metrics tell how much and how often, traces tell where the time went in one request.
- OpenTelemetry standardises telemetry: the API (`opentelemetry`), the SDK (`opentelemetry_sdk`), OTLP export (`opentelemetry-otlp`) and the Collector.
- `tracing-opentelemetry` turns `tracing` spans into OpenTelemetry spans; `#[instrument]` and `TraceLayer` give you request trees almost for free.
- Counters only go up; histograms record distributions of values. Keep metric attributes low-cardinality.
- The Collector receives, processes and exports; Jaeger shows traces as timelines. Configure the endpoint with `OTEL_EXPORTER_OTLP_ENDPOINT` and always shut the providers down.
- The W3C `traceparent` header propagates a trace across services; test your spans with the in-memory exporter.

## Sources and further reading

- [OpenTelemetry — Observability primer](https://opentelemetry.io/docs/concepts/observability-primer/) — the ideas of observability, signals and spans.
- [OpenTelemetry — Rust](https://opentelemetry.io/docs/languages/rust/) — the official getting-started pages for the Rust libraries.
- [OpenTelemetry Collector — Configuration](https://opentelemetry.io/docs/collector/configuration/) — receivers, processors, exporters and pipelines in detail.
- [W3C Trace Context](https://www.w3.org/TR/trace-context/) — the standard behind the `traceparent` header.
- [tracing-opentelemetry on docs.rs](https://docs.rs/tracing-opentelemetry) and [opentelemetry-otlp on docs.rs](https://docs.rs/opentelemetry-otlp) — the APIs used in this lesson.
- [Jaeger documentation](https://www.jaegertracing.io/docs/latest/) — how to use the Jaeger UI and run it in production.

---

[🏠 Start](../START-HERE.md) · [← 16.11 Messaging with NATS](11-messaging-with-nats.md) · [📋 Phase 16](README.md) · [16.13 Configuration, health checks and containers →](13-configuration-health-checks-and-containers.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
