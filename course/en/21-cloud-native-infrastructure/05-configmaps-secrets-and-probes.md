<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 21.4 Deploying a Rust service to Kubernetes](04-deploying-a-rust-service-to-kubernetes.md) · [📋 Phase 21](README.md) · [21.6 Packaging with Helm and Kustomize →](06-helm-and-kustomize.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/21-cloud-native-infrastructure/05-configmaps-secrets-and-probes.md) <!-- nav -->

# Lesson 21.5 — ConfigMaps, Secrets and health probes

> **Phase 21 — Cloud-native infrastructure** · Lesson 5 of 14 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How to keep configuration out of the image with a ConfigMap, and pass it to your service as environment variables.
- How to give your service a secret with a Secret mounted as a file, and why a Secret is *not* encrypted by default.
- The three kinds of probes (readiness, liveness, startup), what Kubernetes does when each one fails, and which endpoint each should call.
- How to change configuration safely: why running pods do not see a changed ConfigMap, and how `kubectl rollout restart` fixes that.
- How a wrong liveness probe makes Kubernetes restart a healthy program.

## Before you start

- You finished [Lesson 21.4 — Deploying a Rust service to Kubernetes](04-deploying-a-rust-service-to-kubernetes.md) and have its `code/p21-hello` folder with `k8s/`.
- You know configuration from the environment and the difference between "alive" and "ready" ([Lesson 16.13 — Configuration, health checks and containers](../16-backend-services/13-configuration-health-checks-and-containers.md)), and bearer tokens ([Lesson 16.8 — Authentication in a web service](../16-backend-services/08-authentication-in-a-web-service.md)).
- You know atomics such as `AtomicBool` ([Lesson 14.5 — Atomics](../14-concurrency-and-async/05-atomics.md)).

## New words

| Word | Meaning |
|---|---|
| **ConfigMap** | A Kubernetes object that holds non-secret configuration as key-value pairs. Pods can read it as environment variables or as files. |
| **Secret** | A Kubernetes object for small secret values (passwords, tokens, keys). Like a ConfigMap, but handled more carefully: not shown by default and can be encrypted in etcd. |
| **base64** | A way to write any bytes with 64 printable characters. It is an *encoding*, not encryption: anyone can decode it. |
| **probe** | A check that the kubelet runs regularly against a container: an HTTP (HyperText Transfer Protocol) request, a TCP (Transmission Control Protocol) connection or a command. |
| **readiness probe** | "Can this pod take traffic now?" If it fails, the pod is removed from the Service's endpoints, but it keeps running. |
| **liveness probe** | "Is this program still working at all?" If it fails several times, the kubelet kills and restarts the container. |
| **startup probe** | "Has the program finished starting?" While it has not succeeded, the other two probes wait. For slow starts. |
| **restart count** | How many times the kubelet restarted a container of a pod; the `RESTARTS` column of `kubectl get pods`. |
| **`rollout restart`** | A kubectl command that replaces all pods of a Deployment with new ones (a rolling update without a code change). |
| **fail fast** | Stopping at once with a clear error when something essential is missing, instead of running in a broken state. |

## 1. Why configuration lives outside the image

In Lesson 21.4 the image was the whole program. But the same program needs different settings in different places: a test cluster and production use different greetings, timeouts, database addresses and passwords. If the settings are inside the image, you need one image per environment, and the image you tested is not the image you run.

The cloud-native rule (Lesson 21.1, and the Twelve-Factor App): **build one image, configure it from outside**. Kubernetes has two objects for this:

| | ConfigMap | Secret |
|---|---|---|
| For | Ordinary settings: greeting, log level, feature switches | Passwords, tokens, keys, certificates |
| Shown by `kubectl get -o yaml` | Plain text | base64 (encoded, not encrypted) |
| Stored in etcd | Plain | Plain by default; encryption at rest can be turned on |
| Read by the pod as | Environment variables or files | Environment variables or files (files are safer) |
| Max size | 1 MiB (mebibyte) | 1 MiB |

## 2. Version 0.3.0 of the service

The new version reads three more settings and offers two more endpoints:

- `GREETING` (from a ConfigMap): the first word of the answer, default `hello`.
- `STARTUP_DELAY_MS` (from a ConfigMap): a simulated warm-up, for example loading a cache. Until it ends, the service is alive but **not ready**.
- `ADMIN_TOKEN_FILE`: the path of a file with a secret token (from a Secret). `GET /admin` answers only to callers who send that token.
- `GET /readyz`: `200 ready` after the warm-up, `503 warming up` before.

