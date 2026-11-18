<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 21.8 Infrastructure as code with OpenTofu](08-infrastructure-as-code-with-opentofu.md) · [📋 Phase 21](README.md) · [21.10 Logs and traces in a cluster →](10-logs-and-traces-in-a-cluster.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) <!-- nav -->

# Lesson 21.9 — Metrics with Prometheus and Grafana

> **Phase 21 — Cloud-native infrastructure** · Lesson 9 of 14 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How Prometheus collects metrics: it *pulls* them from a `/metrics` page of every pod.
- How a Rust service publishes counters, gauges and histograms with the `metrics` crate.
- How to install Prometheus in a kind cluster with Helm and let it find your pods by itself.
- How to ask questions with PromQL: request rate, error ratio and the 95th percentile of latency.
- How to show the answers in Grafana, and why labels must have few values.

## Before you start

- You finished [Lesson 21.8 — Infrastructure as code with OpenTofu](08-infrastructure-as-code-with-opentofu.md).
- You know what metrics, counters and histograms are from [Lesson 16.12 — Logs, metrics and traces with OpenTelemetry](../16-backend-services/12-logs-metrics-and-traces.md). There the service *pushed* metrics to a collector; here Prometheus *pulls* them.
- You can deploy an image to kind ([Lesson 21.4](04-deploying-a-rust-service-to-kubernetes.md)), use probes ([Lesson 21.5](05-configmaps-secrets-and-probes.md)) and install a Helm chart ([Lesson 21.6](06-helm-and-kustomize.md)).
- Your code goes in `~/projects/rust/code/p21-orders`. Lessons 21.10 and 21.11 continue with the same service.

## New words

| Word | Meaning |
|---|---|
| **Prometheus** | An open-source monitoring system. It collects numbers from programs at regular times, stores them, answers queries and can raise alerts. |
| **scrape** | One visit of Prometheus to a program's `/metrics` page to read the current values. It happens every *scrape interval* (here 15 seconds). |
| **target** | One address that Prometheus scrapes, for example one pod. |
| **time series** | One metric name plus one exact set of labels, with its values over time: `http_requests_total{status="500", pod="orders-1"}`. |
| **label** | A key-value pair that splits one metric into several time series, such as `status="404"`. |
| **gauge** | A metric that can go up and down: requests in progress, memory in use, queue length. |
| **PromQL** (Prometheus Query Language) | The language for asking Prometheus questions, for example "requests per second, per status, over the last minute". |
| **percentile** | The value below which a given share of measurements falls. The 95th percentile (p95) of latency is the time that 95% of requests stay under. |
| **cardinality** | How many different time series a metric has. Each new label value adds series; too many overload Prometheus. |
| **Grafana** | An open-source web tool that draws graphs and dashboards from data sources such as Prometheus. |
| **data source** | A connection in Grafana to a database it reads from (here: Prometheus). |
| **dashboard / panel** | A Grafana page / one graph or number on that page, driven by one query. |
| **HTTP** (HyperText Transfer Protocol) / **JSON** (JavaScript Object Notation) / **URL** (Uniform Resource Locator) | The web protocol / a text format for data / a web address. All three appear in the Prometheus and Grafana APIs. |
| **UI** (User Interface) | The part of a program that people look at and click, here the web pages of Prometheus and Grafana. |

## 1. How Prometheus works

In Lesson 16.12 the service sent (*pushed*) its metrics to an OpenTelemetry Collector. **Prometheus** usually works the other way around: every program shows its current numbers on a plain text page, `/metrics`, and Prometheus comes to read (*pulls*) that page every few seconds. One visit is called a **scrape**.

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

Pulling has two advantages. Prometheus knows when a target is *down* (the scrape fails), and a program does not need to know where Prometheus is. In Kubernetes, Prometheus asks the Kubernetes API (Application Programming Interface) which pods exist, so new pods are found automatically.

Every scrape stores one value for every **time series**. A time series is a metric name plus an exact set of **labels**. `http_requests_total{status="200"}` and `http_requests_total{status="500"}` are two different series of the same metric.

