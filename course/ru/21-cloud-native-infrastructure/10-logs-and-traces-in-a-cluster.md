<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 21.9 Метрики с Prometheus и Grafana](09-metrics-with-prometheus-and-grafana.md) · [📋 Этап 21](README.md) · [21.11 Основы SRE: SLI, SLO и бюджет ошибок →](11-sre-basics-slis-slos-and-error-budgets.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/21-cloud-native-infrastructure/10-logs-and-traces-in-a-cluster.md) <!-- nav -->

# Урок 21.10 — Логи и трассировки в кластере

> **Этап 21 — Облачная (cloud-native) инфраструктура** · Урок 10 из 14 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Где на узле лежат логи пода, что на самом деле читает `kubectl logs` и почему эти логи исчезают.
- Как хорошо читать логи: один под, все поды Deployment, последние минуты, предыдущий сбой.
- Как писать из Rust структурированные логи в JSON и фильтровать их через `jq`.
- Как агент узла (OpenTelemetry Collector в виде DaemonSet) собирает логи каждого пода.
- Как трассировки идут из сервиса в коллектор и в Jaeger внутри кластера с пометкой имени пода.

## Перед началом

- Вы прошли [Урок 21.9 — Метрики с Prometheus и Grafana](09-metrics-with-prometheus-and-grafana.md); этот урок расширяет его сервис `code/p21-orders`.
- Вы знаете `tracing`, спаны и `#[instrument]` из [Урока 09.10 — Логирование и трассировка](../09-testing-debugging-benchmarking/10-logging-and-tracing.md), а OpenTelemetry, Collector и Jaeger — из [Урока 16.12](../16-backend-services/12-logs-metrics-and-traces.md). Здесь мы переносим ту же схему в Kubernetes.
- `jq` установлен (в свежих версиях macOS он есть; проверьте командой `jq --version`).

## Новые слова

| Слово | Значение |
|---|---|
| **структурированный лог** (structured log) | Строка лога из именованных полей (здесь JSON, JavaScript Object Notation, нотация объектов JavaScript), а не свободного текста, чтобы программы могли её фильтровать и считать. |
| **ротация логов** (log rotation) | Закрыть разросшийся файл лога, начать новый и удалить самый старый. Для логов подов это делает kubelet. |
| **kubelet** | Агент Kubernetes на каждом узле. Он запускает контейнеры подов и отдаёт их логи для `kubectl logs`. |
| **агент узла** (node agent) | Программа, которая работает по одной копии на каждом узле и выполняет работу для всех подов там, например собирает их файлы логов. |
| **DaemonSet** | Объект Kubernetes, который запускает ровно одну копию пода на каждом узле. Обычный способ запустить агент узла. |
| **том hostPath** (hostPath volume) | Том, который монтирует в под папку самого узла. Мощно и рискованно; здесь используется, чтобы читать `/var/log/pods`. |
| **Downward API** (API — Application Programming Interface, программный интерфейс) | Способ передать контейнеру сведения о его собственном поде (имя, пространство имён, IP-адрес, IP — Internet Protocol, межсетевой протокол) через переменные окружения или файлы. |
| **атрибуты ресурса** (resource attributes) | В OpenTelemetry пары «ключ — значение», которые описывают, *кто* отправляет данные: `service.name`, `k8s.pod.name`, `k8s.namespace.name`. |
| **OTLP** (OpenTelemetry Protocol, протокол OpenTelemetry) | Протокол, который доставляет трассировки, метрики и логи в коллектор (Урок 16.12); здесь — поверх gRPC на порту 4317. |
| **gRPC** (gRPC Remote Procedure Calls, удалённые вызовы процедур gRPC) | Протокол удалённых вызовов из Урока 16.9, который здесь использует OTLP. |
| **CRI** (Container Runtime Interface, интерфейс среды выполнения контейнеров) | Интерфейс между kubelet и средой выполнения контейнеров. Он же задаёт формат файлов логов подов. |

## 1. Куда попадают логи подов

Контейнер в Kubernetes должен писать логи в **стандартный вывод** и **стандартный поток ошибок** и никуда больше. Это правило вы узнали с Docker в Уроке 11.12. В кластере это устроено так:

```text
 container ──stdout/stderr──► container runtime ──► /var/log/pods/<namespace>_<pod>_<uid>/<container>/0.log
                                                         ▲                         (on the node)
 kubectl logs ──► API server ──► kubelet on that node ───┘   reads the file
```

Среда выполнения контейнеров (в kind это containerd) записывает каждую строку в файл на **узле**. **kubelet** ротирует эти файлы, когда они разрастаются, и отдаёт их, когда вы запускаете `kubectl logs`. Узел kind сам является контейнером Docker, поэтому в него можно заглянуть. Сначала запустите кластер этого урока и сервис из раздела 3 или вернитесь сюда позже:

```bash
docker exec course-21-10-control-plane sh -c 'ls /var/log/pods | grep shop'
```