In `code/p21-hello`, set `version = "0.3.0"` in `Cargo.toml`, add the `time` feature of Tokio (`cargo add tokio@1 --features macros,rt-multi-thread,net,signal,time`), and replace `src/main.rs` with:

```rust
use std::sync::Arc;
use std::sync::atomic::{AtomicBool, Ordering};
use std::time::Duration;

use axum::http::{HeaderMap, StatusCode, header::AUTHORIZATION};
use axum::{Router, extract::State, routing::get};
use tokio::net::TcpListener;
use tokio::signal::unix::{SignalKind, signal};

/// Data every request handler can read.
#[derive(Clone)]
struct AppState {
    pod: Arc<str>,
    greeting: Arc<str>,
    admin_token: Arc<str>,
    ready: Arc<AtomicBool>,
}

fn app(state: AppState) -> Router {
    Router::new()
        .route("/", get(hello))
        .route("/healthz", get(|| async { "ok\n" }))
        .route("/readyz", get(readyz))
        .route("/admin", get(admin))
        .with_state(state)
}

async fn hello(State(state): State<AppState>) -> String {
    format!(
        "{} from {} (version {})\n",
        state.greeting,
        state.pod,
        env!("CARGO_PKG_VERSION")
    )
}

/// 200 when the service can take traffic, 503 while it is still warming up.
async fn readyz(State(state): State<AppState>) -> (StatusCode, &'static str) {
    if state.ready.load(Ordering::Relaxed) {
        (StatusCode::OK, "ready\n")
    } else {
        (StatusCode::SERVICE_UNAVAILABLE, "warming up\n")
    }
}

/// Only callers that send `Authorization: Bearer <token>` get in.
async fn admin(State(state): State<AppState>, headers: HeaderMap) -> (StatusCode, &'static str) {
    let expected = format!("Bearer {}", state.admin_token);
    match headers.get(AUTHORIZATION) {
        Some(value) if value.as_bytes() == expected.as_bytes() => {
            (StatusCode::OK, "welcome, admin\n")
        }
        _ => (StatusCode::UNAUTHORIZED, "missing or wrong token\n"),
    }
}

/// Reads an environment variable, or uses a default when it is not set.
fn env_or(name: &str, default: &str) -> String {
    std::env::var(name).unwrap_or_else(|_| default.to_string())
}

/// Finishes when the process gets SIGTERM (from Kubernetes) or Ctrl+C (from you).
async fn shutdown_signal() {
    let mut term = signal(SignalKind::terminate()).expect("cannot listen for SIGTERM");
    tokio::select! {
        _ = term.recv() => println!("received SIGTERM, shutting down"),
        _ = tokio::signal::ctrl_c() => println!("received Ctrl+C, shutting down"),
    }
}

#[tokio::main]
async fn main() {
    let port = env_or("PORT", "8080");
    let token_file = env_or("ADMIN_TOKEN_FILE", "/etc/hello/admin-token");
    let delay_ms: u64 = env_or("STARTUP_DELAY_MS", "0")
        .parse()
        .expect("STARTUP_DELAY_MS must be a whole number");

    // Fail fast: a service without its secret must not start at all.
    let admin_token = match std::fs::read_to_string(&token_file) {
        Ok(text) => text.trim().to_string(),
        Err(err) => {
            eprintln!("cannot read the admin token from {token_file}: {err}");
            std::process::exit(1);
        }
    };

    let state = AppState {
        pod: env_or("HOSTNAME", "localhost").into(),
        greeting: env_or("GREETING", "hello").into(),
        admin_token: admin_token.into(),
        ready: Arc::new(AtomicBool::new(false)),
    };

    // Simulated warm-up (loading a cache, for example). Not ready until it ends.
    let ready = state.ready.clone();
    tokio::spawn(async move {
        tokio::time::sleep(Duration::from_millis(delay_ms)).await;
        ready.store(true, Ordering::Relaxed);
        println!("warm-up finished, ready for traffic");
    });

    let listener = TcpListener::bind(format!("0.0.0.0:{port}"))
        .await
        .expect("cannot bind the port");
    println!(
        "p21-hello {} listening on port {port}",
        env!("CARGO_PKG_VERSION")
    );

    axum::serve(listener, app(state))
        .with_graceful_shutdown(shutdown_signal())
        .await
        .expect("server error");
    println!("all connections closed, bye");
}

#[cfg(test)]
mod tests {
    use super::*;
    use axum::body::Body;
    use axum::http::Request;
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    fn test_state() -> AppState {
        AppState {
            pod: "test-pod".into(),
            greeting: "salam".into(),
            admin_token: "t0ken".into(),
            ready: Arc::new(AtomicBool::new(true)),
        }
    }

    async fn send(state: AppState, request: Request<Body>) -> (StatusCode, String) {
        let response = app(state).oneshot(request).await.unwrap();
        let status = response.status();
        let bytes = response.into_body().collect().await.unwrap().to_bytes();
        (status, String::from_utf8(bytes.to_vec()).unwrap())
    }

    fn get(uri: &str) -> Request<Body> {
        Request::get(uri).body(Body::empty()).unwrap()
    }

    #[tokio::test]
    async fn hello_uses_the_configured_greeting() {
        let (_, body) = send(test_state(), get("/")).await;
        assert!(body.starts_with("salam from test-pod"), "body was {body:?}");
    }

    #[tokio::test]
    async fn admin_without_token_is_unauthorized() {
        let (status, _) = send(test_state(), get("/admin")).await;
        assert_eq!(status, StatusCode::UNAUTHORIZED);
    }

    #[tokio::test]
    async fn admin_with_the_right_token_is_allowed() {
        let request = Request::get("/admin")
            .header(AUTHORIZATION, "Bearer t0ken")
            .body(Body::empty())
            .unwrap();
        let (status, body) = send(test_state(), request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, "welcome, admin\n");
    }
}
```