## 2. Metrics from the Rust service

Our service is the small order service from Lesson 16.12, now with three endpoints: `GET /orders/{id}`, `GET /healthz` for the probes, and `GET /metrics` for Prometheus. To practise with errors, the environment variable `FAIL_EVERY=20` makes every 20th order request fail with status 500.

```bash
cd ~/projects/rust/code
cargo new p21-orders
cd p21-orders
cargo add axum@0.8 metrics@0.24
cargo add metrics-exporter-prometheus@0.18 --no-default-features
cargo add tokio@1 --features macros,rt-multi-thread,signal,time
```

The **`metrics`** crate is a *facade*: your code calls `counter!`, `gauge!` and `histogram!`, and a separately installed *recorder* decides what happens with the values. `metrics-exporter-prometheus` is a recorder that keeps them in memory and renders them in the Prometheus text format. We turn off its default features, because they bring their own HTTP (HyperText Transfer Protocol) server and we already have axum.

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

**What happens:**

1. `PrometheusBuilder::new()...install_recorder()` creates the recorder, makes it the global one, and returns a **handle**. `handle.render()` produces the text for `/metrics`.
2. `set_buckets_for_metric` gives the duration histogram its bucket limits in seconds, from 5 ms to 1 s. Without buckets this exporter writes a *summary* instead of a histogram (see Common mistakes).
3. The small background task calls `run_upkeep()` every 5 seconds; the crate's documentation asks for this when you install the recorder yourself.
4. `fn app` builds the router in one place, so a test can build exactly the same router (Exercise 2).
5. The middleware `track_http` (Lesson 16.4 introduced `middleware::from_fn`) runs around **every** request. It raises the **gauge** `http_requests_in_flight` while the request is running, measures the time, and then adds 1 to the **counter** `http_requests_total` and records the duration in the **histogram**.
6. `MatchedPath` is the *route pattern* `/orders/{id}`, not the real path `/orders/7`. This keeps the number of label values small. `route_layer` (instead of `layer`) runs the middleware only for known routes, where `MatchedPath` exists.
7. `get_order` counts its calls with an atomic counter (Lesson 14.5). When `FAIL_EVERY` is set, every N-th call returns 500.
8. `shutdown_signal` is the one from Lesson 21.4: it waits for `SIGTERM` or Ctrl+C. Kubernetes stops a pod with `SIGTERM`; a program that waited only for Ctrl+C would ignore it (it runs as process 1 of the container) and be killed after the 30-second grace period.

> 📏 **Convention:** Prometheus metric names use `snake_case`, a **base unit** (`_seconds`, `_bytes`, never milliseconds) and the suffix `_total` for counters. Label values are short words with few possibilities: `GET`, `/orders/{id}`, `500`.

Run it locally with `FAIL_EVERY=4 cargo run` and, in a second terminal, send six order requests, one unknown order and one health check, then read the metrics:

```bash
for i in 1 2 3 4 5 6; do curl -s -o /dev/null -w '%{http_code} ' localhost:3000/orders/$i; done; echo
curl -s -o /dev/null -w '%{http_code}\n' localhost:3000/orders/500
curl -s localhost:3000/healthz
curl -s localhost:3000/metrics
```

Shortened output (we removed the buckets of `/healthz`):

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

**What happens:**

1. The fourth order request failed because of `FAIL_EVERY=4`; order 500 does not exist (404).
2. Each `# TYPE` line names a metric and its type. Each following line is **one time series** and its current value.
3. The gauge shows 1: the request that is reading `/metrics` right now is itself in flight.
4. A histogram becomes several series. `_bucket{le="0.05"} 7` means "7 requests took **at most** 0.05 s" (`le` = less or equal). The buckets are *cumulative*: each one includes the smaller ones. `_sum` and `_count` give the total time and the number of requests, so the average is `0.224 / 7 ≈ 0.032` s.

