<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 21.9 Metrics with Prometheus and Grafana](09-metrics-with-prometheus-and-grafana.md) · [📋 Phase 21](README.md) · [21.11 SRE basics: SLIs, SLOs and error budgets →](11-sre-basics-slis-slos-and-error-budgets.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/21-cloud-native-infrastructure/10-logs-and-traces-in-a-cluster.md) <!-- nav -->

# Lesson 21.10 — Logs and traces in a cluster

> **Phase 21 — Cloud-native infrastructure** · Lesson 10 of 14 · ⏱ about 3 hours (2 sittings)

## What you will learn

- Where the logs of a pod live on a node, what `kubectl logs` really reads, and why those logs disappear.
- How to read logs well: one pod, all pods of a Deployment, the last minutes, the previous crash.
- How to write structured JSON logs from Rust and filter them with `jq`.
- How a node agent (an OpenTelemetry Collector as a DaemonSet) collects the logs of every pod.
- How traces travel from the service to the collector and to Jaeger inside the cluster, labelled with the pod name.

## Before you start

- You finished [Lesson 21.9 — Metrics with Prometheus and Grafana](09-metrics-with-prometheus-and-grafana.md); this lesson extends its `code/p21-orders` service.
- You know `tracing`, spans and `#[instrument]` from [Lesson 09.10 — Logging and tracing](../09-testing-debugging-benchmarking/10-logging-and-tracing.md), and OpenTelemetry, the Collector and Jaeger from [Lesson 16.12](../16-backend-services/12-logs-metrics-and-traces.md). Here we move the same set-up into Kubernetes.
- `jq` is installed (recent macOS has it; check with `jq --version`).

## New words

| Word | Meaning |
|---|---|
| **structured log** | A log line made of named fields (here JSON, JavaScript Object Notation) instead of free text, so that programs can filter and count it. |
| **log rotation** | Closing a log file that has become big, starting a new one and deleting the oldest. The kubelet does it for pod logs. |
| **kubelet** | The Kubernetes agent on every node. It starts the containers of the pods and serves their logs to `kubectl logs`. |
| **node agent** | A program that runs once on every node and does work for all pods there, such as collecting their log files. |
| **DaemonSet** | A Kubernetes object that runs exactly one copy of a pod on every node. The usual way to run a node agent. |
| **hostPath volume** | A volume that mounts a folder of the node itself into a pod. Powerful and risky; used here to read `/var/log/pods`. |
| **Downward API** (API: Application Programming Interface) | A way to give a container facts about its own pod (name, namespace, IP address; IP is the Internet Protocol) as environment variables or files. |
| **resource attributes** | In OpenTelemetry, key-value pairs that describe *who* sends the data: `service.name`, `k8s.pod.name`, `k8s.namespace.name`. |
| **OTLP** (OpenTelemetry Protocol) | The protocol that carries traces, metrics and logs to a collector (Lesson 16.12); here over gRPC on port 4317. |
| **gRPC** (gRPC Remote Procedure Calls) | The remote-call protocol from Lesson 16.9 that OTLP uses here. |
| **CRI** (Container Runtime Interface) | The interface between the kubelet and the container runtime. It also defines the format of the pod log files. |

## 1. Where pod logs go

A container in Kubernetes should write its logs to **standard output** and **standard error**, nothing else. You learned this rule with Docker in Lesson 11.12. In a cluster it works like this:

```text
 container ──stdout/stderr──► container runtime ──► /var/log/pods/<namespace>_<pod>_<uid>/<container>/0.log
                                                         ▲                         (on the node)
 kubectl logs ──► API server ──► kubelet on that node ───┘   reads the file
```

The container runtime (containerd in kind) writes every line into a file on the **node**. The **kubelet** rotates these files when they get big, and serves them when you run `kubectl logs`. A kind node is itself a Docker container, so you can look inside. Start the cluster of this lesson and the service from Section 3 first, or come back here later:

```bash
docker exec course-21-10-control-plane sh -c 'ls /var/log/pods | grep shop'
```

```text
shop_orders-6bcc6596f4-gfttn_9a4b593c-5288-44bf-9eaa-fa2b54221494
shop_orders-6bcc6596f4-zsk76_63d4f153-ffd8-42b6-9f06-d3502340b3c4
```