**What happens:**

1. `AppState` gains three fields. `ready` is an `Arc<AtomicBool>`: shared between the warm-up task and every request handler, read and written without a lock. `Ordering::Relaxed` is enough because it is a single flag that guards no other data (Lesson 14.6).
2. `readyz` returns a pair `(StatusCode, &str)`; axum turns it into a response with that status. Kubernetes looks only at the status: 200–399 means success, anything else is a failure.
3. `admin` reads the `Authorization` header and compares it with `Bearer <token>`. Missing or different gives `401 Unauthorized`.
4. `main` reads the token **before** it opens the port. If the file is missing, the program prints why and exits with code 1. This is **fail fast**: a pod that cannot start shows a clear error in its logs, instead of running and failing every admin request later.
5. A spawned task sleeps for `STARTUP_DELAY_MS` and then sets `ready`. The port is open during the warm-up, so `/healthz` already answers: the program is *alive* but not *ready*. This difference is exactly what the two probes measure.
6. The tests build an `AppState` by hand, with a fake token, so they need no files and no environment.

> 🔒 **Security:** the `==` comparison of the token returns as soon as one byte differs, so its time leaks a little information about how many bytes were right. For a real service, compare secrets in constant time (for example with the `subtle` crate) and prefer real authentication (Lesson 16.8). Also note that `text.trim()` removes the newline that most tools write at the end of a file; forgetting this is a classic "the token is right but it does not work" bug.

Try it on your Mac. Create a token file for development and keep it out of Git (add `dev-token.txt` and `admin-token.txt` to `.gitignore`):

```bash
printf 'dev-token\n' > dev-token.txt
cargo test
ADMIN_TOKEN_FILE=dev-token.txt PORT=8090 STARTUP_DELAY_MS=2000 cargo run
```

```text
running 3 tests
test tests::hello_uses_the_configured_greeting ... ok
test tests::admin_with_the_right_token_is_allowed ... ok
test tests::admin_without_token_is_unauthorized ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```text
p21-hello 0.3.0 listening on port 8090
warm-up finished, ready for traffic
```

In a second tab, `curl -s -i http://localhost:8090/readyz` during the first two seconds prints the status line `HTTP/1.1 503 Service Unavailable` and `warming up`; after them, `HTTP/1.1 200 OK` and `ready`. `curl -s -H 'Authorization: Bearer dev-token' http://localhost:8090/admin` prints `welcome, admin`. Stop the service with Ctrl+C. Without `ADMIN_TOKEN_FILE`, it fails fast:

```text
cannot read the admin token from /etc/hello/admin-token: No such file or directory (os error 2)
```

## 3. A ConfigMap

`k8s/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-config
data:
  GREETING: "salam"
  STARTUP_DELAY_MS: "3000"
```

Values in a ConfigMap are always strings, so numbers are quoted. The Deployment in Section 5 reads all keys as environment variables with `envFrom: - configMapRef: name: hello-config`. There are two other ways, which you will meet in other projects: one variable from one key (`env: - name: X valueFrom: configMapKeyRef: ...`), and the ConfigMap mounted as a folder, with one file per key (good for whole configuration files such as `config.toml`).

## 4. A Secret

