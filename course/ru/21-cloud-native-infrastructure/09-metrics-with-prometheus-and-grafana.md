<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 21.8 Инфраструктура как код с OpenTofu](08-infrastructure-as-code-with-opentofu.md) · [📋 Этап 21](README.md) · [21.10 Логи и трассировки в кластере →](10-logs-and-traces-in-a-cluster.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) <!-- nav -->

# Урок 21.9 — Метрики с Prometheus и Grafana

> **Этап 21 — Облачная (cloud-native) инфраструктура** · Урок 9 из 14 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как Prometheus собирает метрики: он сам *забирает* их со страницы `/metrics` каждого пода.
- Как сервис на Rust публикует счётчики, датчики (gauge) и гистограммы с крейтом `metrics`.
- Как установить Prometheus в кластер kind через Helm так, чтобы он сам находил ваши поды.
- Как задавать вопросы на PromQL: частота запросов, доля ошибок и 95-й перцентиль задержки.
- Как показать ответы в Grafana и почему у меток должно быть мало значений.

## Перед началом

- Вы прошли [Урок 21.8 — Инфраструктура как код с OpenTofu](08-infrastructure-as-code-with-opentofu.md).
- Вы знаете, что такое метрики, счётчики и гистограммы, из [Урока 16.12 — Логи, метрики и трассировки с OpenTelemetry](../16-backend-services/12-logs-metrics-and-traces.md). Там сервис сам *отправлял* метрики в коллектор; здесь Prometheus *забирает* их.
- Вы умеете разворачивать образ в kind ([Урок 21.4](04-deploying-a-rust-service-to-kubernetes.md)), пользоваться пробами ([Урок 21.5](05-configmaps-secrets-and-probes.md)) и устанавливать Helm-чарт ([Урок 21.6](06-helm-and-kustomize.md)).
- Код лежит в `~/projects/rust/code/p21-orders`. Уроки 21.10 и 21.11 продолжают работу с этим же сервисом.

## Новые слова

| Слово | Значение |
|---|---|
| **Prometheus** | Система мониторинга с открытым исходным кодом. Она регулярно собирает числа из программ, хранит их, отвечает на запросы и может поднимать тревоги (alerts). |
| **сбор** (scrape) | Одно обращение Prometheus к странице `/metrics` программы, чтобы прочитать текущие значения. Происходит раз в *интервал сбора* (здесь 15 секунд). |
| **цель** (target) | Один адрес, который опрашивает Prometheus, например один под. |
| **временной ряд** (time series) | Имя метрики плюс один точный набор меток со значениями во времени: `http_requests_total{status="500", pod="orders-1"}`. |
| **метка** (label) | Пара «ключ — значение», которая делит одну метрику на несколько временных рядов, например `status="404"`. |
| **датчик** (gauge) | Метрика, которая может расти и уменьшаться: запросы в обработке, занятая память, длина очереди. |
| **PromQL** (Prometheus Query Language, язык запросов Prometheus) | Язык, на котором задают вопросы Prometheus, например «запросов в секунду по статусам за последнюю минуту». |
| **перцентиль** (percentile) | Значение, ниже которого лежит заданная доля измерений. 95-й перцентиль (p95) задержки — время, в которое укладываются 95% запросов. |
| **кардинальность** (cardinality) | Сколько разных временных рядов у метрики. Каждое новое значение метки добавляет ряды; слишком много рядов перегружает Prometheus. |
| **Grafana** | Веб-инструмент с открытым исходным кодом, который рисует графики и дашборды по источникам данных, таким как Prometheus. |
| **источник данных** (data source) | Подключение в Grafana к базе, из которой она читает данные (здесь — к Prometheus). |
| **дашборд / панель** (dashboard / panel) | Страница Grafana / один график или число на этой странице, за которым стоит один запрос. |
| **HTTP** (HyperText Transfer Protocol, протокол передачи гипертекста) / **JSON** (JavaScript Object Notation, нотация объектов JavaScript) / **URL** (Uniform Resource Locator, единый указатель ресурса) | Веб-протокол / текстовый формат данных / веб-адрес. Все три встречаются в API Prometheus и Grafana. |
| **UI** (User Interface, пользовательский интерфейс) | Часть программы, на которую смотрят и по которой щёлкают люди; здесь — веб-страницы Prometheus и Grafana. |

## 1. Как работает Prometheus

В Уроке 16.12 сервис сам отправлял (*push*) свои метрики в OpenTelemetry Collector. **Prometheus** обычно работает наоборот: каждая программа показывает свои текущие числа на простой текстовой странице `/metrics`, а Prometheus каждые несколько секунд приходит и читает (*pull*) эту страницу. Одно такое посещение называется **сбором** (scrape).

