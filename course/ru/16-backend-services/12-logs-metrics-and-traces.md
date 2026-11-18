<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 16.11 Обмен сообщениями через NATS](11-messaging-with-nats.md) · [📋 Этап 16](README.md) · [16.13 Конфигурация, проверки работоспособности и контейнеры →](13-configuration-health-checks-and-containers.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/16-backend-services/12-logs-metrics-and-traces.md) <!-- nav -->

# Урок 16.12 — Логи, метрики и трассировки с OpenTelemetry

> **Этап 16 — Бэкенд-сервисы** · Урок 12 из 16 · ⏱ около 135 минут

## Что вы узнаете

- Что значит наблюдаемость и на какой вопрос отвечает каждый из трёх сигналов: логи, метрики и трассировки.
- Что такое OpenTelemetry: API, SDK, протокол OTLP и Collector.
- Как отправлять спаны крейта `tracing` как трассировки OpenTelemetry и как записывать счётчики и гистограммы.
- Как запустить OpenTelemetry Collector и Jaeger через Docker Compose и посмотреть на трассировку.
- Как трассировка продолжается из одного сервиса в другой через заголовок `traceparent` и как тестировать инструментирование.

## Перед началом

- Вы прошли [Урок 16.11 — Обмен сообщениями через NATS](11-messaging-with-nats.md).
- Вы знаете крейт `tracing`: события, поля, спаны, `#[instrument]`, подписчики и `RUST_LOG` из [Урока 09.10 — Логирование и трассировка](../09-testing-debugging-benchmarking/10-logging-and-tracing.md). Этот урок прямо продолжает его.
- Вы умеете писать сервис на axum ([Урок 16.3](03-web-services-with-axum.md)) и пользоваться Docker Compose ([Урок 11.14](../11-linux-shell-and-containers/14-docker-compose.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **наблюдаемость** (observability) | Насколько хорошо можно понять, что делает работающая система, по данным, которые она отправляет, не меняя её код. |
| **телеметрия** (telemetry) | Данные, которые программа отправляет о себе: логи, метрики и трассировки. |
| **сигнал** (signal) | Один вид телеметрии. У OpenTelemetry три основных сигнала: трассировки, метрики, логи. |
| **метрика** (metric) | Число, измеряемое во времени, например «запросов в секунду» или «занятая память». |
| **счётчик** (counter) | Метрика, которая только растёт: обслуженные запросы, увиденные ошибки. |
| **гистограмма** (histogram) | Метрика, которая записывает много измеренных значений (например, длительностей) и раскладывает их по диапазонам («корзинам», buckets). |
| **атрибут** (attribute) | Пара «ключ — значение», прикреплённая к спану или к значению метрики, например `outcome=found`. |
| **трассировка** (trace) | Вся история одного запроса: дерево спанов, возможно через несколько сервисов, с одним общим trace id. |
| **trace id / span id** | 16-байтовый идентификатор, общий для всех спанов одной трассировки / 8-байтовый идентификатор одного спана. |
| **OpenTelemetry** (OTel) | Открытый стандарт и набор библиотек для создания и отправки телеметрии, не привязанный к поставщику. |
| **SDK** (Software Development Kit, набор средств разработки) | Часть OpenTelemetry, которая собирает, группирует и экспортирует данные; API — часть, которую вызывает ваш код. |
| **OTLP** (OpenTelemetry Protocol, протокол OpenTelemetry) | Протокол, который несёт телеметрию от программ к коллекторам, поверх gRPC (порт 4317) или HTTP (Hypertext Transfer Protocol, протокол передачи гипертекста; порт 4318). |
| **экспортёр** (exporter) | Компонент, который отправляет телеметрию куда-то: в коллектор, в терминал, в файл. |
| **Collector** | Отдельная программа проекта OpenTelemetry, которая принимает телеметрию, обрабатывает её и передаёт дальше в хранилища. |
| **Jaeger** | Инструмент с открытым кодом, который хранит трассировки и показывает их в веб-интерфейсе UI (User Interface, пользовательский интерфейс). |
| **распространение контекста** (context propagation) | Передача trace id и id текущего спана следующему сервису (например, в HTTP-заголовке), чтобы его спаны вошли в ту же трассировку. |

## 1. Три вопроса, три сигнала

Сегодня вечером ваш сервис медленный. Чтобы понять почему, нужны данные, и разным вопросам нужны разные данные:

| Сигнал | На какой вопрос отвечает | Пример |
|---|---|---|
| **Логи** | *Что именно произошло* в этом одном случае? | `WARN order not found order_id=500` |
| **Метрики** | *Сколько, как часто*, во времени? Становится ли хуже? | 3 запроса, 1 из них `not_found`; 95% загрузок занимают < 75 мс |
| **Трассировки** | *Куда ушло время* в этом одном запросе, через сервисы? | всего 54 мс: 53 мс в `load_order`, из них 21 мс в `check_stock` |

Лучше всего они работают вместе: метрика показывает, *что* доля ошибок выросла в 21:05, трассировка показывает, *какой шаг* медленный в падающем запросе, а логи этой трассировки показывают, *что* пошло не так.

**Наблюдаемость** — это способность отвечать на такие вопросы снаружи, по данным, которые отправляет система. Сами данные — это **телеметрия**.

Логи и спаны у вас уже есть из крейта `tracing` (Урок 09.10). Чего не хватает: отправлять спаны туда, где запрос виден как дерево, измерять числа и делать и то и другое стандартным способом, который понимает любой инструмент. Этот стандарт — **OpenTelemetry**.

## 2. OpenTelemetry на одной картинке

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

- **API** (Application Programming Interface, программный интерфейс; крейт `opentelemetry`) — то, что вызывает ваш код: создать счётчик, прибавить 1.
- **SDK** (Software Development Kit, крейт `opentelemetry_sdk`) собирает данные, группирует их в пакеты и передаёт экспортёру.
- **Экспортёр** (крейт `opentelemetry-otlp`) отправляет пакеты по **OTLP** (OpenTelemetry Protocol), здесь поверх gRPC, знакомого вам по Уроку 16.9.
- **Collector** — отдельная программа. Ваш сервис знает только адрес коллектора. Коллектор решает, куда пойдут данные: трассировки в Jaeger, метрики в базу метрик, в терминал. Это можно менять, не трогая сервис.
- **`tracing-opentelemetry`** — мост: он превращает каждый спан `tracing` в спан OpenTelemetry. Ваш код с `#[instrument]` остаётся как есть.

## 3. Коллектор и Jaeger

Создайте папку проекта и два файла для Docker:

```bash
cd ~/projects/rust/code
cargo new p16-otel-demo
cd p16-otel-demo
```

`otel-collector.yaml` — конфигурация коллектора:

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

**Что происходит:**

1. **Приёмники** (receivers) принимают данные. `otlp` слушает OTLP поверх gRPC на порту 4317 внутри контейнера.
2. **Процессоры** (processors) меняют данные по дороге. `batch` группирует элементы, чтобы отправлять меньше запросов, но крупнее.
3. **Экспортёры** (exporters) отправляют данные дальше. `debug` печатает сводку в собственный лог коллектора, чтобы мы видели, что пришло. `otlp_grpc/jaeger` отправляет OTLP в контейнер с именем `jaeger`; часть после `/` — просто имя. `insecure: true` означает «без TLS» (Transport Layer Security, протокол защиты транспортного уровня), что допустимо только внутри одной сети Docker на вашем ноутбуке.
4. **Конвейеры** (pipelines) соединяют их, по одному на сигнал: трассировки идут в оба экспортёра, метрики — только в `debug`.

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

Оба образа опубликованы самими проектами (OpenTelemetry и Jaeger). Коллектор получает наш файл как том только для чтения; веб-интерфейс Jaeger — на порту 16686, а внутри сети Jaeger тоже принимает OTLP на 4317. Запустите их:

```bash
docker compose up -d
docker compose ps --format 'table {{.Service}}\t{{.Image}}\t{{.Status}}'
```

```text
SERVICE     IMAGE                                          STATUS
collector   otel/opentelemetry-collector-contrib:0.165.0   Up 5 seconds
jaeger      jaegertracing/jaeger:2.23.0                    Up 5 seconds
```

## 4. Отправка трассировок и метрик из Rust

Добавьте крейты:

```bash
cargo add opentelemetry@0.33 opentelemetry_sdk@0.33
cargo add opentelemetry-otlp@0.33 --no-default-features --features grpc-tonic,trace,metrics
cargo add tracing@0.1 tracing-opentelemetry@0.34 axum@0.8
cargo add tracing-subscriber@0.3 --features env-filter
cargo add tower-http@0.7 --features trace
cargo add tokio@1 --features macros,rt-multi-thread,signal,time
```

`--no-default-features --features grpc-tonic,trace,metrics` выбирает транспорт gRPC и только те сигналы, которые мы используем. Вынесите настройку в отдельный модуль `src/telemetry.rs`:

```rust
use opentelemetry::global;
use opentelemetry::trace::TracerProvider as _;
use opentelemetry_sdk::Resource;
use opentelemetry_sdk::metrics::SdkMeterProvider;
use opentelemetry_sdk::trace::SdkTracerProvider;
use tracing_subscriber::EnvFilter;
use tracing_subscriber::layer::SubscriberExt;
use tracing_subscriber::util::SubscriberInitExt;

/// Держит провайдеры живыми; вызовите `shutdown` перед завершением программы.
pub struct Telemetry {
    tracer_provider: SdkTracerProvider,
    meter_provider: SdkMeterProvider,
}

pub fn init(service_name: &'static str) -> Result<Telemetry, Box<dyn std::error::Error>> {
    let resource = Resource::builder().with_service_name(service_name).build();

    // Трассировки: спаны tracing -> спаны OpenTelemetry -> OTLP поверх gRPC.
    let span_exporter = opentelemetry_otlp::SpanExporter::builder()
        .with_tonic()
        .build()?;
    let tracer_provider = SdkTracerProvider::builder()
        .with_resource(resource.clone())
        .with_batch_exporter(span_exporter)
        .build();
    let tracer = tracer_provider.tracer(service_name);

    // Метрики: счётчики и гистограммы -> OTLP поверх gRPC, отправляются периодически.
    let metric_exporter = opentelemetry_otlp::MetricExporter::builder()
        .with_tonic()
        .build()?;
    let meter_provider = SdkMeterProvider::builder()
        .with_resource(resource)
        .with_periodic_exporter(metric_exporter)
        .build();
    global::set_meter_provider(meter_provider.clone());

    // Логи: один подписчик `tracing` с тремя слоями.
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
    /// Отправляет всё, что ещё лежит в буферах.
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

**Что происходит:**

1. **Ресурс** (resource) описывает, *кто* отправляет данные. `service.name` — самый важный атрибут: Jaeger перечисляет трассировки по имени сервиса.
2. `SpanExporter::builder().with_tonic()` создаёт экспортёр OTLP поверх gRPC. Если адреса нет в коде, он берёт переменную окружения `OTEL_EXPORTER_OTLP_ENDPOINT` или `http://localhost:4317`, если она не задана. Конфигурация из окружения, как в Уроке 08.15.
3. **Провайдер трассировщика** (tracer provider) владеет конвейером для спанов. `with_batch_exporter` собирает законченные спаны в памяти и отправляет их пакетами из фонового потока, так что запрос никогда не ждёт сеть.
4. **Провайдер метрик** (meter provider) делает то же для метрик. *Периодический* экспортёр отправляет текущие значения по умолчанию каждые 60 секунд (`OTEL_METRIC_EXPORT_INTERVAL` меняет это, в миллисекундах). `global::set_meter_provider` позволяет любому коду получить meter через `global::meter(...)`.
5. У подписчика три **слоя**, и все получают одни и те же спаны и события: фильтр из `RUST_LOG` (Урок 09.10), слой `fmt`, который печатает строки логов в стандартный вывод, и слой OpenTelemetry, который экспортирует спаны.
6. `shutdown` выталкивает то, что ещё в буферах. Без него последние пакеты теряются при выходе программы.

**Логи** мы оставляем строками в стандартном выводе: так требуют двенадцать факторов, и Docker их собирает. OpenTelemetry умеет экспортировать и логи (крейт `opentelemetry-appender-tracing`); настройка следует тому же образцу с третьим провайдером.

Теперь сервис, `src/main.rs`:

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
    tokio::time::sleep(Duration::from_millis(30)).await; // как будто запрос к базе данных
    if id > 100 {
        return None;
    }
    check_stock(id).await;
    Some(format!("order {id}: 2 books\n"))
}

#[instrument]
async fn check_stock(id: u32) {
    tokio::time::sleep(Duration::from_millis(20)).await; // как будто вызов другого сервиса
}
```

**Что происходит:**

1. `TraceLayer` из `tower-http` — это middleware (промежуточный слой, Урок 16.4), который открывает спан с именем `request` для каждого HTTP-запроса, с методом и URI (Uniform Resource Identifier, унифицированный идентификатор ресурса — путь запроса). По умолчанию его спаны имеют уровень `DEBUG`; мы поднимаем их до `INFO`, чтобы наш фильтр их оставил. `on_response` пишет в лог статус и задержку каждого ответа.
2. `#[instrument]` на трёх функциях создаёт спаны `get_order`, `load_order` и `check_stock` с аргументами в качестве полей. Поскольку каждая вызывается внутри предыдущей, они образуют **дерево**.
3. `global::meter(...)` даёт meter (измеритель). `u64_counter` создаёт **счётчик**; `add(1, &[KeyValue::new("outcome", "found")])` прибавляет 1 с **атрибутом**, так что найденные и ненайденные запросы считаются отдельно.
4. `f64_histogram` создаёт **гистограмму** с единицей `ms`. `record` добавляет одно измеренное значение. SDK хранит количество, сумму, минимум, максимум и сколько значений попало в каждую корзину.
5. `with_graceful_shutdown` (Урок 14.14) останавливает сервер по Ctrl+C и даёт `main` продолжиться, чтобы `telemetry.shutdown()` отправил последние данные.

> ⚡ **Производительность:** создать инструмент дёшево, потому что SDK переиспользует инструмент с тем же именем, но не бесплатно. В сервисе побольше создавайте счётчики и гистограммы один раз при запуске и храните их в состоянии приложения (Урок 16.3). Запись значения очень дешёвая: во время запроса никакого сетевого вызова не происходит.

## 5. Смотрим на данные

Запустите сервис (из `code/p16-otel-demo`, при запущенных сервисах Compose):

```bash
cargo run
```

Во втором терминале отправьте три запроса:

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

Затем нажмите Ctrl+C в первом терминале. Сервис напечатал:

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

Это **логи**: каждая строка показывает путь спанов (`request{...}:get_order{...}`) и поля. Теперь посмотрите, что получил коллектор:

```bash
docker compose logs --no-log-prefix collector
```

Сокращённый вывод (мы убрали строки запуска самого коллектора, а из каждой строки спана — атрибуты `code.*` и `thread.*`):

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

**Что происходит:**

1. **Метрики:** у счётчика по одному значению на набор атрибутов: 1 «не найдено» и 2 «найдено». Гистограмма показывает 3 значения с суммой около 140 мс; `le50=1 le75=2` значит, что одно значение было ≤ 50 мс, а два — между 50 и 75 мс.
2. **Трассировки:** каждая строка — спан: имя, **trace id** (32 шестнадцатеричные цифры, 16 байт), **span id** (16 шестнадцатеричных цифр, 8 байт), атрибуты. Все четыре спана первого запроса делят trace id `4e19c4…`. У запроса заказа 500 нет спана `check_stock`, потому что `load_order` вернулась раньше.
3. `busy_ns` и `idle_ns` добавляет `tracing-opentelemetry`: время работы и время ожидания (наши `sleep`) в наносекундах.
4. Дочерние спаны заканчиваются раньше родителей, поэтому идут первыми.

Теперь откройте **Jaeger** по адресу <http://localhost:16686>. Выберите сервис `p16-otel-demo` и нажмите **Find Traces**. Откройте трассировку для `/orders/7`. Jaeger рисует спаны как временную шкалу; для нашей трассировки он показывает:

```text
request          ████████████████████████████████████████  54.0 ms
  get_order      ████████████████████████████████████████  53.8 ms
    load_order   ███████████████████████████████████████   53.3 ms
      check_stock                       ████████████████   21.1 ms
```

С одного взгляда видно, куда уходит время: 30 мс в «базе данных» внутри `load_order`, затем 21 мс в `check_stock`. В настоящей системе `check_stock` мог бы быть в другом сервисе, и его спаны всё равно появились бы в этом дереве (Упражнение 3 показывает как).

> 🔒 **Безопасность:** телеметрия покидает ваш сервис, и её читают многие люди. Никогда не кладите пароли, токены, номера карт и другие персональные данные в поля спанов, поля логов или атрибуты метрик (Урок 09.10). `#[instrument]` по умолчанию записывает **все** аргументы; для секретных используйте `#[instrument(skip(password))]`.

> ⚡ **Производительность:** у атрибутов метрик должно быть мало возможных значений (`outcome=found|not_found`). Каждая новая комбинация создаёт новый временной ряд в базе метрик. Атрибут вроде `user_id` с миллионами значений (высокая *кардинальность*) может её перегрузить. Кладите такие id в спаны и логи.

## Частые ошибки

> ⚠️ **Частая ошибка:** коллектор не запущен. Сервис всё равно работает, и это правильно, но SDK сообщает о неудачных экспортах:

```text
ERROR opentelemetry_sdk: name="BatchSpanProcessor.ExportError" error="Operation failed: TonicTracesClient export failed with gRPC code: Unavailable: transport error: tcp connect error: tcp connect error: Connection refused (os error 61)"
```

Исправление: `docker compose up -d`. Телеметрия никогда не должна ломать сам сервис.

> ⚠️ **Частая ошибка:** завершить `main` без `telemetry.shutdown()`. Пакетный экспортёр отправляет данные раз в несколько секунд; последние спаны и все метрики с последнего периодического экспорта теряются. Короткие программы тогда вообще ничего не показывают в Jaeger.

> ⚠️ **Частая ошибка:** в Jaeger нет спана `request`, только `get_order`. `TraceLayer` создаёт свои спаны на уровне `DEBUG`, и фильтр `info` их отбрасывает. Поднимите уровень через `DefaultMakeSpan::new().level(Level::INFO)` или задайте `RUST_LOG=info,tower_http=debug`.

## Попробуйте сами

1. Отправьте десять запросов командой `for i in 1 2 3 4 5 6 7 8 9 10; do curl -s http://127.0.0.1:3000/orders/$i > /dev/null; done`, остановите сервис и найдите значение счётчика в логе коллектора.
2. Запустите сервис с `RUST_LOG=warn cargo run`. Строки `info` исчезают из терминала. Доходят ли спаны до Jaeger? (Нет: фильтр действует на все слои этого подписчика.)
3. Закончив урок, остановите всё командой `docker compose down`.

## Упражнения

### Упражнение 1 — Выберите сигнал (легко)

Для каждого вопроса скажите, какой сигнал отвечает на него лучше всего: (a) «Сколько запросов в минуту падало за последние 24 часа?» (b) «Почему запрос с trace id `77c66c…` занял 3 секунды?» (c) «Какое точное сообщение об ошибке было, когда упал заказ 500?»

<details>
<summary>Подсказка</summary>

Метрики — числа во времени, трассировки — тайминги одного запроса, логи — подробности одного события.

</details>

<details>
<summary>Ответ</summary>

(a) Метрики: счётчик неудачных запросов, показанный как скорость во времени. (b) Трассировка: её спаны показывают, какой шаг занял время. (c) Логи: событие `warn!` с его полями, лучше всего найденное по тому же trace id.

Почему это работает: каждый сигнал дёшев для своего типа вопросов. Считать по логам или читать тайминги по метрикам можно, но медленно и неточно.

</details>

### Упражнение 2 — Тест для ваших спанов (средне)

Напишите тест в `src/main.rs`, который проверяет, что `check_stock` — **дочерний** спан `load_order`. Используйте экспортёр в памяти из SDK: добавьте `opentelemetry_sdk` как dev-зависимость с фичей `testing` (`cargo add --dev opentelemetry_sdk@0.33 --features testing`).

<details>
<summary>Подсказка</summary>

Постройте `SdkTracerProvider` с `.with_simple_exporter(InMemorySpanExporter::default())` (сохраните клон экспортёра), сделайте подписчика только со слоем OpenTelemetry и установите его для этого теста через `tracing::subscriber::set_default`. После `load_order(7).await` метод `exporter.get_finished_spans()` возвращает спаны. У каждого есть `name`, `span_context.span_id()` и `parent_span_id`.

</details>

<details>
<summary>Ответ</summary>

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

Почему это работает: *простой* (simple) экспортёр экспортирует каждый спан сразу по его окончании, в список в памяти. `set_default` устанавливает подписчика только для текущего потока, пока не удалён `_guard`, а `#[tokio::test]` выполняет тест в одном потоке, поэтому другие тесты не затронуты. Дочерний спан заканчивается первым, поэтому он элемент 0, и его `parent_span_id` — это id `load_order`.

</details>

### Упражнение 3 — Продолжить трассировку вызывающего (сложно)

Когда сервис A вызывает сервис B, A отправляет заголовок W3C (World Wide Web Consortium, Консорциум Всемирной паутины) **Trace Context**:

```text
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
```

В нём четыре части: версия `00`, trace id, span id вызывающего и флаги (`01` = sampled, «записывать эту трассировку»). Сделайте так, чтобы сервис продолжал такую трассировку: его спан `request` должен получить trace id из заголовка и спан вызывающего в качестве родителя. Проверьте командой `curl -H 'traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01' http://127.0.0.1:3000/orders/9` и найдите трассировку `0af7651916cd43dd8448eb211c80319c` в Jaeger.

<details>
<summary>Подсказка</summary>

Три части: (1) в `telemetry::init` зарегистрируйте пропагатор через `global::set_text_map_propagator(TraceContextPropagator::new())` (из `opentelemetry_sdk::propagation`); (2) небольшой тип, реализующий `opentelemetry::propagation::Extractor` для `&HeaderMap`; (3) своя функция `make_span` для `TraceLayer::make_span_with`, которая извлекает контекст и вызывает `span.set_parent(context)` (трейт `tracing_opentelemetry::OpenTelemetrySpanExt`).

</details>

<details>
<summary>Ответ</summary>

В `src/telemetry.rs` добавьте `use opentelemetry_sdk::propagation::TraceContextPropagator;` и после `global::set_meter_provider(...)`:

```rust
    global::set_text_map_propagator(TraceContextPropagator::new());
```

В `src/main.rs` измените импорты на `use opentelemetry::propagation::Extractor;`, `use tower_http::trace::{DefaultOnResponse, TraceLayer};` и `use tracing_opentelemetry::OpenTelemetrySpanExt;`, используйте `.make_span_with(make_span)` в `TraceLayer` и добавьте:

```rust
/// Позволяет пропагатору OpenTelemetry читать HTTP-заголовки.
struct HeaderExtractor<'a>(&'a axum::http::HeaderMap);

impl Extractor for HeaderExtractor<'_> {
    fn get(&self, key: &str) -> Option<&str> {
        self.0.get(key).and_then(|value| value.to_str().ok())
    }

    fn keys(&self) -> Vec<&str> {
        self.0.keys().map(|name| name.as_str()).collect()
    }
}

/// Создаёт спан запроса и продолжает трассировку вызывающего, если она есть.
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

После `curl` в Jaeger есть трассировка `0af7651916cd43dd8448eb211c80319c`, и её спан `request` ссылается на родительский спан `b7ad6b7169203331`, который живёт у (воображаемого) вызывающего.

Почему это работает: пропагатор знает формат `traceparent`. `extract` читает заголовок через `HeaderExtractor` и строит контекст с удалённым спаном. `set_parent` делает наш спан `request` его дочерним, поэтому он получает тот же trace id, а все наши спаны ниже следуют за ним. Без заголовка `extract` возвращает пустой контекст, и начинается новая трассировка, как раньше. Вызывающая сторона делает обратное с помощью `Injector`, который записывает заголовок в исходящий запрос.

</details>

## Вопросы для самопроверки

1. Каким сигналом вы увидите, что доля ошибок удвоилась в 21:05?

<details>
<summary>Ответ</summary>

Метриками: счётчиком ошибок (с атрибутом вроде `outcome`), который смотрят как скорость во времени.

</details>

2. Что делают приёмники, процессоры и экспортёры коллектора?

<details>
<summary>Ответ</summary>

Приёмники принимают телеметрию (здесь OTLP на порту 4317), процессоры меняют её по дороге (здесь `batch` группирует), экспортёры отправляют её дальше (в отладочный лог, в Jaeger). Конвейеры соединяют их для каждого сигнала.

</details>

3. Как спаны одного запроса в Jaeger узнают, что они вместе?

<details>
<summary>Ответ</summary>

У них общий trace id, и каждый спан (кроме корневого) хранит span id своего родителя. Jaeger строит дерево по этим ссылкам на родителей.

</details>

4. Зачем сервис вызывает `telemetry.shutdown()` перед выходом?

<details>
<summary>Ответ</summary>

SDK отправляет данные пакетами и периодически. `shutdown` выталкивает буферы; без него последние спаны и значения метрик теряются.

</details>

5. Почему `user_id` — плохой атрибут метрики, но нормальное поле спана?

<details>
<summary>Ответ</summary>

Каждое отдельное значение атрибута создаёт новый временной ряд; миллионы пользователей означают миллионы рядов и перегруженную базу метрик (высокая кардинальность). Спан — одна запись об одном запросе, поэтому id там ничего лишнего не стоит (если это не секретные персональные данные).

</details>

6. Что содержит заголовок `traceparent` и зачем он нужен?

<details>
<summary>Ответ</summary>

Версию, trace id, span id вызывающего и флаги. Он переносит контекст трассировки в следующий сервис, чтобы спаны следующего сервиса вошли в ту же трассировку.

</details>

## Итоги

- Логи говорят, что произошло, метрики — сколько и как часто, трассировки — куда ушло время в одном запросе.
- OpenTelemetry стандартизирует телеметрию: API (`opentelemetry`), SDK (`opentelemetry_sdk`), экспорт OTLP (`opentelemetry-otlp`) и Collector.
- `tracing-opentelemetry` превращает спаны `tracing` в спаны OpenTelemetry; `#[instrument]` и `TraceLayer` дают деревья запросов почти даром.
- Счётчики только растут; гистограммы записывают распределения значений. Держите кардинальность атрибутов метрик низкой.
- Collector принимает, обрабатывает и экспортирует; Jaeger показывает трассировки как временные шкалы. Задавайте адрес через `OTEL_EXPORTER_OTLP_ENDPOINT` и всегда останавливайте провайдеры.
- Заголовок W3C `traceparent` распространяет трассировку через сервисы; проверяйте свои спаны экспортёром в памяти.

## Источники и что почитать

- [OpenTelemetry — Observability primer](https://opentelemetry.io/docs/concepts/observability-primer/) — идеи наблюдаемости, сигналов и спанов.
- [OpenTelemetry — Rust](https://opentelemetry.io/docs/languages/rust/) — официальные страницы для начала работы с библиотеками для Rust.
- [OpenTelemetry Collector — Configuration](https://opentelemetry.io/docs/collector/configuration/) — приёмники, процессоры, экспортёры и конвейеры подробно.
- [W3C Trace Context](https://www.w3.org/TR/trace-context/) — стандарт, на котором основан заголовок `traceparent`.
- [tracing-opentelemetry на docs.rs](https://docs.rs/tracing-opentelemetry) и [opentelemetry-otlp на docs.rs](https://docs.rs/opentelemetry-otlp) — API, используемые в этом уроке.
- [Документация Jaeger](https://www.jaegertracing.io/docs/latest/) — как пользоваться интерфейсом Jaeger и запускать его в продакшене.

---

[🏠 Начало](../START-HERE.md) · [← 16.11 Обмен сообщениями через NATS](11-messaging-with-nats.md) · [📋 Этап 16](README.md) · [16.13 Конфигурация, проверки работоспособности и контейнеры →](13-configuration-health-checks-and-containers.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