Create a random token and a Secret from it. Do not write the token into a YAML (YAML Ain't Markup Language) file in Git; create the Secret directly from the file:

```bash
openssl rand -hex 16 > admin-token.txt
kubectl create secret generic hello-admin --from-file=admin-token=admin-token.txt
```

(You will create the cluster first, in Section 6; the commands are shown here to explain them.) `generic` is the ordinary kind of Secret; `--from-file=admin-token=admin-token.txt` makes one key, `admin-token`, whose value is the content of the file. Now look at what Kubernetes stores:

```bash
kubectl get secret hello-admin -o yaml
```

```text
apiVersion: v1
data:
  admin-token: Y2MxMjlhYTRkYTRiNTY0YzNjYjE0Y2IzZWE3YjQ1Y2MK
kind: Secret
metadata:
  creationTimestamp: "2026-11-18T08:43:47Z"
  name: hello-admin
  namespace: default
  resourceVersion: "536"
  uid: f2487b1e-47fe-430b-8bf2-8f968624281d
type: Opaque
```

The value looks unreadable, but it is only **base64**. Anyone who may read the Secret can decode it:

```bash
kubectl get secret hello-admin -o jsonpath='{.data.admin-token}' | base64 -d
```

```text
cc129aa4da4b564c3cb14cb3ea7b45cc
```

That is exactly the content of `admin-token.txt` (including its final newline). So the protection of a Secret comes from **who may read it** (access rules in the API, Application Programming Interface, server; the least privilege of Phase 10), from encryption of etcd if the cluster administrator turns it on, and from keeping it out of Git — not from base64.

The Deployment in Section 5 mounts the Secret as a **read-only folder** (`volumes` and `volumeMounts`): each key becomes a file. The pod then has the file `/etc/hello/admin-token`, and `ADMIN_TOKEN_FILE` points to it. A file is safer than an environment variable: environment variables are easy to leak (a crash report that prints the environment, a child process that inherits them, `kubectl describe pod` printing every variable set with `value:`), and the kubelet can update mounted files when the Secret changes.

> 🔒 **Security:** never commit a Secret manifest with real values, not even "only base64". Real teams keep secrets in a secrets manager or commit them only in encrypted form; tools such as Sealed Secrets, SOPS (Secrets OPerationS) and External Secrets Operator exist for this. For this course, `kubectl create secret` from a file that is in `.gitignore` is enough.

## 5. Probes: is it alive, is it ready?

The kubelet on each node can check each container regularly. Three kinds of checks exist, and they answer different questions:

| Probe | Question | When it fails | Our endpoint |
|---|---|---|---|
| **readiness** | Can it take traffic now? | Removed from the Service's endpoints; keeps running; added back when it passes again | `/readyz` |
| **liveness** | Is it working at all, or stuck? | After `failureThreshold` failures: container killed and restarted | `/healthz` |
| **startup** | Has it finished starting? | Until it succeeds, readiness and liveness are not run; if it never succeeds, the container is restarted | (not needed here) |

The full `k8s/deployment.yaml` of this lesson:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  labels:
    app: hello
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: course-p21-hello:0.3.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - configMapRef:
                name: hello-config
          env:
            - name: ADMIN_TOKEN_FILE
              value: /etc/hello/admin-token
          volumeMounts:
            - name: admin-secret
              mountPath: /etc/hello
              readOnly: true
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
            periodSeconds: 2
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            periodSeconds: 10
            failureThreshold: 3
          resources:
            requests:
              cpu: 50m
              memory: 16Mi
            limits:
              memory: 64Mi
          securityContext:
            runAsNonRoot: true
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
      volumes:
        - name: admin-secret
          secret:
            secretName: hello-admin
```

**What happens:**

1. `envFrom` turns every key of `hello-config` into an environment variable; `env` adds one more, `ADMIN_TOKEN_FILE`.
2. `volumes` declares a volume of the pod made from the Secret; `volumeMounts` puts it at `/etc/hello` in the container. Because the root file system is read-only (Lesson 21.4), this mounted folder is the only way to give the program files.
3. `readinessProbe` sends `GET /readyz` to the port named `http` every 2 seconds. Until it gets a 2xx or 3xx answer, the pod is `0/1` ready and receives no traffic from the Service.
4. `livenessProbe` sends `GET /healthz` every 10 seconds; after 3 failures in a row (`failureThreshold`) the kubelet restarts the container. Other useful fields: `initialDelaySeconds` (wait before the first check) and `timeoutSeconds` (default 1 second).
5. The image tag is `0.3.0`; everything else is unchanged from Lesson 21.4.

> ⚠️ **Common mistake:** a liveness probe that checks dependencies, such as "can I reach the database?". When the database is down, *every* pod fails liveness and Kubernetes restarts all of them in a loop, which does not repair the database and adds load when it comes back. Liveness should answer only "is this process stuck?". Dependencies belong in readiness, and even there carefully (if the whole service is not ready, it receives no traffic at all).

## 6. Deploy and observe

Build, create the cluster, load the image, create the Secret, apply:

```bash
cargo test
docker build -t course-p21-hello:0.3.0 .
kind create cluster --name course-21-05
kubectl wait --for=condition=Ready node --all --timeout=120s
kind load docker-image course-p21-hello:0.3.0 --name course-21-05
openssl rand -hex 16 > admin-token.txt
kubectl create secret generic hello-admin --from-file=admin-token=admin-token.txt
kubectl apply -f k8s/
```

```text
configmap/hello-config created
deployment.apps/hello created
service/hello created
```

(The output of the earlier commands is as in Lesson 21.4 and Section 4.) Look at the pods at once, and again after the rollout:

```bash
kubectl get pods -l app=hello
kubectl rollout status deployment/hello
kubectl get pods -l app=hello
```

```text
NAME                    READY   STATUS    RESTARTS   AGE
hello-c6cf94684-b5hp4   0/1     Running   0          2s
hello-c6cf94684-fshn6   0/1     Running   0          2s
hello-c6cf94684-xgd48   0/1     Running   0          2s
Waiting for deployment "hello" rollout to finish: 0 of 3 updated replicas are available...
Waiting for deployment "hello" rollout to finish: 1 of 3 updated replicas are available...
Waiting for deployment "hello" rollout to finish: 2 of 3 updated replicas are available...
deployment "hello" successfully rolled out
NAME                    READY   STATUS    RESTARTS   AGE
hello-c6cf94684-b5hp4   1/1     Running   0          6s
hello-c6cf94684-fshn6   1/1     Running   0          6s
hello-c6cf94684-xgd48   1/1     Running   0          6s
```

`Running` but `0/1`: the containers run, but they are not ready during the 3-second warm-up. The events of a pod tell the story (use your own pod name):

```bash
kubectl describe pod hello-c6cf94684-b5hp4
```

```text
Events:
  Type     Reason     Age              From               Message
  ----     ------     ----             ----               -------
  Normal   Scheduled  6s               default-scheduler  Successfully assigned default/hello-c6cf94684-b5hp4 to course-21-05-control-plane
  Normal   Pulled     5s               kubelet            spec.containers{hello}: Container image "course-p21-hello:0.3.0" already present on machine and can be accessed by the pod
  Normal   Created    5s               kubelet            spec.containers{hello}: Container created
  Normal   Started    5s               kubelet            spec.containers{hello}: Container started
  Warning  Unhealthy  2s (x3 over 5s)  kubelet            spec.containers{hello}: Readiness probe failed: HTTP probe failed with statuscode: 503
```

(Only the `Events` part is shown.) Three readiness failures with `503`, then success. This is what you want: during the warm-up, no request reaches the pod. Now that all three are ready, the EndpointSlice lists them, and the program's logs show the warm-up:

```bash
kubectl get endpointslices -l kubernetes.io/service-name=hello
kubectl logs hello-c6cf94684-b5hp4
```

```text
NAME          ADDRESSTYPE   PORTS   ENDPOINTS                          AGE
hello-xmzmt   IPv4          8080    10.244.0.5,10.244.0.6,10.244.0.7   6s
p21-hello 0.3.0 listening on port 8080
warm-up finished, ready for traffic
```

Call the service from inside the cluster, first without and then with the token. `--env` passes the token into the temporary pod as a variable, and the single quotes make the pod's shell (not yours) expand `$TOKEN`:

```bash
kubectl run client -q --rm -i --restart=Never --image=busybox:1.37 -- \
  sh -c 'sleep 2; wget -qO- http://hello; wget -qO- http://hello/admin'
kubectl run client -q --rm -i --restart=Never --image=busybox:1.37 --env="TOKEN=$(cat admin-token.txt)" -- \
  sh -c 'sleep 2; wget -qO- --header "Authorization: Bearer $TOKEN" http://hello/admin'
```

```text
salam from hello-c6cf94684-fshn6 (version 0.3.0)
wget: server returned error: HTTP/1.1 401 Unauthorized
welcome, admin
```

The greeting came from the ConfigMap, the token from the Secret, and the same image would work in any other cluster with other values.

## 7. Changing configuration

Change `GREETING: "salam"` to `GREETING: "privet"` in `k8s/configmap.yaml` and apply only that file. Then ask the service:

```bash
kubectl apply -f k8s/configmap.yaml
kubectl run client -q --rm -i --restart=Never --image=busybox:1.37 -- sh -c 'sleep 2; wget -qO- http://hello'
```

```text
configmap/hello-config configured
salam from hello-c6cf94684-b5hp4 (version 0.3.0)
```

Still `salam`. Environment variables are set **once**, when the container starts; a process cannot see later changes to them. The ConfigMap changed, but the pod template of the Deployment did not, so no controller replaced anything. Ask for new pods:

```bash
kubectl rollout restart deployment/hello
kubectl rollout status deployment/hello
kubectl run client -q --rm -i --restart=Never --image=busybox:1.37 -- sh -c 'sleep 2; wget -qO- http://hello'
```

```text
deployment.apps/hello restarted
Waiting for deployment "hello" rollout to finish: 0 out of 3 new replicas have been updated...
Waiting for deployment "hello" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "hello" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "hello" rollout to finish: 1 old replicas are pending termination...
deployment "hello" successfully rolled out
privet from hello-b79875f4f-wftgp (version 0.3.0)
```

(Repeated "Waiting" lines are left out.) `rollout restart` adds a time stamp annotation to the pod template, which counts as a template change, so the Deployment does a normal rolling update with readiness checks and no downtime. A Helm chart (Lesson 21.6) can do this automatically: it puts a checksum of the ConfigMap into the template, so that every configuration change also changes the template.

> 💡 **Tip:** files from a mounted ConfigMap or Secret *are* updated by the kubelet after a change (with a delay of up to about a minute), but only if the program reads the file again. Our program reads the token once at start, so it still needs a restart. Reading once and restarting on change is simpler and easier to reason about.

## 8. When a liveness probe is wrong

Make a realistic mistake: change the liveness path in `k8s/deployment.yaml` from `/healthz` to `/nope` (an endpoint that does not exist), apply it, and wait about a minute:

```bash
kubectl apply -f k8s/deployment.yaml
kubectl rollout status deployment/hello
kubectl get pods -l app=hello
```

```text
NAME                     READY   STATUS    RESTARTS      AGE
hello-7d97657c47-49c7s   1/1     Running   1 (25s ago)   55s
hello-7d97657c47-7hp9p   1/1     Running   1 (20s ago)   50s
hello-7d97657c47-vl5lc   1/1     Running   1 (30s ago)   60s
```

The rollout succeeded (readiness passes), but every pod has already been **restarted** once, and it will happen again every 30 seconds or so. The last events of a pod say why:

```text
  Normal   Killing    25s                kubelet            spec.containers{hello}: Container hello failed liveness probe, will be restarted
  Warning  Unhealthy  22s (x6 over 55s)  kubelet            spec.containers{hello}: Readiness probe failed: HTTP probe failed with statuscode: 503
  Warning  Unhealthy  5s (x5 over 45s)   kubelet            spec.containers{hello}: Liveness probe failed: HTTP probe failed with statuscode: 404
```

`404` from `/nope`, three times in a row, so the kubelet killed the container. The logs of the **previous** (killed) container show that the program itself was perfectly healthy and simply received `SIGTERM`:

```bash
kubectl logs hello-7d97657c47-49c7s --previous
```

```text
p21-hello 0.3.0 listening on port 8080
warm-up finished, ready for traffic
received SIGTERM, shutting down
all connections closed, bye
```

This is a real-world lesson: a rising `RESTARTS` count with clean logs usually means a wrong probe, not a crashing program. (A crashing program would show a panic in `--previous`, and the status `CrashLoopBackOff`.) Fix the path back to `/healthz` and apply again; the rolling update replaces the pods, and the restarts stop.

## 9. Clean up

```bash
kind delete cluster --name course-21-05
docker image rm course-p21-hello:0.3.0
```

Keep `admin-token.txt` out of Git, and keep the code: Lesson 21.6 packages it.

## Common mistakes

**Expecting a ConfigMap change to reach running pods.** As Section 7 showed, environment variables do not change in a running process. Fix: `kubectl rollout restart deployment/<name>`, or let Helm add a checksum of the configuration to the pod template.

**Treating base64 as protection.** A Secret manifest in Git with `data: admin-token: Y2Mx...` is a leaked secret: `base64 -d` reveals it in a second. Fix: create Secrets from files that are in `.gitignore`, or use an encrypted-secrets tool.

**A missing secret, discovered late.** A service that reads its token only when the first admin request arrives starts "fine" and fails later, at a bad moment. Fix: read and check every required setting at start and exit with a clear message, as `main` does here (`cannot read the admin token from ...`). The pod then never becomes ready, and the rolling update stops before it reaches users.

## Try it yourself

1. Set `STARTUP_DELAY_MS: "0"` in the ConfigMap, apply, restart the Deployment, and check that the pods become ready almost immediately (no `Readiness probe failed` events).
2. Delete the Secret (`kubectl delete secret hello-admin`) and restart the Deployment. The new pod stays in `ContainerCreating`, and its events say `MountVolume.SetUp failed for volume "admin-secret" : secret "hello-admin" not found`. Why does the service keep answering? (Recreate the Secret afterwards; the pod then starts by itself.)
3. Clean up as in Section 9.

## Exercises

### Exercise 1 — A slow warm-up (easy)

Set `STARTUP_DELAY_MS` to `"10000"` (10 seconds), apply the ConfigMap, run `kubectl rollout restart deployment/hello`, and measure how long `kubectl rollout status` takes. Explain the number using `maxSurge: 1` and `maxUnavailable: 0`.

<details>
<summary>Hint</summary>

Use `/usr/bin/time -p kubectl rollout status deployment/hello`. Think: how many pods are replaced at the same time, and how long does each new pod need to become ready?

</details>

<details>
<summary>Answer</summary>

```text
deployment "hello" successfully rolled out
real 32.84
user 0.04
sys 0.02
```

About 33 seconds. With `maxSurge: 1` and `maxUnavailable: 0`, Kubernetes replaces one pod at a time and waits until the new pod is **ready** before it stops an old one. Each new pod needs about 10 seconds of warm-up, plus a little for starting and for the next readiness check (every 2 seconds): 3 pods × about 11 seconds ≈ 33 seconds. A faster start makes every update and every scale-up faster; a larger `maxSurge` replaces more pods in parallel. Set the value back to `"3000"` afterwards.

</details>

### Exercise 2 — Test the readiness endpoint (medium)

Write a test `readyz_follows_the_ready_flag`: with `ready` set to `false`, `/readyz` answers `503`; after you set it to `true`, it answers `200` with the body `ready\n`.

<details>
<summary>Hint</summary>

`test_state()` gives a state with `ready: true`. Because `ready` is an `Arc<AtomicBool>`, a clone of the state shares the same flag: change it with `state.ready.store(false, Ordering::Relaxed)`, send a request with `state.clone()`, change it again, send again.

</details>

<details>
<summary>Answer</summary>

```rust
    #[tokio::test]
    async fn readyz_follows_the_ready_flag() {
        let state = test_state();
        state.ready.store(false, Ordering::Relaxed);
        let (status, _) = send(state.clone(), get("/readyz")).await;
        assert_eq!(status, StatusCode::SERVICE_UNAVAILABLE);

        state.ready.store(true, Ordering::Relaxed);
        let (status, body) = send(state, get("/readyz")).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, "ready\n");
    }