```text
                    Kubernetes API: "which pods exist?"
                               │
                               ▼
  ┌──────────────┐  scrape  ┌──────────────────────┐  PromQL  ┌──────────┐
  │ pod orders-1 │◄─────────│      Prometheus      │◄─────────│ Grafana  │
  │   /metrics   │  every   │ stores time series   │  query   │ graphs   │
  ├──────────────┤   15 s   │ answers queries      │          └──────────┘
  │ pod orders-2 │◄─────────│                      │
  │   /metrics   │          └──────────────────────┘
  └──────────────┘
```

У модели «забирать» два преимущества. Prometheus знает, когда цель *не работает* (сбор не удаётся), а программе не нужно знать, где находится Prometheus. В Kubernetes Prometheus спрашивает Kubernetes API (Application Programming Interface, программный интерфейс), какие поды существуют, поэтому новые поды находятся автоматически.

При каждом сборе сохраняется одно значение для каждого **временного ряда**. Временной ряд — это имя метрики плюс точный набор **меток**. `http_requests_total{status="200"}` и `http_requests_total{status="500"}` — два разных ряда одной метрики.

## 2. Метрики из сервиса на Rust

Наш сервис — небольшой сервис заказов из Урока 16.12, теперь с тремя эндпоинтами: `GET /orders/{id}`, `GET /healthz` для проб и `GET /metrics` для Prometheus. Чтобы потренироваться с ошибками, переменная окружения `FAIL_EVERY=20` заставляет каждый 20-й запрос заказа завершаться статусом 500.

```bash
cd ~/projects/rust/code
cargo new p21-orders
cd p21-orders
cargo add axum@0.8 metrics@0.24
cargo add metrics-exporter-prometheus@0.18 --no-default-features
cargo add tokio@1 --features macros,rt-multi-thread,signal,time
```

Крейт **`metrics`** — это *фасад*: ваш код вызывает `counter!`, `gauge!` и `histogram!`, а отдельно установленный *рекордер* (recorder) решает, что делать со значениями. `metrics-exporter-prometheus` — рекордер, который хранит значения в памяти и выводит их в текстовом формате Prometheus. Мы отключаем его фичи по умолчанию, потому что они приносят собственный сервер HTTP (HyperText Transfer Protocol, протокол передачи гипертекста), а axum у нас уже есть.

`src/main.rs`:

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicU64, Ordering};
use std::time::{Duration, Instant};

use axum::Router;
use axum::extract::{MatchedPath, Path, Request, State};
use axum::http::StatusCode;
use axum::middleware::{self, Next};
use axum::response::Response;
use axum::routing::get;
use metrics_exporter_prometheus::{Matcher, PrometheusBuilder, PrometheusHandle};
use tokio::signal::unix::{SignalKind, signal};

#[derive(Clone)]
struct AppState {
    metrics: PrometheusHandle,
    fail_every: u64,         // 0 = never fail; 5 = every 5th order request fails
    counter: Arc<AtomicU64>, // how many order requests we have seen
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
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
    println!(
        "listening on {} (FAIL_EVERY={fail_every})",
        listener.local_addr()?
    );
    axum::serve(listener, app(state))
        .with_graceful_shutdown(shutdown_signal())
        .await?;
    Ok(())
}

/// Finishes on SIGTERM (sent by Kubernetes when it stops a pod) or Ctrl+C.
async fn shutdown_signal() {
    let mut term = signal(SignalKind::terminate()).expect("cannot listen for SIGTERM");
    tokio::select! {
        _ = term.recv() => {}
        _ = tokio::signal::ctrl_c() => {}
    }
}

fn app(state: AppState) -> Router {
    Router::new()
        .route("/orders/{id}", get(get_order))
        .route("/healthz", get(|| async { "ok\n" }))
        .route("/metrics", get(render_metrics))
        .route_layer(middleware::from_fn(track_http))
        .with_state(state)
}

/// Middleware: count every request and measure how long it took.
async fn track_http(req: Request, next: Next) -> Response {
    let path = req
        .extensions()
        .get::<MatchedPath>()
        .map(|p| p.as_str().to_owned())
        .unwrap_or_else(|| "unknown".to_owned());
    let method = req.method().to_string();

    let in_flight = metrics::gauge!("http_requests_in_flight");
    in_flight.increment(1.0);
    let start = Instant::now();
    let response = next.run(req).await;
    let seconds = start.elapsed().as_secs_f64();
    in_flight.decrement(1.0);

    let status = response.status().as_u16().to_string();
    metrics::counter!("http_requests_total",
        "method" => method.clone(), "path" => path.clone(), "status" => status)
    .increment(1);
    metrics::histogram!("http_request_duration_seconds",
        "method" => method, "path" => path)
    .record(seconds);
    response
}