```text
shop_orders-6bcc6596f4-gfttn_9a4b593c-5288-44bf-9eaa-fa2b54221494
shop_orders-6bcc6596f4-zsk76_63d4f153-ffd8-42b6-9f06-d3502340b3c4
```

Одна папка на под: пространство имён, имя пода и уникальный идентификатор пода. Внутри — по файлу на контейнер. Каждая строка в формате CRI (Container Runtime Interface): время, поток, флаг (`F` = полная строка) и ваш текст:

```text
2026-11-18T08:24:13.694157632Z stdout F {"timestamp":"2026-11-18T08:24:13.694033Z","level":"INFO","message":"listening","addr":"0.0.0.0:3000","fail_every":20,"target":"p21_orders"}
```

Важны три следствия:

1. Когда под **удаляется**, удаляется и его папка. Для `kubectl logs` его логов больше нет.
2. Когда исчезает узел, исчезают все логи на нём.
3. Ротация хранит лишь несколько мегабайт на контейнер. Старые строки нагруженного сервиса пропадают за минуты.

Поэтому `kubectl logs` нужен, чтобы посмотреть *сейчас*. Чтобы хранить логи и искать в них потом, **агент узла** должен копировать файлы в надёжное место (раздел 4).

## 2. Читаем логи с `kubectl logs`

Команды, которые вы будете использовать чаще всего:

| Команда | Что показывает |
|---|---|
| `kubectl logs <pod>` | Все текущие строки одного пода (его единственного контейнера) |
| `kubectl logs <pod> -c <container>` | Один контейнер пода, в котором их несколько |
| `kubectl logs deployment/orders` | Один под Deployment, выбранный kubectl |
| `kubectl logs -l app=orders --prefix` | Все поды с этой меткой, перед каждой строкой — её под |
| `kubectl logs ... --tail=20` / `--since=5m` | Только последние 20 строк / последние 5 минут |
| `kubectl logs ... -f` | Продолжать показывать новые строки, как `tail -f` |
| `kubectl logs <pod> --previous` | Логи **предыдущего** контейнера, того, который упал |

`--previous` спасает после сбоя: Kubernetes перезапускает контейнер, и у нового — свежие пустые логи. Причина сбоя — в старом (в Уроке 21.5 так выяснилось, что перезапущенная программа была здорова, а ошибалась проба).

Когда подов несколько, всегда смотрите **все**. `kubectl logs deployment/orders` выбирает один под и сообщает об этом:

```bash
kubectl -n shop logs deployment/orders --tail=2
kubectl -n shop logs -l app=orders --prefix --tail=1
```

```text
Found 2 pods, using pod/orders-6bcc6596f4-gfttn
{"timestamp":"2026-11-18T08:24:19.517034Z","level":"INFO","message":"finished processing request","latency":"31 ms","status":404,"target":"tower_http::trace::on_response","span":{"method":"GET","uri":"/orders/500","version":"HTTP/1.1","name":"request"}}
{"timestamp":"2026-11-18T08:24:23.694455Z","level":"INFO","message":"finished processing request","latency":"0 ms","status":200,"target":"tower_http::trace::on_response","span":{"method":"GET","uri":"/healthz","version":"HTTP/1.1","name":"request"}}
[pod/orders-6bcc6596f4-gfttn/orders] {"timestamp":"2026-11-18T08:24:23.694455Z","level":"INFO","message":"finished processing request","latency":"0 ms","status":200,"target":"tower_http::trace::on_response","span":{"method":"GET","uri":"/healthz","version":"HTTP/1.1","name":"request"}}
[pod/orders-6bcc6596f4-zsk76/orders] {"timestamp":"2026-11-18T08:24:23.695964Z","level":"INFO","message":"finished processing request","latency":"0 ms","status":200,"target":"tower_http::trace::on_response","span":{"method":"GET","uri":"/healthz","version":"HTTP/1.1","name":"request"}}
```

Эти строки уже **структурированы**: это объекты JSON. Следующий раздел показывает, как сервис их пишет.

## 3. Структурированные логи и трассировки из сервиса

В Уроке 16.12 логи были текстовыми строками для людей. В кластере логи читают в основном *программы* (коллекторы, поисковые системы), поэтому каждая строка должна быть одним объектом JSON с именованными полями. Крейт `tracing-subscriber` умеет это с фичей `json`. Ещё мы отправляем спаны как трассировки, как в Уроке 16.12, но только трассировки: метрики уже идут в Prometheus.

Из `code/p21-orders` добавьте крейты:

```bash
cargo add opentelemetry@0.33 opentelemetry_sdk@0.33
cargo add opentelemetry-otlp@0.33 --no-default-features --features grpc-tonic,trace
cargo add tracing@0.1 tracing-opentelemetry@0.34
cargo add tracing-subscriber@0.3 --features env-filter,json
cargo add tower-http@0.7 --features trace
```