One folder per pod: namespace, pod name and the pod's unique id. Inside is one file per container. Each line has the CRI (Container Runtime Interface) format: a time, the stream, a flag (`F` = full line) and your text:

```text
2026-11-18T08:24:13.694157632Z stdout F {"timestamp":"2026-11-18T08:24:13.694033Z","level":"INFO","message":"listening","addr":"0.0.0.0:3000","fail_every":20,"target":"p21_orders"}
```

Three consequences matter:

1. When a pod is **deleted**, its folder is deleted too. Its logs are gone for `kubectl logs`.
2. When a node disappears, all logs on it disappear.
3. Rotation keeps only a few megabytes per container. Old lines of a busy service vanish within minutes.

So `kubectl logs` is for looking *now*. To keep logs and search them later, a **node agent** must copy the files somewhere safe (Section 4).

## 2. Reading logs with `kubectl logs`

The commands you will use most:

| Command | What it shows |
|---|---|
| `kubectl logs <pod>` | All current lines of one pod (its only container) |
| `kubectl logs <pod> -c <container>` | One container of a pod with several |
| `kubectl logs deployment/orders` | One pod of the Deployment, chosen by kubectl |
| `kubectl logs -l app=orders --prefix` | All pods with this label, each line prefixed with its pod |
| `kubectl logs ... --tail=20` / `--since=5m` | Only the last 20 lines / the last 5 minutes |
| `kubectl logs ... -f` | Keep following new lines, like `tail -f` |
| `kubectl logs <pod> --previous` | The logs of the **previous** container, the one that crashed |

`--previous` is the life-saver after a crash: Kubernetes restarts the container, and the new one has fresh, empty logs. The reason for the crash is in the old one (in Lesson 21.5 it showed that a restarted program was healthy and the probe was wrong).

With several pods, always look at **all** of them. `kubectl logs deployment/orders` picks one pod and says so:

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

These lines are already **structured**: JSON objects. The next section shows how the service writes them.

## 3. Structured logs and traces from the service

In Lesson 16.12, logs were text lines for people. In a cluster, logs are read mostly by *programs* (collectors, search tools), so each line should be one JSON object with named fields. The `tracing-subscriber` crate does that with its `json` feature. We also send the spans as traces, as in Lesson 16.12, but only traces: metrics already go to Prometheus.

From `code/p21-orders`, add the crates:

```bash
cargo add opentelemetry@0.33 opentelemetry_sdk@0.33
cargo add opentelemetry-otlp@0.33 --no-default-features --features grpc-tonic,trace
cargo add tracing@0.1 tracing-opentelemetry@0.34
cargo add tracing-subscriber@0.3 --features env-filter,json
cargo add tower-http@0.7 --features trace
```

`src/telemetry.rs`, the trace half of the Lesson 16.12 module, with JSON output:

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

**What happens:**

1. `.json()` turns every event into one JSON line. `flatten_event(true)` puts the event's fields (`message`, `order_id`) at the top level instead of inside a nested `fields` object, which makes filtering easier. `with_span_list(false)` leaves out the list of all parent spans and keeps only the current one in `span`, so the lines stay shorter.
2. `Resource::builder()` reads the environment variable `OTEL_RESOURCE_ATTRIBUTES` (if it is set) in addition to the service name. We will use it to attach the pod name to every span.
3. The exporter reads its address from `OTEL_EXPORTER_OTLP_ENDPOINT`, as in Lesson 16.12. In the cluster, the Deployment sets it.

In `src/main.rs`, add `mod telemetry;` at the top and these imports:

```rust
use tower_http::trace::{DefaultMakeSpan, DefaultOnResponse, TraceLayer};
use tracing::{Level, info, instrument, warn};
```

Then change `main`, `app` and `get_order`, and add `load_order`. `AppState`, `shutdown_signal`, `track_http` and `render_metrics` stay exactly as in Lesson 21.9.

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

**What happens:**