async fn get_order(
    State(state): State<AppState>,
    Path(id): Path<u32>,
) -> Result<String, StatusCode> {
    let n = state.counter.fetch_add(1, Ordering::Relaxed) + 1;
    tokio::time::sleep(Duration::from_millis(30)).await; // pretend database query
    if state.fail_every > 0 && n % state.fail_every == 0 {
        return Err(StatusCode::INTERNAL_SERVER_ERROR);
    }
    if id > 100 {
        return Err(StatusCode::NOT_FOUND);
    }
    Ok(format!("order {id}: 2 books\n"))
}

async fn render_metrics(State(state): State<AppState>) -> String {
    state.metrics.render()
}
```

**Что происходит:**

1. `PrometheusBuilder::new()...install_recorder()` создаёт рекордер, делает его глобальным и возвращает **дескриптор** (handle). `handle.render()` выдаёт текст для `/metrics`.
2. `set_buckets_for_metric` задаёт гистограмме длительности границы корзин (buckets) в секундах, от 5 мс до 1 с. Без корзин этот экспортёр пишет *сводку* (summary) вместо гистограммы (см. «Частые ошибки»).
3. Маленькая фоновая задача каждые 5 секунд вызывает `run_upkeep()`; документация крейта просит об этом, когда вы устанавливаете рекордер сами.
4. `fn app` собирает маршрутизатор в одном месте, чтобы тест мог собрать точно такой же (Упражнение 2).
5. Middleware `track_http` (`middleware::from_fn` появился в Уроке 16.4) работает вокруг **каждого** запроса. Он увеличивает **датчик** `http_requests_in_flight`, пока запрос выполняется, измеряет время, а затем добавляет 1 к **счётчику** `http_requests_total` и записывает длительность в **гистограмму**.
6. `MatchedPath` — это *шаблон маршрута* `/orders/{id}`, а не настоящий путь `/orders/7`. Так число значений метки остаётся маленьким. `route_layer` (вместо `layer`) запускает middleware только для известных маршрутов, где `MatchedPath` существует.
7. `get_order` считает свои вызовы атомарным счётчиком (Урок 14.5). Если задан `FAIL_EVERY`, каждый N-й вызов возвращает 500.
8. `shutdown_signal` — та же функция, что в Уроке 21.4: она ждёт `SIGTERM` или Ctrl+C. Kubernetes останавливает под сигналом `SIGTERM`; программа, которая ждёт только Ctrl+C, проигнорировала бы его (она работает как процесс 1 контейнера) и была бы убита после 30-секундного периода ожидания.

> 📏 **Соглашение:** имена метрик Prometheus пишутся в `snake_case`, в **базовых единицах** (`_seconds`, `_bytes`, никогда не миллисекунды) и с суффиксом `_total` у счётчиков. Значения меток — короткие слова с небольшим числом вариантов: `GET`, `/orders/{id}`, `500`.

Запустите сервис локально командой `FAIL_EVERY=4 cargo run` и во втором терминале отправьте шесть запросов заказов, один запрос несуществующего заказа и одну проверку здоровья, а затем прочитайте метрики:

```bash
for i in 1 2 3 4 5 6; do curl -s -o /dev/null -w '%{http_code} ' localhost:3000/orders/$i; done; echo
curl -s -o /dev/null -w '%{http_code}\n' localhost:3000/orders/500
curl -s localhost:3000/healthz
curl -s localhost:3000/metrics
```

Сокращённый вывод (мы убрали корзины `/healthz`):

```text
200 200 200 500 200 200
404
ok
# TYPE http_requests_total counter
http_requests_total{method="GET",path="/healthz",status="200"} 1
http_requests_total{method="GET",path="/orders/{id}",status="200"} 5
http_requests_total{method="GET",path="/orders/{id}",status="404"} 1
http_requests_total{method="GET",path="/orders/{id}",status="500"} 1

# TYPE http_requests_in_flight gauge
http_requests_in_flight 1

# TYPE http_request_duration_seconds histogram
...
http_request_duration_seconds_bucket{method="GET",path="/orders/{id}",le="0.025"} 0
http_request_duration_seconds_bucket{method="GET",path="/orders/{id}",le="0.05"} 7
http_request_duration_seconds_bucket{method="GET",path="/orders/{id}",le="0.1"} 7
...
http_request_duration_seconds_bucket{method="GET",path="/orders/{id}",le="+Inf"} 7
http_request_duration_seconds_sum{method="GET",path="/orders/{id}"} 0.224044791
http_request_duration_seconds_count{method="GET",path="/orders/{id}"} 7
```

**Что происходит:**

1. Четвёртый запрос заказа упал из-за `FAIL_EVERY=4`; заказа 500 не существует (404).
2. Каждая строка `# TYPE` называет метрику и её тип. Каждая следующая строка — **один временной ряд** и его текущее значение.
3. Датчик показывает 1: запрос, который сейчас читает `/metrics`, сам находится в обработке.
4. Гистограмма превращается в несколько рядов. `_bucket{le="0.05"} 7` означает «7 запросов заняли **не больше** 0,05 с» (`le` = less or equal, меньше или равно). Корзины *накопительные*: каждая включает меньшие. `_sum` и `_count` дают общее время и число запросов, поэтому среднее равно `0.224 / 7 ≈ 0.032` с.