`src/telemetry.rs` — половина модуля из Урока 16.12, отвечающая за трассировки, с выводом в JSON:

```rust
use opentelemetry::trace::TracerProvider as _;
use opentelemetry_sdk::Resource;
use opentelemetry_sdk::trace::SdkTracerProvider;
use tracing_subscriber::EnvFilter;
use tracing_subscriber::layer::SubscriberExt;
use tracing_subscriber::util::SubscriberInitExt;

/// Sets up JSON logs on standard output and trace export over OTLP.
pub fn init(service_name: &'static str) -> Result<SdkTracerProvider, Box<dyn std::error::Error>> {
    let resource = Resource::builder().with_service_name(service_name).build();
    let exporter = opentelemetry_otlp::SpanExporter::builder()
        .with_tonic()
        .build()?;
    let provider = SdkTracerProvider::builder()
        .with_resource(resource)
        .with_batch_exporter(exporter)
        .build();

    tracing_subscriber::registry()
        .with(EnvFilter::try_from_default_env().unwrap_or_else(|_| "info".into()))
        .with(
            tracing_subscriber::fmt::layer()
                .json()
                .flatten_event(true)
                .with_span_list(false),
        )
        .with(tracing_opentelemetry::layer().with_tracer(provider.tracer(service_name)))
        .init();
    Ok(provider)
}
```

**Что происходит:**

1. `.json()` превращает каждое событие в одну строку JSON. `flatten_event(true)` выносит поля события (`message`, `order_id`) на верхний уровень вместо вложенного объекта `fields`, так проще фильтровать. `with_span_list(false)` убирает список всех родительских спанов и оставляет в `span` только текущий, чтобы строки были короче.
2. `Resource::builder()` помимо имени сервиса читает переменную окружения `OTEL_RESOURCE_ATTRIBUTES` (если она задана). С её помощью мы добавим к каждому спану имя пода.
3. Экспортёр берёт адрес из `OTEL_EXPORTER_OTLP_ENDPOINT`, как в Уроке 16.12. В кластере её задаёт Deployment.

В `src/main.rs` добавьте в начало `mod telemetry;` и эти импорты:

```rust
use tower_http::trace::{DefaultMakeSpan, DefaultOnResponse, TraceLayer};
use tracing::{Level, info, instrument, warn};
```

Затем измените `main`, `app` и `get_order` и добавьте `load_order`. `AppState`, `shutdown_signal`, `track_http` и `render_metrics` остаются точно такими же, как в Уроке 21.9.

```rust
#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let tracer_provider = telemetry::init("p21-orders")?;
    let handle = PrometheusBuilder::new()
        .set_buckets_for_metric(
            Matcher::Full("http_request_duration_seconds".to_string()),
            &[0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0],
        )?
        .install_recorder()?;
    let upkeep = handle.clone();
    tokio::spawn(async move {
        loop {
            tokio::time::sleep(Duration::from_secs(5)).await;
            upkeep.run_upkeep(); // drains histogram data so memory does not grow
        }
    });

    let fail_every = std::env::var("FAIL_EVERY")
        .ok()
        .and_then(|v| v.parse().ok())
        .unwrap_or(0);
    let state = AppState {
        metrics: handle,
        fail_every,
        counter: Arc::new(AtomicU64::new(0)),
    };

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await?;
    info!(addr = %listener.local_addr()?, fail_every, "listening");
    axum::serve(listener, app(state))
        .with_graceful_shutdown(shutdown_signal())
        .await?;

    info!("shutting down, sending the last spans");
    if let Err(err) = tracer_provider.shutdown() {
        eprintln!("trace shutdown failed: {err}");
    }
    Ok(())
}

fn app(state: AppState) -> Router {
    Router::new()
        .route("/orders/{id}", get(get_order))
        .route("/healthz", get(|| async { "ok\n" }))
        .route("/metrics", get(render_metrics))
        .route_layer(middleware::from_fn(track_http))
        .layer(
            TraceLayer::new_for_http()
                .make_span_with(DefaultMakeSpan::new().level(Level::INFO))
                .on_response(DefaultOnResponse::new().level(Level::INFO)),
        )
        .with_state(state)
}

#[instrument(skip(state))]
async fn get_order(
    State(state): State<AppState>,
    Path(id): Path<u32>,
) -> Result<String, StatusCode> {
    let n = state.counter.fetch_add(1, Ordering::Relaxed) + 1;
    load_order(id).await;
    if state.fail_every > 0 && n % state.fail_every == 0 {
        warn!(
            order_id = id,
            request_number = n,
            "simulated database failure"
        );
        return Err(StatusCode::INTERNAL_SERVER_ERROR);
    }
    if id > 100 {
        warn!(order_id = id, "order not found");
        return Err(StatusCode::NOT_FOUND);
    }
    info!(order_id = id, "order sent");
    Ok(format!("order {id}: 2 books\n"))
}

#[instrument]
async fn load_order(id: u32) {
    tokio::time::sleep(Duration::from_millis(30)).await; // pretend database query
}
```