| Type | Goes | Example question | Example |
|---|---|---|---|
| counter | only up (back to 0 when the program restarts) | How many requests per second? | `http_requests_total` |
| gauge | up and down | How many right now? | `http_requests_in_flight` |
| histogram | counts values in buckets | How slow are the slowest 5%? | `http_request_duration_seconds` |

## 3. Prometheus in the cluster

Create a cluster, build the image and load it into kind as in Lesson 21.4. The `Dockerfile` is the two-stage file from Lesson 11.13 with the name `p21-orders`:

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

Now `k8s/orders.yaml` with a namespace, a Deployment of two pods and a Service:

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

The three **annotations** on the pod template are the only new part. They are a common agreement, not a Kubernetes feature: "scrape me, on port 3000, at `/metrics`". The Prometheus set-up below looks for them. To keep the file short, it leaves out the resources and the `securityContext` of Lesson 21.4; a real service keeps them.

```bash
kubectl apply -f k8s/orders.yaml
kubectl -n shop rollout status deployment/orders
```

Install Prometheus with the official chart of the Prometheus community. The chart can also install Alertmanager and several helper programs; we switch them off to keep the cluster small. `k8s/prometheus-values.yaml`:

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

Shortened output:

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

The warning is expected: without a persistent volume the data lives only as long as the pod, which is fine on a laptop. The chart configures a scrape job called `kubernetes-pods`. It asks the Kubernetes API for all pods, **keeps** only those with `prometheus.io/scrape: "true"`, takes port and path from the other two annotations, and copies the namespace and pod name into the labels `namespace` and `pod`.

Open the Prometheus UI (User Interface) through a port-forward and keep it running in its own terminal:

```bash
kubectl -n monitoring port-forward service/prometheus-server 9090:80
```

In the browser, <http://localhost:9090/targets> lists every target. The same list is available as JSON (JavaScript Object Notation) from the HTTP API; `jq` (preinstalled on recent macOS) picks the interesting fields:

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

Both pods were found and are `up`. The chart also scrapes Kubernetes itself and Prometheus's own metrics.

> 🔒 **Security:** `/metrics` is readable by anyone who can reach the pod. Never put secrets or personal data into metric names or labels. In a real cluster, a network policy allows only Prometheus to reach the metrics port.

## 4. Asking questions with PromQL

Metrics are only useful with traffic. Start a small load generator inside the cluster. It uses the official `busybox` image and requests orders 1 to 110 in a loop (101–110 do not exist):

```bash
kubectl -n shop run load --image=busybox:1.37 --restart=Never -- \
  sh -c 'while true; do for i in $(seq 1 110); do wget -q -O /dev/null http://orders/orders/$i 2>/dev/null; sleep 0.1; done; done'
```