```

```text
test tests::readyz_follows_the_ready_flag ... ok
test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `Clone` on `AppState` clones the `Arc`, not the `AtomicBool`, so the router and the test look at the same flag. This is the same sharing that lets the warm-up task change what the handlers see.

</details>

### Exercise 3 — Stop being ready before you stop (hard)

Lesson 21.4 mentioned a race: right after `SIGTERM`, some connections may still arrive. Change `shutdown_signal` so that, after the signal, it first sets `ready` to `false`, prints a line, waits 2 seconds, and only then lets axum stop. Release it as version 0.3.1, deploy it, delete one pod while you follow its logs, and time the deletion.

<details>
<summary>Hint</summary>

`shutdown_signal` needs the flag: give it a parameter `ready: Arc<AtomicBool>`, and in `main` clone `state.ready` into a variable **before** `app(state)` moves `state`. Remember `cargo test` before `docker build` (the version changed).

</details>

<details>
<summary>Answer</summary>

```rust
/// Finishes when the process gets SIGTERM (from Kubernetes) or Ctrl+C (from you).
/// Before it finishes, it reports "not ready" and waits, so that the Service
/// stops sending new connections while this pod still answers.
async fn shutdown_signal(ready: Arc<AtomicBool>) {
    let mut term = signal(SignalKind::terminate()).expect("cannot listen for SIGTERM");
    tokio::select! {
        _ = term.recv() => println!("received SIGTERM, shutting down"),
        _ = tokio::signal::ctrl_c() => println!("received Ctrl+C, shutting down"),
    }
    ready.store(false, Ordering::Relaxed);
    println!("not ready any more, waiting 2 s for the endpoints to update");
    tokio::time::sleep(Duration::from_secs(2)).await;
}
```