1. `info!(addr = %..., fail_every, "listening")` replaces the `println!`. Every message is now a structured event; `%` records the address with its `Display` form.
2. `TraceLayer` opens a `request` span for every HTTP (HyperText Transfer Protocol) request and logs "finished processing request" with status and latency (Lesson 16.12).
3. `#[instrument(skip(state))]` makes a `get_order` span with the order id as a field, but skips the state, which is big and not interesting. `load_order` gets its own child span, so the trace shows the "database" time separately.
4. The `warn!` and `info!` events carry `order_id` as a field, so you can search for one order later.
5. After `SIGTERM`, `main` calls `tracer_provider.shutdown()`, which sends the spans that are still waiting in the batch. That is one more reason to handle `SIGTERM`: a pod that is killed loses its last traces.

Set `version = "0.2.0"` in `Cargo.toml` and run `cargo test` (it also updates `Cargo.lock`, Lesson 21.4). Then build the image `course-p21-orders:0.2.0` with the Dockerfile from Lesson 21.9, create the cluster and load the image:

```bash
docker build -t course-p21-orders:0.2.0 .
kind create cluster --name course-21-10
kind load docker-image course-p21-orders:0.2.0 --name course-21-10
```

The Deployment in `k8s/orders.yaml` is the one from Lesson 21.9 with the new image and more environment variables (the Prometheus annotations are left out to keep it short; they do no harm):

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

**What happens:**

1. `OTEL_EXPORTER_OTLP_ENDPOINT` points to a collector Service in the namespace `observability`, which we create in the next section.
2. `valueFrom.fieldRef` is the **Downward API**: Kubernetes fills `POD_NAME` with the pod's own name and `POD_NAMESPACE` with its namespace.
3. `$(POD_NAME)` inside another value is replaced by Kubernetes with that variable. So every pod sends spans with its own `k8s.pod.name`, and in Jaeger you can see *which* pod handled a request.

Apply it only after the collector exists (Section 4), otherwise the first spans fail to export (the service keeps working).

## 4. A collector on every node

The collector of Lesson 16.12 received traces. In a cluster it can also be the **node agent** for logs: it reads the files in `/var/log/pods` and sends every line to a log store. One collector must run on **every** node, because each node has only its own files. That is the job of a **DaemonSet**: Kubernetes runs exactly one copy of its pod on every node, and on new nodes automatically.

`k8s/observability.yaml` creates the namespace and Jaeger (the same image as in Lesson 16.12):

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

`k8s/collector.yaml` holds the collector's configuration, the DaemonSet and a Service:

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

**What happens:**

1. The `file_log` receiver reads the log files of all pods in the namespace `shop` (`include` is a file pattern). `start_at: end` reads only new lines, not the old ones. `include_file_path: true` records the file path with every line.
2. The `container` operator understands the CRI line format: it removes the time and the stream prefix, and it takes the namespace, pod name and container name **from the file path** and stores them as resource attributes.
3. The `logs` pipeline sends the lines to the `debug` exporter, which prints them into the collector's own log. In a real cluster you would export to a log store instead (for example Grafana Loki or Elasticsearch, through their OTLP receivers); the receiving side would be the only change.
4. The `traces` pipeline is the one from Lesson 16.12: OTLP in, Jaeger out.
5. The **hostPath volume** mounts the node's folder `/var/log/pods` into the pod, read-only. The files belong to `root` and are readable only by it (`-rw-r-----`), so the collector runs as user 0.

> 🔒 **Security:** A pod with a hostPath volume and `runAsUser: 0` can read files of every pod on the node, including their logs. Give this power only to a trusted agent, mount read-only, mount only the folder it needs, and keep it in its own namespace. This is also one more reason never to log passwords and tokens: everybody with access to the log pipeline can read them.

Apply everything and send some traffic:

```bash
kubectl apply -f k8s/observability.yaml -f k8s/collector.yaml
kubectl -n observability rollout status daemonset/otel-collector
kubectl apply -f k8s/orders.yaml
kubectl -n shop rollout status deployment/orders
kubectl -n shop run load --rm -i --restart=Never --image=busybox:1.37 -- \
  sh -c 'for i in $(seq 1 60); do wget -q -O /dev/null http://orders/orders/$i 2>/dev/null; done; true'
kubectl -n observability logs daemonset/otel-collector | grep -E '^ResourceLog|order sent' | head -3
```