**Что происходит:**

1. `info!(addr = %..., fail_every, "listening")` заменяет `println!`. Теперь каждое сообщение — структурированное событие; `%` записывает адрес в его форме `Display`.
2. `TraceLayer` открывает спан `request` для каждого запроса HTTP (HyperText Transfer Protocol) и пишет «finished processing request» со статусом и задержкой (Урок 16.12).
3. `#[instrument(skip(state))]` создаёт спан `get_order` с номером заказа в поле, но пропускает состояние: оно большое и неинтересное. `load_order` получает собственный дочерний спан, поэтому трассировка показывает время «базы данных» отдельно.
4. События `warn!` и `info!` несут `order_id` как поле, поэтому потом можно искать один конкретный заказ.
5. После `SIGTERM` функция `main` вызывает `tracer_provider.shutdown()`, и та отправляет спаны, которые ещё ждут в пакете. Это ещё одна причина обрабатывать `SIGTERM`: убитый под теряет свои последние трассировки.

Укажите `version = "0.2.0"` в `Cargo.toml` и выполните `cargo test` (он заодно обновит `Cargo.lock`, Урок 21.4). Затем соберите образ `course-p21-orders:0.2.0` с Dockerfile из Урока 21.9, создайте кластер и загрузите образ:

```bash
docker build -t course-p21-orders:0.2.0 .
kind create cluster --name course-21-10
kind load docker-image course-p21-orders:0.2.0 --name course-21-10
```

Deployment в `k8s/orders.yaml` — тот же, что в Уроке 21.9, с новым образом и дополнительными переменными окружения (аннотации Prometheus убраны для краткости; вреда от них нет):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: shop
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: orders
  namespace: shop
spec:
  replicas: 2
  selector:
    matchLabels:
      app: orders
  template:
    metadata:
      labels:
        app: orders
    spec:
      containers:
        - name: orders
          image: course-p21-orders:0.2.0
          env:
            - name: FAIL_EVERY
              value: "20"
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: http://otel-collector.observability.svc:4317
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: OTEL_RESOURCE_ATTRIBUTES
              value: k8s.pod.name=$(POD_NAME),k8s.namespace.name=$(POD_NAMESPACE)
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /healthz
              port: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: orders
  namespace: shop
spec:
  selector:
    app: orders
  ports:
    - port: 80
      targetPort: 3000
```

**Что происходит:**

1. `OTEL_EXPORTER_OTLP_ENDPOINT` указывает на Service коллектора в пространстве имён `observability`, которое мы создадим в следующем разделе.
2. `valueFrom.fieldRef` — это **Downward API**: Kubernetes записывает в `POD_NAME` собственное имя пода, а в `POD_NAMESPACE` — его пространство имён.
3. `$(POD_NAME)` внутри другого значения Kubernetes заменяет этой переменной. Поэтому каждый под отправляет спаны со своим `k8s.pod.name`, и в Jaeger видно, *какой* под обработал запрос.

Применяйте его только после того, как появится коллектор (раздел 4), иначе первые спаны не удастся отправить (сервис при этом продолжит работать).

## 4. Коллектор на каждом узле

Коллектор из Урока 16.12 принимал трассировки. В кластере он может быть ещё и **агентом узла** для логов: читать файлы в `/var/log/pods` и отправлять каждую строку в хранилище логов. Коллектор должен работать на **каждом** узле, потому что на каждом узле есть только свои файлы. Это работа для **DaemonSet**: Kubernetes запускает ровно одну копию его пода на каждом узле, а на новых узлах — автоматически.

`k8s/observability.yaml` создаёт пространство имён и Jaeger (тот же образ, что в Уроке 16.12):

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: observability
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
  namespace: observability
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jaeger
  template:
    metadata:
      labels:
        app: jaeger
    spec:
      containers:
        - name: jaeger
          image: jaegertracing/jaeger:2.23.0
          ports:
            - containerPort: 16686
            - containerPort: 4317
---
apiVersion: v1
kind: Service
metadata:
  name: jaeger
  namespace: observability
spec:
  selector:
    app: jaeger
  ports:
    - name: ui
      port: 16686
    - name: otlp
      port: 4317
```