In `main`:

```rust
    let ready_flag = state.ready.clone();
    axum::serve(listener, app(state))
        .with_graceful_shutdown(shutdown_signal(ready_flag))
```

After building, loading and `kubectl set image deployment/hello hello=course-p21-hello:0.3.1`, deleting a pod took about 3 seconds instead of 1:

```text
pod "hello-7bfbb55b66-447nr" deleted from default namespace
real 2.93
user 0.03
sys 0.02
```

and its logs showed the new order:

```text
p21-hello 0.3.1 listening on port 8080
warm-up finished, ready for traffic
received SIGTERM, shutting down
not ready any more, waiting 2 s for the endpoints to update
all connections closed, bye
```

Why it works: Kubernetes removes a terminating pod from the endpoints anyway, but that change needs a moment to reach kube-proxy on every node. During the 2-second pause the pod still accepts and answers connections that arrive late, and reports "not ready" to any probe. Only then does axum stop accepting. The pause must stay well below the grace period (30 s).

</details>

## Quiz

1. Why should configuration not be built into the image?

<details>
<summary>Answer</summary>

Because the same image must run in different environments with different settings. With configuration outside (ConfigMaps, Secrets), the image you tested is exactly the image you run.

</details>

2. Is a Kubernetes Secret encrypted? What actually protects it?