Shortened output (each log line is followed by more attributes, which we cut; a `ResourceLog` line starts the lines of one pod and lists its resource attributes):

```text
{"timestamp":"2026-11-18T08:24:49.013154Z","level":"INFO","message":"order sent","order_id":1,"target":"p21_orders","span":{"id":"1","name":"get_order"}} logtag=F log.file.path=/var/log/pods/shop_orders-6bcc6596f4-gfttn_9a4b593c-5288-44bf-9eaa-fa2b54221494/ord...
{"timestamp":"2026-11-18T08:24:49.046520Z","level":"INFO","message":"order sent","order_id":2,"target":"p21_orders","span":{"id":"2","name":"get_order"}} logtag=F log.file.path=/var/log/pods/shop_orders-6bcc6596f4-zsk76_63d4f153-ffd8-42b6-9f06-d3502340b3c4/ord...
ResourceLog #1 k8s.container.name=orders k8s.container.restart_count=0 k8s.namespace.name=shop k8s.pod.name=orders-6bcc6596f4-gfttn k8s.pod.uid=9a4b593c-5288-44bf-9eaa-fa2b54221494
```

The collector read the lines from the node and knows which pod they came from. Now delete a pod and compare:

```bash
kubectl -n shop delete pod orders-6bcc6596f4-gfttn
kubectl -n shop logs orders-6bcc6596f4-gfttn
```

```text
pod "orders-6bcc6596f4-gfttn" deleted from shop namespace
error: error from server (NotFound): pods "orders-6bcc6596f4-gfttn" not found in namespace "shop"
```

`kubectl logs` has nothing any more, but the collector had already copied the lines. That is the whole point of log collection.