| Тип | Как меняется | Пример вопроса | Пример |
|---|---|---|---|
| счётчик (counter) | только растёт (сбрасывается в 0 при перезапуске программы) | Сколько запросов в секунду? | `http_requests_total` |
| датчик (gauge) | растёт и уменьшается | Сколько прямо сейчас? | `http_requests_in_flight` |
| гистограмма (histogram) | считает значения по корзинам | Насколько медленны самые медленные 5%? | `http_request_duration_seconds` |

## 3. Prometheus в кластере

Создайте кластер, соберите образ и загрузите его в kind, как в Уроке 21.4. `Dockerfile` — файл с двумя стадиями сборки из Урока 11.13 с именем `p21-orders`:

```dockerfile
# Stage 1: build the program with the full Rust image.
FROM rust:1.100 AS build
WORKDIR /app
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release --locked

# Stage 2: copy only the program into a small image without a shell.
FROM gcr.io/distroless/cc-debian13
COPY --from=build /app/target/release/p21-orders /usr/local/bin/p21-orders
EXPOSE 3000
USER nonroot
ENTRYPOINT ["/usr/local/bin/p21-orders"]
```

```bash
kind create cluster --name course-21-09
docker build -t course-p21-orders:0.1.0 .
kind load docker-image course-p21-orders:0.1.0 --name course-21-09
```

Теперь `k8s/orders.yaml` с пространством имён, Deployment из двух подов и Service:

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
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: orders
          image: course-p21-orders:0.1.0
          env:
            - name: FAIL_EVERY
              value: "20"
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

Новое здесь только три **аннотации** в шаблоне пода. Это распространённая договорённость, а не возможность Kubernetes: «собирай мои метрики, порт 3000, путь `/metrics`». Настройка Prometheus ниже ищет именно их. Чтобы файл оставался коротким, в нём нет ресурсов и `securityContext` из Урока 21.4; в настоящем сервисе они остаются.

```bash
kubectl apply -f k8s/orders.yaml
kubectl -n shop rollout status deployment/orders
```

Установите Prometheus официальным чартом сообщества Prometheus. Чарт умеет ставить ещё Alertmanager и несколько вспомогательных программ; мы отключаем их, чтобы кластер оставался маленьким. `k8s/prometheus-values.yaml`:

```yaml
# Only the Prometheus server; the extra components are switched off.
alertmanager:
  enabled: false
prometheus-pushgateway:
  enabled: false
kube-state-metrics:
  enabled: false
prometheus-node-exporter:
  enabled: false
server:
  global:
    scrape_interval: 15s
  persistentVolume:
    enabled: false
```

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install prometheus prometheus-community/prometheus --version 29.47.2 \
  --namespace monitoring --create-namespace -f k8s/prometheus-values.yaml
kubectl -n monitoring rollout status deployment/prometheus-server
```

Сокращённый вывод:

```text
NAME: prometheus
NAMESPACE: monitoring
STATUS: deployed
...
The Prometheus server can be accessed via port 80 on the following DNS name from within your cluster:
prometheus-server.monitoring.svc.cluster.local
...
######   WARNING: Persistence is disabled!!! You will lose your data when   #####
######            the Server pod is terminated.                             #####
...
deployment "prometheus-server" successfully rolled out
```

Предупреждение ожидаемо: без постоянного тома данные живут столько же, сколько под, и на ноутбуке это нормально. Чарт настраивает задание сбора `kubernetes-pods`. Оно спрашивает Kubernetes API обо всех подах, **оставляет** только те, у которых есть `prometheus.io/scrape: "true"`, берёт порт и путь из двух других аннотаций и копирует пространство имён и имя пода в метки `namespace` и `pod`.

Откройте UI (User Interface) Prometheus через port-forward и оставьте его работать в отдельном терминале:

```bash
kubectl -n monitoring port-forward service/prometheus-server 9090:80
```

В браузере <http://localhost:9090/targets> показывает все цели. Тот же список доступен в виде JSON (JavaScript Object Notation, нотация объектов JavaScript) через HTTP API; `jq` (он есть в свежих версиях macOS) выбирает нужные поля:

```bash
curl -s 'localhost:9090/api/v1/targets?state=active' \
  | jq -r '.data.activeTargets[] | [.labels.job, .labels.pod // "-", .health] | @tsv'