`k8s/collector.yaml` содержит конфигурацию коллектора, DaemonSet и Service:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: otel-collector
  namespace: observability
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
      file_log:
        include: [/var/log/pods/shop_*/*/*.log]
        include_file_path: true
        start_at: end
        operators:
          - type: container
    processors:
      batch: {}
    exporters:
      debug:
        verbosity: normal
      otlp_grpc/jaeger:
        endpoint: jaeger.observability.svc:4317
        tls:
          insecure: true
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch]
          exporters: [otlp_grpc/jaeger]
        logs:
          receivers: [file_log]
          processors: [batch]
          exporters: [debug]
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: otel-collector
  namespace: observability
spec:
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      containers:
        - name: collector
          image: otel/opentelemetry-collector-contrib:0.165.0
          args: ["--config=/etc/otelcol/config.yaml"]
          securityContext:
            runAsUser: 0
          ports:
            - containerPort: 4317
          volumeMounts:
            - name: config
              mountPath: /etc/otelcol
            - name: pod-logs
              mountPath: /var/log/pods
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: otel-collector
        - name: pod-logs
          hostPath:
            path: /var/log/pods
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: observability
spec:
  selector:
    app: otel-collector
  ports:
    - port: 4317
```

**Что происходит:**

1. Приёмник `file_log` читает файлы логов всех подов пространства имён `shop` (`include` — шаблон файлов). `start_at: end` читает только новые строки, а не старые. `include_file_path: true` сохраняет путь к файлу с каждой строкой.
2. Оператор `container` понимает формат строк CRI: убирает время и префикс потока, а пространство имён, имя пода и имя контейнера берёт **из пути к файлу** и сохраняет как атрибуты ресурса.
3. Конвейер `logs` отправляет строки в экспортёр `debug`, который печатает их в собственный лог коллектора. В настоящем кластере вы бы экспортировали их в хранилище логов (например, Grafana Loki или Elasticsearch через их приёмники OTLP); изменилась бы только принимающая сторона.
4. Конвейер `traces` — тот же, что в Уроке 16.12: на входе OTLP, на выходе Jaeger.
5. **Том hostPath** монтирует папку узла `/var/log/pods` в под только для чтения. Файлы принадлежат `root` и читаются только им (`-rw-r-----`), поэтому коллектор работает от пользователя 0.

> 🔒 **Безопасность:** под с томом hostPath и `runAsUser: 0` может читать файлы всех подов на узле, включая их логи. Давайте такую власть только доверенному агенту, монтируйте только для чтения, только нужную папку и держите агент в отдельном пространстве имён. Это ещё одна причина никогда не писать в логи пароли и токены: их прочитает каждый, у кого есть доступ к конвейеру логов.

Примените всё и отправьте немного трафика:

```bash
kubectl apply -f k8s/observability.yaml -f k8s/collector.yaml
kubectl -n observability rollout status daemonset/otel-collector
kubectl apply -f k8s/orders.yaml
kubectl -n shop rollout status deployment/orders
kubectl -n shop run load --rm -i --restart=Never --image=busybox:1.37 -- \
  sh -c 'for i in $(seq 1 60); do wget -q -O /dev/null http://orders/orders/$i 2>/dev/null; done; true'
kubectl -n observability logs daemonset/otel-collector | grep -E '^ResourceLog|order sent' | head -3
```

Сокращённый вывод (за каждой строкой лога следуют ещё атрибуты, мы их обрезали; строка `ResourceLog` начинает строки одного пода и перечисляет его атрибуты ресурса):

```text
{"timestamp":"2026-11-18T08:24:49.013154Z","level":"INFO","message":"order sent","order_id":1,"target":"p21_orders","span":{"id":"1","name":"get_order"}} logtag=F log.file.path=/var/log/pods/shop_orders-6bcc6596f4-gfttn_9a4b593c-5288-44bf-9eaa-fa2b54221494/ord...
{"timestamp":"2026-11-18T08:24:49.046520Z","level":"INFO","message":"order sent","order_id":2,"target":"p21_orders","span":{"id":"2","name":"get_order"}} logtag=F log.file.path=/var/log/pods/shop_orders-6bcc6596f4-zsk76_63d4f153-ffd8-42b6-9f06-d3502340b3c4/ord...
ResourceLog #1 k8s.container.name=orders k8s.container.restart_count=0 k8s.namespace.name=shop k8s.pod.name=orders-6bcc6596f4-gfttn k8s.pod.uid=9a4b593c-5288-44bf-9eaa-fa2b54221494
```

Коллектор прочитал строки с узла и знает, из какого они пода. Теперь удалите под и сравните:

```bash
kubectl -n shop delete pod orders-6bcc6596f4-gfttn
kubectl -n shop logs orders-6bcc6596f4-gfttn
```

```text
pod "orders-6bcc6596f4-gfttn" deleted from shop namespace
error: error from server (NotFound): pods "orders-6bcc6596f4-gfttn" not found in namespace "shop"
```

У `kubectl logs` больше ничего нет, а коллектор уже скопировал строки. В этом и весь смысл сбора логов.

> 💡 **Совет:** в большом кластере сервис отправляет трассировки коллектору на **своём** узле (через IP-адрес узла и `hostPort`), а не через Service, который может выбрать коллектор на другом узле. На одном узле kind разницы нет, поэтому мы оставляем простой Service.

## 5. Смотрим трассировки в Jaeger

Откройте Jaeger через port-forward:

```bash
kubectl -n observability port-forward service/jaeger 16686:16686
```

На <http://localhost:16686> выберите сервис `p21-orders`, операцию `get_order` и нажмите **Find Traces**. Откройте одну трассировку. В каждой три спана, `request` → `get_order` → `load_order`, а на странице трассировки раздел **Process** перечисляет атрибуты ресурса, среди них `k8s.pod.name` и `k8s.namespace.name`.

У Jaeger есть и HTTP API. Краткая сводка двух трассировок заказов (идентификатор трассировки, спан, миллисекунды, под) выглядит так:

```text
0c1bc130	get_order	31	orders-6bcc6596f4-gfttn
0c1bc130	load_order	31	orders-6bcc6596f4-gfttn
0c1bc130	request	31	orders-6bcc6596f4-gfttn
13adf1a3	get_order	31	orders-6bcc6596f4-zsk76
13adf1a3	load_order	31	orders-6bcc6596f4-zsk76
13adf1a3	request	31	orders-6bcc6596f4-zsk76
```

Две трассировки, два разных пода: Service распределил запросы, и каждый спан знает, где он выполнялся. Запросы, вернувшие 500, помечены как ошибки: `TraceLayer` пишет «response failed» на уровне `ERROR`, а `tracing-opentelemetry` затем ставит спану статус *error*. В форме поиска Jaeger напишите в **Tags** `error=true`, чтобы найти только неудачные запросы.

Теперь три сигнала указывают друг на друга. Метрика в Grafana показывает рост доли ошибок (Урок 21.9); Jaeger показывает неудачные трассировки, а также в каком поде и каком спане они упали; логи этого пода, отфильтрованные по `order_id`, говорят, что именно произошло.

> ⚡ **Производительность:** каждый запрос становится трассировкой, включая проверки здоровья каждые 10 секунд от каждого пода. В нагруженном сервисе отправляют только *выборку* трассировок (например, 10%), заданную через `OTEL_TRACES_SAMPLER=traceidratio` и `OTEL_TRACES_SAMPLER_ARG=0.1`, а пробы не трассируют совсем (Упражнение 3).

## Частые ошибки

> ⚠️ **Частая ошибка:** `--previous` для контейнера, который ни разу не падал:

```text
Error from server (BadRequest): previous terminated container "orders" in pod "orders-6bcc6596f4-gfttn" not found
```

Предыдущего контейнера ещё нет; читайте текущие логи. После сбоя `--previous` — первое, что нужно запустить.

> ⚠️ **Частая ошибка:** использовать имя Deployment как имя пода: `kubectl -n shop logs orders` падает с `pods "orders" not found`. Пишите `deployment/orders` или используйте `-l app=orders --prefix`, чтобы видеть все поды.

> ⚠️ **Частая ошибка:** забыть `include_file_path: true` в приёмнике `file_log`. Оператору `container` нужен путь, чтобы узнать под, и каждая строка падает с ошибкой:

```text
error	helper/transformer.go:160	Failed to process entry	{... "error": "operator 'container' has 'add_metadata_from_filepath' enabled, but the log record attribute 'log.file.path' is missing. Perhaps enable the 'include_file_path' option?" ...}
```

Раньше приёмник назывался `filelog`; коллектор 0.165.0 его ещё принимает, но предупреждает: `"filelog" alias is deprecated; use "file_log" instead`.

> ⚠️ **Частая ошибка:** писать логи в файл внутри контейнера. Kubernetes их не видит, они исчезают вместе с контейнером и могут заполнить его диск. Пишите только в стандартный вывод.

## Попробуйте сами

1. Следите за логами всех подов, пока отправляете запросы: `kubectl -n shop logs -l app=orders --prefix -f`, а во втором терминале запустите команду нагрузки из раздела 4. Остановите клавишами Ctrl+C.
2. Посчитайте строки логов за последние пять минут: `kubectl -n shop logs -l app=orders --since=5m --tail=-1 | wc -l`. (`--tail=-1` означает «без ограничения»; с `-l` kubectl по умолчанию показывает только 10 строк на под.)
3. После удаления пода выше снова посмотрите на папку логов подов на узле: папки удалённого пода нет, а у нового пода новая папка.
4. Когда закончите: `kind delete cluster --name course-21-10`.

## Упражнения

### Упражнение 1 — Найдите сбои (легко)

Только с помощью `kubectl` и `jq` выведите время и URI (Uniform Resource Identifier, здесь путь запроса) каждого запроса, завершившегося статусом 500, со **всех** подов заказов, в компактном JSON. Одна строка на неудачный запрос.

<details>
<summary>Подсказка</summary>

У строк «finished processing request» есть числовое поле `status`, а URI лежит в `.span.uri`. `jq -c 'select(...) | {...}'` фильтрует и меняет форму.

</details>

<details>
<summary>Ответ</summary>

```bash
kubectl -n shop logs -l app=orders --tail=-1 | jq -c 'select(.status == 500) | {timestamp, uri: .span.uri}'
```

```text
{"timestamp":"2026-11-18T08:25:54.613294Z","uri":"/orders/8"}
{"timestamp":"2026-11-18T08:25:54.776897Z","uri":"/orders/13"}
{"timestamp":"2026-11-18T08:25:55.771095Z","uri":"/orders/43"}
```

Почему это работает: `-l app=orders --tail=-1` читает все строки всех подов; раз каждая строка — JSON, `jq` проверяет поле, а не ищет текст. У строк без `status` получается `null`, а это не `500`, поэтому они пропускаются.

</details>

### Упражнение 2 — Тест строки лога в JSON (средне)

**Напишите тест**, который отправляет `GET /orders/500` в маршрутизатор и проверяет, что в логах есть строка JSON с `"message":"order not found"`, `"level":"WARN"` и `"order_id":500`. Тест не должен печатать в терминал и не должен менять глобальный подписчик.

<details>
<summary>Подсказка</summary>

Соберите подписчика через `tracing_subscriber::fmt().json().flatten_event(true).with_writer(...).finish()` и включите его для текущего потока через `tracing::subscriber::set_default(subscriber)` (сохраните возвращённый guard-объект). Писатель должен реализовывать `MakeWriter`; хватит маленького типа вокруг `Arc<Mutex<Vec<u8>>>`, который реализует `std::io::Write` и `MakeWriter`. Используйте тестовый рантайм `current_thread`, как в Уроке 21.9.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use std::sync::Mutex;

    use axum::body::Body;
    use tower::ServiceExt;
    use tracing_subscriber::fmt::MakeWriter;

    /// A writer that keeps everything in memory, so the test can read the logs.
    #[derive(Clone, Default)]
    struct Buffer(Arc<Mutex<Vec<u8>>>);

    impl std::io::Write for Buffer {
        fn write(&mut self, bytes: &[u8]) -> std::io::Result<usize> {
            self.0.lock().unwrap().extend_from_slice(bytes);
            Ok(bytes.len())
        }
        fn flush(&mut self) -> std::io::Result<()> {
            Ok(())
        }
    }

    impl<'a> MakeWriter<'a> for Buffer {
        type Writer = Buffer;
        fn make_writer(&'a self) -> Buffer {
            self.clone()
        }
    }

    #[tokio::test(flavor = "current_thread")]
    async fn unknown_order_is_logged_as_json_warning() {
        let buffer = Buffer::default();
        let subscriber = tracing_subscriber::fmt()
            .json()
            .flatten_event(true)
            .with_writer(buffer.clone())
            .finish();
        let _guard = tracing::subscriber::set_default(subscriber);

        let state = AppState {
            metrics: PrometheusBuilder::new().build_recorder().handle(),
            fail_every: 0,
            counter: Arc::new(AtomicU64::new(0)),
        };
        let request = axum::http::Request::get("/orders/500")
            .body(Body::empty())
            .unwrap();
        let response = app(state).oneshot(request).await.unwrap();
        assert_eq!(response.status(), StatusCode::NOT_FOUND);

        let logs = String::from_utf8(buffer.0.lock().unwrap().clone()).unwrap();
        let line = logs
            .lines()
            .find(|line| line.contains(r#""message":"order not found""#))
            .expect("no 'order not found' line");
        assert!(line.contains(r#""level":"WARN""#), "line was: {line}");
        assert!(line.contains(r#""order_id":500"#), "line was: {line}");
    }
}
```

```text
running 1 test
test tests::unknown_order_is_logged_as_json_warning ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.23s
```

Почему это работает: `set_default` включает подписчика только в этом потоке, пока жив guard-объект, а однопоточный рантайм выполняет там весь маршрутизатор. Писатель сохраняет байты, поэтому тест читает настоящие строки JSON. Вызовы `metrics` никуда не идут, потому что рекордер не установлен, и это безвредно. (Оставьте dev-зависимости `tower` и `http-body-util` из Урока 21.9.)

</details>

### Упражнение 3 — Не трассируйте проверки здоровья (сложно)

Каждый запрос пробы даёт строку лога и трассировку. Измените `fn app` так, чтобы `/healthz` по-прежнему отвечал `ok`, но **не** давал строки «finished processing request» и трассировки. Проверьте локально через `cargo run`, один `curl localhost:3000/healthz` и один `curl localhost:3000/orders/1`. Затем скажите, что ещё изменилось для `/healthz`.

<details>
<summary>Подсказка</summary>

В axum `.layer(...)` и `.route_layer(...)` оборачивают только маршруты, добавленные **до** них.

</details>

<details>
<summary>Ответ</summary>

```rust
fn app(state: AppState) -> Router {
    Router::new()
        .route("/orders/{id}", get(get_order))
        .route("/metrics", get(render_metrics))
        .route_layer(middleware::from_fn(track_http))
        .layer(
            TraceLayer::new_for_http()
                .make_span_with(DefaultMakeSpan::new().level(Level::INFO))
                .on_response(DefaultOnResponse::new().level(Level::INFO)),
        )
        .route("/healthz", get(|| async { "ok\n" })) // added after the layers: not traced
        .with_state(state)
}
```

Сокращённый лог запуска (строки обрезаны до 120 символов):

```text
{"timestamp":"2026-11-18T08:27:37.927158Z","level":"INFO","message":"listening","addr":"0.0.0.0:3000","fail_every":0,"ta
{"timestamp":"2026-11-18T08:27:39.306927Z","level":"INFO","message":"order sent","order_id":1,"target":"p21_orders","spa
{"timestamp":"2026-11-18T08:27:39.307080Z","level":"INFO","message":"finished processing request","latency":"32 ms","sta
```

Почему это работает: маршрут `/healthz` добавлен после обоих слоёв, поэтому его не оборачивают ни `TraceLayer`, ни `track_http`. Строку «finished processing request» дал только запрос заказа. Что ещё изменилось: `/healthz` больше не считается и в `http_requests_total`, и обычно именно это и нужно, ведь пробы — не настоящие пользователи.

</details>

## Вопросы для самопроверки

1. Откуда `kubectl logs` берёт данные и почему эти логи теряются при удалении пода?

<details>
<summary>Ответ</summary>

От kubelet узла, на котором работает под; kubelet читает файлы логов в `/var/log/pods/<namespace>_<pod>_<uid>/` на этом узле. Папка удаляется вместе с подом, а ротация удаляет старые строки ещё раньше.

</details>

2. Ваш под перезапустился три раза. Какая команда покажет, почему он упал в последний раз?

<details>
<summary>Ответ</summary>

`kubectl logs <pod> --previous`. У текущего контейнера новые пустые логи; ошибка — в предыдущем.

</details>

3. Почему сборщик логов разворачивают как DaemonSet, а не как Deployment?

<details>
<summary>Ответ</summary>

На каждом узле есть только файлы логов его собственных подов. DaemonSet запускает ровно один коллектор на каждом узле, включая новые, поэтому прочитан будет каждый файл.

</details>

4. Что Downward API даёт контейнеру и как мы его использовали?

<details>
<summary>Ответ</summary>

Сведения о его собственном поде, например имя и пространство имён. Мы положили их в `OTEL_RESOURCE_ATTRIBUTES`, поэтому каждый спан несёт `k8s.pod.name` и `k8s.namespace.name`.

</details>

5. Почему в кластере логи в JSON лучше текстовых строк?

<details>
<summary>Ответ</summary>

Программы (коллекторы, хранилища логов, `jq`) могут фильтровать и считать по полям, например `status == 500` или `order_id == 7`, без хрупкого поиска по тексту.

</details>

## Итоги

- Контейнеры пишут логи в стандартный вывод; среда выполнения записывает строки в файлы `/var/log/pods` на узле, а kubelet ротирует их и отдаёт `kubectl logs`.
- Используйте `-l app=... --prefix`, чтобы видеть все поды, `--since` и `--tail`, чтобы ограничить вывод, `-f`, чтобы следить, и `--previous` после сбоя.
- Пишите структурированные логи в JSON (`tracing-subscriber` с `json`), с идентификаторами в полях и никогда — с секретами.
- Агент узла, здесь OpenTelemetry Collector в виде DaemonSet с томом hostPath только для чтения, сохраняет логи и после исчезновения подов.
- Трассировки идут из сервиса по OTLP в коллектор и дальше в Jaeger; Downward API добавляет к каждому спану имя пода.
- Метрики, трассировки и логи вместе отвечают, *что* сломалось, *где* и *как именно*.

## Источники и что почитать

- [Kubernetes — Logging architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/) — куда попадают логи контейнеров, ротация и агенты узлов.
- [Справочник kubectl logs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/) — все параметры `kubectl logs`.
- [Kubernetes — DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) и [Downward API](https://kubernetes.io/docs/concepts/workloads/pods/downward-api/) — две возможности Kubernetes, которые используют коллектор и сервис.
- [OpenTelemetry — Important components for Kubernetes](https://opentelemetry.io/docs/platforms/kubernetes/collector/components/) — приёмники и процессоры коллектора для кластеров.
- [Приёмник filelog](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver) — параметры приёмника файлов логов и оператора `container`.
- [OpenTelemetry — SDK configuration](https://opentelemetry.io/docs/languages/sdk-configuration/general/) — `OTEL_RESOURCE_ATTRIBUTES`, `OTEL_TRACES_SAMPLER` и другие переменные.
- [tracing-subscriber — формат JSON](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/format/struct.Json.html) — параметры, использованные для логов в JSON.

---

[🏠 Начало](../START-HERE.md) · [← 21.9 Метрики с Prometheus и Grafana](09-metrics-with-prometheus-and-grafana.md) · [📋 Этап 21](README.md) · [21.11 Основы SRE: SLI, SLO и бюджет ошибок →](11-sre-basics-slis-slos-and-error-budgets.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