> 💡 **Tip:** On a big cluster, the service sends its traces to the collector on **its own node** (through the node's IP address and a `hostPort`), not through a Service that may pick a collector on another node. On one kind node there is no difference, so we keep the simple Service.

## 5. Looking at traces in Jaeger

Open Jaeger through a port-forward:

```bash
kubectl -n observability port-forward service/jaeger 16686:16686
```

In <http://localhost:16686>, choose the service `p21-orders`, the operation `get_order`, and click **Find Traces**. Open one trace. Each trace has three spans, `request` → `get_order` → `load_order`, and on the trace page the **Process** section lists the resource attributes, among them `k8s.pod.name` and `k8s.namespace.name`.

Jaeger also has an HTTP API. A compact summary of two order traces (trace id, span, milliseconds, pod) looks like this:

```text
0c1bc130	get_order	31	orders-6bcc6596f4-gfttn
0c1bc130	load_order	31	orders-6bcc6596f4-gfttn
0c1bc130	request	31	orders-6bcc6596f4-gfttn
13adf1a3	get_order	31	orders-6bcc6596f4-zsk76
13adf1a3	load_order	31	orders-6bcc6596f4-zsk76
13adf1a3	request	31	orders-6bcc6596f4-zsk76
```

Two traces, two different pods: the Service spread the requests, and every span knows where it ran. Requests that returned 500 are marked as errors: `TraceLayer` logs "response failed" at `ERROR` level, and `tracing-opentelemetry` then sets the span status to *error*. In the Jaeger search form, write `error=true` into **Tags** to find only failed requests.

The three signals now point to each other. A metric in Grafana shows the error rate going up (Lesson 21.9); Jaeger shows failed traces and which pod and which span they failed in; the logs of that pod, filtered by `order_id`, tell you what exactly happened.

> ⚡ **Performance:** Every request becomes a trace, including the health checks every 10 seconds from every pod. In a busy service you send only a *sample* of traces (for example 10%), set with `OTEL_TRACES_SAMPLER=traceidratio` and `OTEL_TRACES_SAMPLER_ARG=0.1`, and you do not trace the probes at all (Exercise 3).

## Common mistakes

> ⚠️ **Common mistake:** `--previous` on a container that never crashed:

```text
Error from server (BadRequest): previous terminated container "orders" in pod "orders-6bcc6596f4-gfttn" not found
```

There is no previous container yet; read the current logs instead. After a crash, `--previous` is the first thing to run.

> ⚠️ **Common mistake:** Using the Deployment's name as a pod name: `kubectl -n shop logs orders` fails with `pods "orders" not found`. Write `deployment/orders`, or use `-l app=orders --prefix` to see all pods.

> ⚠️ **Common mistake:** Forgetting `include_file_path: true` in the `file_log` receiver. The `container` operator needs the path to know the pod, and every line fails:

```text
error	helper/transformer.go:160	Failed to process entry	{... "error": "operator 'container' has 'add_metadata_from_filepath' enabled, but the log record attribute 'log.file.path' is missing. Perhaps enable the 'include_file_path' option?" ...}
```

The receiver used to be called `filelog`; collector 0.165.0 still accepts it but warns `"filelog" alias is deprecated; use "file_log" instead`.

> ⚠️ **Common mistake:** Writing logs into a file inside the container. Kubernetes never sees them, they disappear with the container, and they can fill its disk. Write to standard output only.

## Try it yourself

1. Follow the logs of all pods while you send requests: `kubectl -n shop logs -l app=orders --prefix -f`, and in a second terminal run the load command from Section 4. Stop with Ctrl+C.
2. Count the log lines of the last five minutes: `kubectl -n shop logs -l app=orders --since=5m --tail=-1 | wc -l`. (`--tail=-1` means "no limit"; with `-l`, kubectl shows only 10 lines per pod by default.)
3. Look at the pod log folder on the node again after the pod deletion above: the deleted pod's folder is gone, and the new pod has a new folder.
4. When you are done: `kind delete cluster --name course-21-10`.

## Exercises

### Exercise 1 — Find the failures (easy)

Using only `kubectl` and `jq`, print the time and the URI (Uniform Resource Identifier, here the request path) of every request that ended with status 500, from **all** order pods, as compact JSON. One line per failed request.

<details>
<summary>Hint</summary>

The "finished processing request" lines have a numeric field `status` and the URI in `.span.uri`. `jq -c 'select(...) | {...}'` filters and reshapes.

</details>

<details>
<summary>Answer</summary>

```bash
kubectl -n shop logs -l app=orders --tail=-1 | jq -c 'select(.status == 500) | {timestamp, uri: .span.uri}'
```

```text
{"timestamp":"2026-11-18T08:25:54.613294Z","uri":"/orders/8"}
{"timestamp":"2026-11-18T08:25:54.776897Z","uri":"/orders/13"}
{"timestamp":"2026-11-18T08:25:55.771095Z","uri":"/orders/43"}
```

Why it works: `-l app=orders --tail=-1` reads all lines of all pods; because every line is JSON, `jq` can test a field instead of searching text. Lines without `status` give `null`, which is not `500`, so they are skipped.

</details>

### Exercise 2 — Test the JSON log line (medium)

**Write a test** that sends `GET /orders/500` to the router and checks that the logs contain a JSON line with `"message":"order not found"`, `"level":"WARN"` and `"order_id":500`. The test must not print to the terminal and must not change the global subscriber.

<details>
<summary>Hint</summary>

Build a subscriber with `tracing_subscriber::fmt().json().flatten_event(true).with_writer(...).finish()` and activate it for the current thread with `tracing::subscriber::set_default(subscriber)` (keep the returned guard). The writer must implement `MakeWriter`; a small type around `Arc<Mutex<Vec<u8>>>` that implements `std::io::Write` and `MakeWriter` is enough. Use a `current_thread` test runtime, as in Lesson 21.9.

</details>

<details>
<summary>Answer</summary>

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

Why it works: `set_default` makes the subscriber active only on this thread while the guard lives, and the current-thread runtime runs the whole router there. The writer stores the bytes, so the test reads the real JSON lines. The `metrics` calls go nowhere, because no recorder is installed, which is harmless. (Keep the dev-dependencies `tower` and `http-body-util` from Lesson 21.9.)

</details>

### Exercise 3 — Do not trace the health checks (hard)

Every probe request produces a log line and a trace. Change `fn app` so that `/healthz` still answers `ok` but produces **no** "finished processing request" line and no trace. Check it locally with `cargo run`, one `curl localhost:3000/healthz` and one `curl localhost:3000/orders/1`. Then say what else changed for `/healthz`.

<details>
<summary>Hint</summary>

In axum, `.layer(...)` and `.route_layer(...)` wrap only the routes that were added **before** them.

</details>

<details>
<summary>Answer</summary>

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

Shortened log of the run (lines cut at 120 characters):

```text
{"timestamp":"2026-11-18T08:27:37.927158Z","level":"INFO","message":"listening","addr":"0.0.0.0:3000","fail_every":0,"ta
{"timestamp":"2026-11-18T08:27:39.306927Z","level":"INFO","message":"order sent","order_id":1,"target":"p21_orders","spa
{"timestamp":"2026-11-18T08:27:39.307080Z","level":"INFO","message":"finished processing request","latency":"32 ms","sta
```

Why it works: the route `/healthz` is added after both layers, so neither `TraceLayer` nor `track_http` wraps it. Only the order request produced a "finished processing request" line. What else changed: `/healthz` is no longer counted in `http_requests_total` either, which is usually what you want, because probes are not real users.

</details>

## Quiz

1. Where does `kubectl logs` get its data from, and why are those logs lost when a pod is deleted?

<details>
<summary>Answer</summary>

From the kubelet of the pod's node, which reads the log files under `/var/log/pods/<namespace>_<pod>_<uid>/` on that node. The folder is removed together with the pod, and rotation removes old lines even earlier.

</details>

2. Your pod restarted three times. Which command shows why it crashed last time?

<details>
<summary>Answer</summary>

`kubectl logs <pod> --previous`. The current container has new, empty logs; the previous one has the error.

</details>

3. Why is a log collector deployed as a DaemonSet and not as a Deployment?

<details>
<summary>Answer</summary>

Each node has only the log files of its own pods. A DaemonSet runs exactly one collector on every node, including new nodes, so every file is read.

</details>

4. What does the Downward API give a container, and how did we use it?

<details>
<summary>Answer</summary>

Facts about its own pod, such as its name and namespace. We put them into `OTEL_RESOURCE_ATTRIBUTES`, so every span carries `k8s.pod.name` and `k8s.namespace.name`.

</details>

5. Why are JSON logs better than text lines in a cluster?

<details>
<summary>Answer</summary>

Programs (collectors, log stores, `jq`) can filter and count by field, for example `status == 500` or `order_id == 7`, without fragile text searches.

</details>

## Summary

- Containers log to standard output; the runtime writes the lines into files under `/var/log/pods` on the node, and the kubelet rotates them and serves `kubectl logs`.
- Use `-l app=... --prefix` to see all pods, `--since` and `--tail` to limit, `-f` to follow, and `--previous` after a crash.
- Write structured JSON logs (`tracing-subscriber` with `json`), with ids as fields, never secrets.
- A node agent, here the OpenTelemetry Collector as a DaemonSet with a read-only hostPath, keeps the logs after pods are gone.
- Traces go from the service over OTLP to the collector and on to Jaeger; the Downward API adds the pod name to every span.
- Metrics, traces and logs together answer *that*, *where* and *what* went wrong.

## Sources and further reading

- [Kubernetes — Logging architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/) — where container logs go, rotation, and node-level agents.
- [kubectl logs reference](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_logs/) — every option of `kubectl logs`.
- [Kubernetes — DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/) and [Downward API](https://kubernetes.io/docs/concepts/workloads/pods/downward-api/) — the two Kubernetes features used by the collector and the service.
- [OpenTelemetry — Important components for Kubernetes](https://opentelemetry.io/docs/platforms/kubernetes/collector/components/) — collector receivers and processors for clusters.
- [The filelog receiver](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver/filelogreceiver) — options of the log file receiver and the `container` operator.
- [OpenTelemetry — SDK configuration](https://opentelemetry.io/docs/languages/sdk-configuration/general/) — `OTEL_RESOURCE_ATTRIBUTES`, `OTEL_TRACES_SAMPLER` and other variables.
- [tracing-subscriber — the JSON format](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/fmt/format/struct.Json.html) — the options used for JSON logs.

---

[🏠 Start](../START-HERE.md) · [← 21.9 Metrics with Prometheus and Grafana](09-metrics-with-prometheus-and-grafana.md) · [📋 Phase 21](README.md) · [21.11 SRE basics: SLIs, SLOs and error budgets →](11-sre-basics-slis-slos-and-error-budgets.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