```

```text
kubernetes-api-servers	-	up
kubernetes-nodes	-	up
kubernetes-nodes-cadvisor	-	up
kubernetes-pods	orders-7df76c9bb7-g2zr6	up
kubernetes-pods	orders-7df76c9bb7-l8dnb	up
kubernetes-service-endpoints	-	up
kubernetes-service-endpoints	-	up
prometheus	-	up
```

Оба пода найдены и в состоянии `up`. Чарт также собирает метрики самого Kubernetes и собственные метрики Prometheus.

> 🔒 **Безопасность:** `/metrics` может прочитать любой, кто достучится до пода. Никогда не кладите секреты или персональные данные в имена метрик и метки. В настоящем кластере сетевая политика (network policy) разрешает доступ к порту метрик только Prometheus.

## 4. Вопросы на PromQL

Метрики полезны только при трафике. Запустите внутри кластера маленький генератор нагрузки. Он использует официальный образ `busybox` и по кругу запрашивает заказы с 1 по 110 (101–110 не существуют):

```bash
kubectl -n shop run load --image=busybox:1.37 --restart=Never -- \
  sh -c 'while true; do for i in $(seq 1 110); do wget -q -O /dev/null http://orders/orders/$i 2>/dev/null; sleep 0.1; done; done'
```

Подождите две минуты, затем попробуйте запросы ниже на странице **Query** в UI Prometheus (<http://localhost:9090/query>) или через `curl`, как показано. В **PromQL** (Prometheus Query Language) два главных вида значений: *мгновенный вектор* (instant vector, одно значение на ряд, «сейчас») и *диапазонный вектор* (range vector, значения каждого ряда за временное окно, записывается как `[1m]`).

**Запрос 1 — сырой счётчик.** Одно имя метрики возвращает все ряды с их текущими значениями:

```bash
curl -s localhost:9090/api/v1/query --data-urlencode 'query=http_requests_total{pod="orders-7df76c9bb7-g2zr6"}' \
  | jq -c '.data.result[] | [.metric.path, .metric.status, .value[1]]'
```

```text
["/healthz","200","20"]
["/orders/{id}","500","34"]
["/metrics","200","12"]
["/orders/{id}","200","598"]
["/orders/{id}","404","57"]
```

Сырые счётчики только растут и начинают с 0 после перезапуска пода. Обычно же вас интересует «как быстро он растёт?».

**Запрос 2 — запросы в секунду по статусам.** `rate(x[1m])` берёт значения счётчика за последнюю минуту и вычисляет средний прирост **в секунду**; сбросы счётчика он учитывает. `sum by (status)` складывает ряды обоих подов и оставляет только метку `status`:

```text
sum by (status) (rate(http_requests_total{path="/orders/{id}"}[1m]))
```

```text
[{"status":"200"},"6.333333333333332"]
[{"status":"404"},"0.6444444444444444"]
[{"status":"500"},"0.37777777777777777"]
```

Всего около 7,4 запроса заказов в секунду; 0,38 в секунду — с ошибкой.

**Запрос 3 — доля ошибок.** Разделите неудачные запросы на все:

```text
sum(rate(http_requests_total{path="/orders/{id}", status=~"5.."}[1m]))
  / sum(rate(http_requests_total{path="/orders/{id}"}[1m]))
```

```text
[{},"0.05135951661631421"]
```

`status=~"5.."` — совпадение по регулярному выражению: любой статус, начинающийся с 5. Результат — 5,1%, ровно столько, сколько должен дать `FAIL_EVERY=20`. В Уроке 21.11 вы превратите эту долю в цель надёжности.

**Запрос 4 — 95-й перцентиль задержки.** `histogram_quantile` оценивает **перцентиль** по корзинам:

```text
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket{path="/orders/{id}"}[1m])))
```

```text
[{},"0.0488219696969697"]
```

95% запросов заказов заняли меньше примерно 49 мс. Обратите внимание на слово *оценивает*: Prometheus знает только, что запросы попали между границами корзин 25 мс и 50 мс, и предполагает, что внутри корзины они распределены равномерно. На самом деле запросы занимали около 31 мс. Выбирайте границы корзин рядом со значениями, которые для вас важны.

> ⚡ **Производительность:** каждое значение метки умножает число рядов. У `path` 3 значения, у `status` 4, у `method` 1, и подов 2: не больше 24 рядов счётчика и по 11 рядов гистограммы (корзины + sum + count) на каждую пару «путь — под». Метка вроде `user_id` или сырой путь `/orders/7` создали бы ряд на каждого пользователя или каждый заказ, и память Prometheus росла бы с каждым из них (Упражнение 3 это показывает).

## 5. Графики в Grafana

Собственный UI Prometheus хорош, чтобы пробовать запросы. Для дашбордов, на которые команда смотрит каждый день, используют **Grafana**. Разверните её с **заранее подготовленным** (provisioned) источником данных: файл в ConfigMap сообщает Grafana при запуске, где находится Prometheus, и никому не нужно собирать это щелчками. `k8s/grafana.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  prometheus.yaml: |
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        uid: prometheus
        url: http://prometheus-server.monitoring.svc
        access: proxy
        isDefault: true
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
        - name: grafana
          image: grafana/grafana:13.3.1
          ports:
            - containerPort: 3000
          readinessProbe:
            httpGet:
              path: /api/health
              port: 3000
          volumeMounts:
            - name: datasources
              mountPath: /etc/grafana/provisioning/datasources
      volumes:
        - name: datasources
          configMap:
            name: grafana-datasources