<details>
<summary>Answer</summary>

Not by default: the values are only base64-encoded, and etcd stores them as they are unless the administrator turns on encryption at rest. Protection comes from access control (who may read Secrets), optional etcd encryption, and keeping secrets out of Git.

</details>

3. What happens to a pod whose readiness probe fails? And whose liveness probe fails three times?

<details>
<summary>Answer</summary>

Readiness failure: the pod is removed from the Service's endpoints and gets no traffic, but keeps running. Liveness failure (after `failureThreshold` times): the kubelet kills the container and starts it again; the restart count grows.

</details>

4. Why should a liveness probe not check the database?

<details>
<summary>Answer</summary>

If the database is down, every pod fails liveness and is restarted again and again. Restarting does not fix the database, and it adds load. Liveness should check only whether the process itself is stuck.

</details>

5. You changed a ConfigMap that is used through `envFrom`. Why do the pods not see the change, and what do you run?

<details>
<summary>Answer</summary>

Environment variables are set when a container starts and do not change in a running process; the pod template did not change, so nothing was replaced. Run `kubectl rollout restart deployment/<name>`.

</details>

6. The `RESTARTS` column of your pods keeps growing, but `kubectl logs --previous` shows a clean shutdown after `SIGTERM`. What is the likely cause?

<details>
<summary>Answer</summary>