Wait two minutes, then try the queries below on the **Query** page of the Prometheus UI (<http://localhost:9090/query>), or with `curl` as shown. **PromQL** (Prometheus Query Language) has two main kinds of values: an *instant vector* (one value per series, "now") and a *range vector* (the values of each series over a time window, written `[1m]`).

**Query 1 — the raw counter.** A metric name alone returns every series with its current value:

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

Raw counters only grow, and they restart from 0 when a pod restarts. The question you usually have is "how fast is it growing?".

**Query 2 — requests per second by status.** `rate(x[1m])` takes the counter values of the last minute and computes the average increase **per second**; it handles counter resets. `sum by (status)` adds the series of both pods together and keeps only the `status` label:

```text
sum by (status) (rate(http_requests_total{path="/orders/{id}"}[1m]))
```

```text
[{"status":"200"},"6.333333333333332"]
[{"status":"404"},"0.6444444444444444"]
[{"status":"500"},"0.37777777777777777"]
```

About 7.4 order requests per second in total; 0.38 per second fail.

**Query 3 — the error ratio.** Divide failed requests by all requests:

```text
sum(rate(http_requests_total{path="/orders/{id}", status=~"5.."}[1m]))
  / sum(rate(http_requests_total{path="/orders/{id}"}[1m]))
```

```text
[{},"0.05135951661631421"]
```

`status=~"5.."` is a regular-expression match: every status that starts with 5. The result is 5.1%, exactly what `FAIL_EVERY=20` should give. You will turn this ratio into a reliability goal in Lesson 21.11.

**Query 4 — the 95th percentile of latency.** `histogram_quantile` estimates a **percentile** from the buckets:

```text
histogram_quantile(0.95,
  sum by (le) (rate(http_request_duration_seconds_bucket{path="/orders/{id}"}[1m])))
```

```text
[{},"0.0488219696969697"]
```

95% of order requests took less than about 49 ms. Note the word *estimates*: Prometheus knows only that the requests fell between the bucket limits 25 ms and 50 ms, and it assumes they are spread evenly inside that bucket. The real requests took about 31 ms. Choose bucket limits close to the values you care about.

> ⚡ **Performance:** Every label value multiplies the number of series. `path` has 3 values, `status` 4, `method` 1, and there are 2 pods: at most 24 counter series, and 11 histogram series (buckets + sum + count) per path and pod. A label like `user_id` or the raw path `/orders/7` would create a series per user or per order, and Prometheus memory grows with every one (Exercise 3 shows it).

## 5. Graphs in Grafana

Prometheus's own UI is good for trying queries. For dashboards that a team looks at every day, people use **Grafana**. Deploy it with a **provisioned** data source: a file in a ConfigMap tells Grafana at start-up where Prometheus is, so nobody has to click it together. `k8s/grafana.yaml`:

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

**What happens:**

1. Grafana reads every YAML (YAML Ain't Markup Language) file in `/etc/grafana/provisioning/datasources` when it starts. The ConfigMap is mounted exactly there.
2. The data source URL (Uniform Resource Locator, a web address) is the Service name of Prometheus inside the cluster (DNS, the Domain Name System of the cluster, from Lesson 21.2). `access: proxy` means the Grafana server sends the queries, not your browser.
3. The readiness probe on `/api/health` keeps the pod out of the Service until Grafana is really listening.

```bash
kubectl apply -f k8s/grafana.yaml
kubectl -n monitoring rollout status deployment/grafana
kubectl -n monitoring port-forward service/grafana 3001:3000
```

We use local port 3001, so that port 3000 stays free for your own service. Check the API from another terminal:

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

Now open <http://localhost:3001> and log in as `admin` with the password `admin` (Grafana asks you to choose a new one; for this local cluster you may skip). Build a first dashboard:

1. **Dashboards → New → New dashboard → Add visualization**, choose the data source **Prometheus**.
2. Switch the query editor from *Builder* to *Code* and paste Query 2 from Section 4. Write `{{status}}` into **Legend** so each line is named after its status.
3. Set the panel title "Order requests per second" and click **Save dashboard**.
4. Add a second panel with Query 4 and the unit **seconds (s)** in the panel options: "p95 latency".

You see three lines for 200, 404 and 500 that move a little around 6.3, 0.6 and 0.4, and a latency line near 0.049 s.

> 🔒 **Security:** `admin`/`admin` is acceptable only on your laptop. In a real cluster, set the variable `GF_SECURITY_ADMIN_PASSWORD` from a Secret (Lesson 21.5) and do not expose Grafana to the internet without a login in front of it.

> 💡 **Tip:** Dashboards can be provisioned too: export a dashboard as JSON (**Share → Export**) and mount it from a ConfigMap into `/etc/grafana/provisioning/dashboards`, so dashboards live in Git like the rest. Bigger set-ups often use the Prometheus Operator (chart `kube-prometheus-stack`), where you describe scrape targets as Kubernetes objects (`ServiceMonitor`) instead of annotations.

## Common mistakes

> ⚠️ **Common mistake:** Forgetting `set_buckets_for_metric`. Then `metrics-exporter-prometheus` renders the duration as a **summary**: the percentiles are computed inside each pod.

```text
# TYPE http_request_duration_seconds summary
http_request_duration_seconds{method="GET",path="/orders/{id}",quantile="0.5"} 0.035518555954252656
http_request_duration_seconds{method="GET",path="/orders/{id}",quantile="0.95"} 0.035518555954252656
```

Percentiles from two pods cannot be added or averaged into a correct percentile for the whole service. Buckets can be summed (`sum by (le)`), which is why we use a histogram.

> ⚠️ **Common mistake:** Scraping the **Service** instead of the pods (a static target `orders.shop.svc:80`). Every scrape lands on a random pod, so the counter jumps between the values of different pods, and `rate()` shows nonsense. Scrape every pod; the `kubernetes-pods` job does that.

> ⚠️ **Common mistake:** A range that is too short. With a 15-second scrape interval, `rate(x[15s])` often has fewer than two samples and returns nothing. Use at least four times the scrape interval: `[1m]`.

## Try it yourself

1. In the Prometheus UI, run `sum by (pod) (rate(http_requests_total{namespace="shop"}[1m]))`. You get one line per pod; the Service spreads the load roughly evenly.
2. Scale to three pods with `kubectl -n shop scale deployment orders --replicas=3`. Within about 30 seconds, a third target appears on the targets page without any change to Prometheus.
3. When you are done with this lesson: `kubectl -n shop delete pod load`, then `kind delete cluster --name course-21-09`.

## Exercises

### Exercise 1 — The 404 ratio (easy)

Write a PromQL query that returns the share of order requests that end with status 404, as a number between 0 and 1. With the load generator above it should be close to `10 / 110 ≈ 0.09`.

<details>
<summary>Hint</summary>

It is Query 3 with a different status matcher. An exact match uses `=`, not `=~`.

</details>

<details>
<summary>Answer</summary>

```text
sum(rate(http_requests_total{path="/orders/{id}", status="404"}[1m]))
  / sum(rate(http_requests_total{path="/orders/{id}"}[1m]))
```

With the rates from Query 2: `0.644 / (6.333 + 0.644 + 0.378) ≈ 0.088`. Why it works: both sides cover the same path and time window, so the ratio is the share of 404 answers. It is a little below 10/110 because some requests for orders 101–110 hit the "every 20th fails" rule and returned 500 instead.

</details>

### Exercise 2 — Test that a request is counted (medium)

**Write a test** in `src/main.rs` that sends `GET /orders/7` to the router (without a network, as in Lesson 16.3, Section 8), then reads `/metrics` and checks that it contains `http_requests_total{method="GET",path="/orders/{id}",status="200"} 1`. The global recorder must not be used, so that tests cannot disturb each other.

<details>
<summary>Hint</summary>

Add the dev-dependencies from Lesson 16.3: `cargo add --dev tower@0.5 --features util` and `cargo add --dev http-body-util@0.1`. `PrometheusBuilder::new().build_recorder()` builds a recorder without installing it; `metrics::set_default_local_recorder(&recorder)` makes it active **for the current thread** until the returned guard is dropped. Use `#[tokio::test(flavor = "current_thread")]` so that the whole test runs on one thread.

</details>

<details>
<summary>Answer</summary>

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

Why it works: the local recorder catches every `counter!` and `histogram!` call made on this thread, and the current-thread runtime runs the router on that thread. The message prints the whole metrics text if the check fails, which makes a failure easy to understand. The test also shows that the label is the route pattern, not `/orders/7`.

</details>

### Exercise 3 — See cardinality grow (hard)

In a copy of the project, replace the `MatchedPath` code in `track_http` with `let path = req.uri().path().to_owned();`. Run the service locally, request orders 1 to 50, and count the lines of `/metrics` that start with `http_requests_total` and with `http_request_duration_seconds`. Compare with the original code. Explain what would happen with a million orders.

<details>
<summary>Hint</summary>

`curl -s localhost:3000/metrics | grep -c '^http_requests_total'` counts lines. Run the same requests against both versions.

</details>

<details>
<summary>Answer</summary>

With the raw path (the two `grep` commands run one after the other, so the second one already sees the first `/metrics` request as a path):

```text
50
561
```

With `MatchedPath`, the same commands print:

```text
1
22
```

Why it works: each distinct label value creates new series. With the raw path, every order id is a new value: 50 counter series, and 51 paths × 11 histogram lines. A million orders would mean millions of series per pod, which costs memory in the service and in Prometheus and makes every query slow. The route pattern has a fixed, small number of values, whatever the traffic.

</details>

## Quiz

1. What is the difference between "push" and "pull" for metrics, and which one does Prometheus use?

<details>
<summary>Answer</summary>

With push the program sends its metrics to a collector (Lesson 16.12). With pull, which Prometheus uses, the program shows them on `/metrics` and Prometheus reads them every scrape interval. Pull also tells Prometheus when a target is down.

</details>

2. Why do we graph `rate(http_requests_total[1m])` and not `http_requests_total`?

<details>
<summary>Answer</summary>

A counter only grows (and resets on restart), so its raw value says little. `rate` turns it into "per second over the last minute" and handles resets.

</details>

3. What does `http_request_duration_seconds_bucket{le="0.05"} 7` mean?

<details>
<summary>Answer</summary>

Seven requests took at most 0.05 seconds. Buckets are cumulative: they include all smaller buckets.

</details>

4. Why is the p95 from `histogram_quantile` only an estimate?

<details>
<summary>Answer</summary>

Prometheus stores only counts per bucket, not each value. It assumes the values are spread evenly inside the bucket where the percentile falls. Bucket limits close to the important values make the estimate better.

</details>

5. Why must labels never contain ids such as `user_id`?

<details>
<summary>Answer</summary>

Each value creates new time series (high cardinality). Millions of series overload the service's memory and Prometheus. Put ids in logs and traces instead.

</details>

## Summary

- Prometheus pulls metrics: every pod shows a text page `/metrics`, and Prometheus scrapes it, here every 15 seconds.
- In Rust, the `metrics` facade with `metrics-exporter-prometheus` gives counters, gauges and histograms; a middleware measures every request.
- Use base units (`_seconds`), `_total` for counters, and route patterns, not raw paths, as labels.
- The Prometheus Helm chart finds pods through the Kubernetes API and the `prometheus.io/*` annotations.
- PromQL: `rate()` for counters, `sum by (...)` to combine pods, a ratio for errors, `histogram_quantile` for percentiles.
- Grafana draws dashboards from Prometheus; provision data sources (and dashboards) from files, and protect the login.

## Sources and further reading

- [Prometheus — Overview](https://prometheus.io/docs/introduction/overview/) and [Metric types](https://prometheus.io/docs/concepts/metric_types/) — how Prometheus works and the four metric types.
- [Prometheus — Querying basics](https://prometheus.io/docs/prometheus/latest/querying/basics/) and [Query functions](https://prometheus.io/docs/prometheus/latest/querying/functions/) — PromQL, `rate` and `histogram_quantile`.
- [Prometheus — Metric and label naming](https://prometheus.io/docs/practices/naming/) and [Histograms and summaries](https://prometheus.io/docs/practices/histograms/) — conventions and why histograms can be aggregated.
- [Prometheus — Exposition formats](https://prometheus.io/docs/instrumenting/exposition_formats/) — the exact text format of `/metrics`.
- [prometheus-community Helm charts](https://github.com/prometheus-community/helm-charts) — the chart used in this lesson and its values.
- [metrics on docs.rs](https://docs.rs/metrics) and [metrics-exporter-prometheus on docs.rs](https://docs.rs/metrics-exporter-prometheus) — the Rust APIs, including upkeep and local recorders.
- [Grafana — Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/) and [Get started with Grafana and Prometheus](https://grafana.com/docs/grafana/latest/getting-started/get-started-grafana-prometheus/) — data sources and dashboards from files, and a first dashboard.

---

[🏠 Start](../START-HERE.md) · [← 21.8 Infrastructure as code with OpenTofu](08-infrastructure-as-code-with-opentofu.md) · [📋 Phase 21](README.md) · [21.10 Logs and traces in a cluster →](10-logs-and-traces-in-a-cluster.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