---
apiVersion: v1
kind: Service
metadata:
  name: grafana
  namespace: monitoring
spec:
  selector:
    app: grafana
  ports:
    - port: 3000
      targetPort: 3000
```

**Что происходит:**

1. При запуске Grafana читает все файлы YAML (YAML Ain't Markup Language) в `/etc/grafana/provisioning/datasources`. ConfigMap смонтирован ровно туда.
2. URL (Uniform Resource Locator, веб-адрес) источника данных — имя Service Prometheus внутри кластера (DNS, Domain Name System, система доменных имён; DNS кластера описан в Уроке 21.2). `access: proxy` означает, что запросы отправляет сервер Grafana, а не ваш браузер.
3. Проба готовности на `/api/health` не пускает под в Service, пока Grafana действительно не начнёт слушать порт.

```bash
kubectl apply -f k8s/grafana.yaml
kubectl -n monitoring rollout status deployment/grafana
kubectl -n monitoring port-forward service/grafana 3001:3000
```

Мы используем локальный порт 3001, чтобы порт 3000 оставался свободным для вашего собственного сервиса. Проверьте API из другого терминала:

```bash
curl -s localhost:3001/api/health
curl -s -u admin:admin localhost:3001/api/datasources | jq -c '.[] | {name,type,url}'
```

```text
{
  "database": "ok",
  "version": "13.3.1",
  "commit": "da3aa3ff574e7dbbf808c66b6daa744defed0441"
}
{"name":"Prometheus","type":"prometheus","url":"http://prometheus-server.monitoring.svc"}
```

Теперь откройте <http://localhost:3001> и войдите как `admin` с паролем `admin` (Grafana попросит выбрать новый; для этого локального кластера можно пропустить). Постройте первый дашборд:

1. **Dashboards → New → New dashboard → Add visualization**, выберите источник данных **Prometheus**.
2. Переключите редактор запроса с *Builder* на *Code* и вставьте Запрос 2 из раздела 4. В поле **Legend** напишите `{{status}}`, чтобы каждая линия называлась по своему статусу.
3. Задайте заголовок панели «Order requests per second» и нажмите **Save dashboard**.
4. Добавьте вторую панель с Запросом 4 и единицей **seconds (s)** в настройках панели: «p95 latency».

Вы увидите три линии для 200, 404 и 500, которые немного колеблются около 6,3, 0,6 и 0,4, и линию задержки около 0,049 с.

> 🔒 **Безопасность:** `admin`/`admin` допустимо только на вашем ноутбуке. В настоящем кластере задайте переменную `GF_SECURITY_ADMIN_PASSWORD` из Secret (Урок 21.5) и не открывайте Grafana в интернет без входа перед ней.

> 💡 **Совет:** дашборды тоже можно подготовить заранее: экспортируйте дашборд как JSON (**Share → Export**) и смонтируйте его из ConfigMap в `/etc/grafana/provisioning/dashboards` — тогда дашборды живут в Git, как всё остальное. В больших установках часто используют Prometheus Operator (чарт `kube-prometheus-stack`), где цели сбора описываются объектами Kubernetes (`ServiceMonitor`), а не аннотациями.

## Частые ошибки

> ⚠️ **Частая ошибка:** забыть `set_buckets_for_metric`. Тогда `metrics-exporter-prometheus` выводит длительность как **сводку** (summary): перцентили вычисляются внутри каждого пода.

```text
# TYPE http_request_duration_seconds summary
http_request_duration_seconds{method="GET",path="/orders/{id}",quantile="0.5"} 0.035518555954252656
http_request_duration_seconds{method="GET",path="/orders/{id}",quantile="0.95"} 0.035518555954252656
```

Перцентили двух подов нельзя сложить или усреднить так, чтобы получился правильный перцентиль всего сервиса. Корзины складывать можно (`sum by (le)`), поэтому мы используем гистограмму.

> ⚠️ **Частая ошибка:** собирать метрики через **Service**, а не с подов (статическая цель `orders.shop.svc:80`). Каждый сбор попадает на случайный под, счётчик прыгает между значениями разных подов, и `rate()` показывает чепуху. Собирайте метрики с каждого пода; задание `kubernetes-pods` так и делает.

> ⚠️ **Частая ошибка:** слишком короткий диапазон. При интервале сбора 15 секунд в `rate(x[15s])` часто меньше двух значений, и запрос ничего не возвращает. Берите диапазон не меньше четырёх интервалов сбора: `[1m]`.

## Попробуйте сами

1. В UI Prometheus выполните `sum by (pod) (rate(http_requests_total{namespace="shop"}[1m]))`. Вы получите по линии на под; Service распределяет нагрузку примерно поровну.
2. Увеличьте число подов до трёх: `kubectl -n shop scale deployment orders --replicas=3`. Примерно через 30 секунд на странице целей появится третья цель — без единого изменения в Prometheus.
3. Когда закончите урок: `kubectl -n shop delete pod load`, затем `kind delete cluster --name course-21-09`.

## Упражнения

### Упражнение 1 — Доля 404 (легко)

Напишите запрос PromQL, который возвращает долю запросов заказов со статусом 404 — число от 0 до 1. С генератором нагрузки выше оно должно быть близко к `10 / 110 ≈ 0.09`.

<details>
<summary>Подсказка</summary>

Это Запрос 3 с другим условием на статус. Точное совпадение пишется через `=`, а не `=~`.

</details>

<details>
<summary>Ответ</summary>

```text
sum(rate(http_requests_total{path="/orders/{id}", status="404"}[1m]))
  / sum(rate(http_requests_total{path="/orders/{id}"}[1m]))