A failing liveness probe (wrong path, wrong port, too short timeout): the kubelet kills a healthy program. A crashing program would show a panic or an error in the previous logs instead.

</details>

## Summary

- Build one image and configure it from outside: ordinary settings in a ConfigMap, secrets in a Secret.
- `envFrom` turns a ConfigMap into environment variables; a Secret is safer mounted as read-only files.
- base64 is not encryption. Protect Secrets with access control, etcd encryption and by keeping them out of Git.
- Readiness decides traffic (fail → removed from endpoints); liveness decides restarts (fail → container restarted); startup protects slow starts.
- Keep `/healthz` trivial; put "can I serve now?" logic into `/readyz`; fail fast at start when required configuration is missing.
- Running pods do not see a changed ConfigMap in their environment: use `kubectl rollout restart`.
- Growing restarts with clean previous logs point to a wrong probe.

## Sources and further reading

- [Kubernetes documentation — ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) — all ways to use a ConfigMap.
- [Kubernetes documentation — Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) — Secret types and how to use them.
- [Kubernetes documentation — Good practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/) — access control, encryption at rest and other advice.
- [Kubernetes documentation — Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) — every probe field with examples.
- [Kubernetes documentation — kubectl rollout restart](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_restart/) — the command reference.
- [The Twelve-Factor App — III. Config](https://12factor.net/config) — why configuration belongs in the environment.

---

[🏠 Start](../START-HERE.md) · [← 21.4 Deploying a Rust service to Kubernetes](04-deploying-a-rust-service-to-kubernetes.md) · [📋 Phase 21](README.md) · [21.6 Packaging with Helm and Kustomize →](06-helm-and-kustomize.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