```

По значениям из Запроса 2: `0.644 / (6.333 + 0.644 + 0.378) ≈ 0.088`. Почему это работает: обе части берут один и тот же путь и одно и то же окно, поэтому отношение — доля ответов 404. Оно чуть меньше 10/110, потому что часть запросов заказов 101–110 попала под правило «каждый 20-й падает» и вернула 500.

</details>

### Упражнение 2 — Тест, что запрос посчитан (средне)

**Напишите тест** в `src/main.rs`, который отправляет `GET /orders/7` в маршрутизатор (без сети, как в разделе 8 Урока 16.3), затем читает `/metrics` и проверяет, что там есть `http_requests_total{method="GET",path="/orders/{id}",status="200"} 1`. Глобальный рекордер использовать нельзя, чтобы тесты не мешали друг другу.

<details>
<summary>Подсказка</summary>

Добавьте dev-зависимости из Урока 16.3: `cargo add --dev tower@0.5 --features util` и `cargo add --dev http-body-util@0.1`. `PrometheusBuilder::new().build_recorder()` создаёт рекордер, не устанавливая его; `metrics::set_default_local_recorder(&recorder)` делает его активным **для текущего потока**, пока жив возвращённый guard-объект. Используйте `#[tokio::test(flavor = "current_thread")]`, чтобы весь тест работал в одном потоке.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use axum::body::Body;
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    async fn get_text(app: Router, uri: &str) -> String {
        let request = axum::http::Request::get(uri).body(Body::empty()).unwrap();
        let response = app.oneshot(request).await.unwrap();
        let bytes = response.into_body().collect().await.unwrap().to_bytes();
        String::from_utf8(bytes.to_vec()).unwrap()
    }

    #[tokio::test(flavor = "current_thread")]
    async fn a_request_is_counted() {
        // A recorder only for this test, active on this thread.
        let recorder = PrometheusBuilder::new().build_recorder();
        let _guard = metrics::set_default_local_recorder(&recorder);
        let state = AppState {
            metrics: recorder.handle(),
            fail_every: 0,
            counter: Arc::new(AtomicU64::new(0)),
        };
        let app = app(state);

        get_text(app.clone(), "/orders/7").await;
        let text = get_text(app, "/metrics").await;

        assert!(
            text.contains(
                r#"http_requests_total{method="GET",path="/orders/{id}",status="200"} 1"#
            ),
            "metrics were:\n{text}"
        );
    }
}
```

```text
running 1 test
test tests::a_request_is_counted ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.23s
```

Почему это работает: локальный рекордер ловит все вызовы `counter!` и `histogram!` в этом потоке, а однопоточный рантайм выполняет маршрутизатор именно в нём. Сообщение печатает весь текст метрик, если проверка не прошла, поэтому ошибку легко понять. Тест заодно показывает, что меткой служит шаблон маршрута, а не `/orders/7`.

</details>

### Упражнение 3 — Посмотрите, как растёт кардинальность (сложно)

В копии проекта замените код с `MatchedPath` в `track_http` на `let path = req.uri().path().to_owned();`. Запустите сервис локально, запросите заказы с 1 по 50 и посчитайте строки `/metrics`, которые начинаются с `http_requests_total` и с `http_request_duration_seconds`. Сравните с исходным кодом. Объясните, что случилось бы при миллионе заказов.

<details>
<summary>Подсказка</summary>

`curl -s localhost:3000/metrics | grep -c '^http_requests_total'` считает строки. Отправьте одинаковые запросы обеим версиям.

</details>

<details>
<summary>Ответ</summary>

С сырым путём (две команды `grep` выполняются одна за другой, поэтому вторая уже видит первый запрос `/metrics` как путь):

```text
50
561
```

С `MatchedPath` те же команды печатают:

```text
1
22
```

Почему это работает: каждое новое значение метки создаёт новые ряды. С сырым путём каждый номер заказа — новое значение: 50 рядов счётчика и 51 путь × 11 строк гистограммы. Миллион заказов означал бы миллионы рядов на каждый под, а это расход памяти в сервисе и в Prometheus и замедление каждого запроса. У шаблона маршрута фиксированное небольшое число значений при любом трафике.

</details>

## Вопросы для самопроверки

1. Чем отличаются «push» и «pull» для метрик и какой способ использует Prometheus?

<details>
<summary>Ответ</summary>

При push программа сама отправляет метрики в коллектор (Урок 16.12). При pull, который использует Prometheus, программа показывает их на `/metrics`, а Prometheus читает их каждый интервал сбора. Pull к тому же сообщает Prometheus, что цель не работает.

</details>

2. Почему мы рисуем `rate(http_requests_total[1m])`, а не `http_requests_total`?

<details>
<summary>Ответ</summary>

Счётчик только растёт (и сбрасывается при перезапуске), поэтому его сырое значение мало что говорит. `rate` превращает его в «в секунду за последнюю минуту» и учитывает сбросы.

</details>

3. Что означает `http_request_duration_seconds_bucket{le="0.05"} 7`?

<details>
<summary>Ответ</summary>

Семь запросов заняли не больше 0,05 секунды. Корзины накопительные: они включают все меньшие корзины.

</details>

4. Почему p95 из `histogram_quantile` — только оценка?

<details>
<summary>Ответ</summary>

Prometheus хранит только количество значений в каждой корзине, а не сами значения. Он предполагает, что значения равномерно распределены внутри корзины, куда попал перцентиль. Границы корзин рядом с важными значениями делают оценку точнее.

</details>

5. Почему в метках никогда не должно быть идентификаторов вроде `user_id`?

<details>
<summary>Ответ</summary>

Каждое значение создаёт новые временные ряды (высокая кардинальность). Миллионы рядов перегружают память сервиса и Prometheus. Идентификаторы кладите в логи и трассировки.

</details>

## Итоги

- Prometheus забирает метрики сам: каждый под показывает текстовую страницу `/metrics`, а Prometheus собирает её, здесь каждые 15 секунд.
- В Rust фасад `metrics` вместе с `metrics-exporter-prometheus` даёт счётчики, датчики и гистограммы; middleware измеряет каждый запрос.
- Используйте базовые единицы (`_seconds`), `_total` для счётчиков и шаблоны маршрутов, а не сырые пути, в метках.
- Helm-чарт Prometheus находит поды через Kubernetes API и аннотации `prometheus.io/*`.
- PromQL: `rate()` для счётчиков, `sum by (...)`, чтобы объединить поды, отношение для доли ошибок, `histogram_quantile` для перцентилей.
- Grafana рисует дашборды по данным Prometheus; готовьте источники данных (и дашборды) из файлов и защищайте вход.

## Источники и что почитать

- [Prometheus — Overview](https://prometheus.io/docs/introduction/overview/) и [Metric types](https://prometheus.io/docs/concepts/metric_types/) — как работает Prometheus и четыре типа метрик.
- [Prometheus — Querying basics](https://prometheus.io/docs/prometheus/latest/querying/basics/) и [Query functions](https://prometheus.io/docs/prometheus/latest/querying/functions/) — PromQL, `rate` и `histogram_quantile`.
- [Prometheus — Metric and label naming](https://prometheus.io/docs/practices/naming/) и [Histograms and summaries](https://prometheus.io/docs/practices/histograms/) — соглашения и почему гистограммы можно агрегировать.
- [Prometheus — Exposition formats](https://prometheus.io/docs/instrumenting/exposition_formats/) — точный текстовый формат `/metrics`.
- [Helm-чарты prometheus-community](https://github.com/prometheus-community/helm-charts) — чарт из этого урока и его значения.
- [metrics на docs.rs](https://docs.rs/metrics) и [metrics-exporter-prometheus на docs.rs](https://docs.rs/metrics-exporter-prometheus) — API на Rust, включая upkeep и локальные рекордеры.
- [Grafana — Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/) и [Get started with Grafana and Prometheus](https://grafana.com/docs/grafana/latest/getting-started/get-started-grafana-prometheus/) — источники данных и дашборды из файлов и первый дашборд.

---

[🏠 Начало](../START-HERE.md) · [← 21.8 Инфраструктура как код с OpenTofu](08-infrastructure-as-code-with-opentofu.md) · [📋 Этап 21](README.md) · [21.10 Логи и трассировки в кластере →](10-logs-and-traces-in-a-cluster.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
