<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 21.3 A local cluster with kind and kubectl](03-a-local-cluster-with-kind.md) · [📋 Phase 21](README.md) · [21.5 ConfigMaps, Secrets and health probes →](05-configmaps-secrets-and-probes.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/21-cloud-native-infrastructure/04-deploying-a-rust-service-to-kubernetes.md) <!-- nav -->

# Lesson 21.4 — Deploying a Rust service to Kubernetes

> **Phase 21 — Cloud-native infrastructure** · Lesson 4 of 14 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- How to write a small axum service that behaves well on Kubernetes: it reads its port from the environment, logs to standard output and stops gracefully on `SIGTERM`.
- How to build a small, non-root image for it and load the image into a kind cluster without a registry.
- How to write a Deployment and a Service for your own program, with resource requests and limits and a strict security context.
- How a rolling update replaces pods one by one, and how to roll back.
- What exactly happens when Kubernetes stops a pod.

## Before you start

- You finished [Lesson 21.3 — A local cluster with kind and kubectl](03-a-local-cluster-with-kind.md).
- You can write an axum service and test its router with `oneshot` ([Lesson 16.3 — Web services with axum](../16-backend-services/03-web-services-with-axum.md)).
- You know multi-stage Dockerfiles for Rust ([Lesson 11.13](../11-linux-shell-and-containers/13-dockerfiles-for-rust.md)) and graceful shutdown with Tokio ([Lesson 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md)).

## New words

| Word | Meaning |
|---|---|
| **registry** | A server that stores images, such as Docker Hub or GitHub's `ghcr.io`. Nodes normally download (*pull*) images from a registry. |
| **`imagePullPolicy`** | When the kubelet downloads the image: `IfNotPresent` (only if the node does not have it), `Always`, or `Never`. |
| **distroless image** | A base image that contains only what a program needs to run (C library, certificates, time zones), with no shell and no package manager. |
| **rolling update** | Replacing the pods of a Deployment a few at a time, so that the service keeps running during the update. |
| **`maxSurge` / `maxUnavailable`** | Two numbers of a rolling update: how many pods may exist *above* the desired count, and how many may be missing *below* it. |
| **revision** | A numbered version of a Deployment's pod template. Every change of the template makes a new revision. |
| **rollback** | Going back to an earlier revision. |
| **resource request** | The amount of CPU (Central Processing Unit) and memory a container is guaranteed. The scheduler places pods by their requests. |
| **resource limit** | The most a container may use. Above its memory limit it is killed. |
| **millicore** (`m`) | One thousandth of a CPU core: `50m` = 5% of one core. |
| **Mi** (mebibyte) | 1,048,576 bytes (2²⁰). Kubernetes writes memory as `16Mi`, `1Gi`. |
| **OOM** (Out Of Memory) **kill** | The Linux kernel stops a process that goes over its memory limit. |
| **security context** | Settings of a pod or container that restrict what it may do: run as a non-root user, read-only file system, no extra Linux capabilities. |
| **grace period** | The time Kubernetes waits after `SIGTERM` before it sends `SIGKILL` (default 30 seconds). |

## 1. A service that fits the platform

Lesson 21.1 listed what the platform expects from a service. Here is a small service that does all of it. Create the package:

```bash
cd ~/projects/rust
cargo new code/p21-hello
cd code/p21-hello
cargo add axum@0.8
cargo add tokio@1 --features macros,rt-multi-thread,net,signal
cargo add --dev tower@0.5 --features util
cargo add --dev http-body-util@0.1
```

`Cargo.toml` now has:

```toml
[package]
name = "p21-hello"
version = "0.1.0"
edition = "2024"

[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["macros", "rt-multi-thread", "net", "signal"] }

[dev-dependencies]
http-body-util = "0.1"
tower = { version = "0.5", features = ["util"] }
```

Replace `src/main.rs` with:

```rust
use std::sync::Arc;

use axum::{Router, extract::State, routing::get};
use tokio::net::TcpListener;
use tokio::signal::unix::{SignalKind, signal};

/// Data every request handler can read.
#[derive(Clone)]
struct AppState {
    pod: Arc<str>,
}

fn app(state: AppState) -> Router {
    Router::new()
        .route("/", get(hello))
        .route("/healthz", get(|| async { "ok\n" }))
        .with_state(state)
}

async fn hello(State(state): State<AppState>) -> String {
    format!(
        "hello from {} (version {})\n",
        state.pod,
        env!("CARGO_PKG_VERSION")
    )
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
    let port = std::env::var("PORT").unwrap_or_else(|_| "8080".to_string());
    let pod = std::env::var("HOSTNAME").unwrap_or_else(|_| "localhost".to_string());

    let listener = TcpListener::bind(format!("0.0.0.0:{port}"))
        .await
        .expect("cannot bind the port");
    println!(
        "p21-hello {} listening on port {port}",
        env!("CARGO_PKG_VERSION")
    );

    axum::serve(listener, app(AppState { pod: pod.into() }))
        .with_graceful_shutdown(shutdown_signal())
        .await
        .expect("server error");
    println!("all connections closed, bye");
}

#[cfg(test)]
mod tests {
    use super::*;
    use axum::body::Body;
    use axum::http::{Request, StatusCode};
    use http_body_util::BodyExt;
    use tower::ServiceExt;

    async fn get_text(uri: &str) -> (StatusCode, String) {
        let state = AppState {
            pod: "test-pod".into(),
        };
        let request = Request::get(uri).body(Body::empty()).unwrap();
        let response = app(state).oneshot(request).await.unwrap();
        let status = response.status();
        let bytes = response.into_body().collect().await.unwrap().to_bytes();
        (status, String::from_utf8(bytes.to_vec()).unwrap())
    }

    #[tokio::test]
    async fn hello_names_the_pod_and_the_version() {
        let (status, body) = get_text("/").await;
        assert_eq!(status, StatusCode::OK);
        let expected = format!(
            "hello from test-pod (version {})\n",
            env!("CARGO_PKG_VERSION")
        );
        assert_eq!(body, expected);
    }

    #[tokio::test]
    async fn healthz_says_ok() {
        assert_eq!(
            get_text("/healthz").await,
            (StatusCode::OK, "ok\n".to_string())
        );
    }
}
```

**What happens:**

1. `GET /` answers with the name of the pod and the version. `env!("CARGO_PKG_VERSION")` is filled in **at compile time** from `version` in `Cargo.toml`, so the binary always knows which version it is. Seeing the pod name in each answer will show us how a Service spreads requests.
2. `GET /healthz` answers `ok`. Lesson 21.5 lets Kubernetes call it.
3. The port comes from the environment variable `PORT` (default 8080), and the pod name from `HOSTNAME`. Kubernetes sets `HOSTNAME` to the pod's name in every container.
4. The listener binds to `0.0.0.0`, which means "all network interfaces". `127.0.0.1` would accept connections only from inside the same container, and the Service could not reach the program.
5. `shutdown_signal` waits for `SIGTERM` or Ctrl+C. `with_graceful_shutdown` makes axum stop accepting new connections when the future finishes, let running requests complete, and then return. Every log line goes to standard output with `println!`.
6. The tests send requests to the router in memory, with no network (Lesson 16.3). The expected text uses the same `env!` macro, so the test still passes after you change the version.

Run the tests, then the service on another port to see the `PORT` variable work:

```bash
cargo test
PORT=8090 cargo run
```

```text
running 2 tests
test tests::healthz_says_ok ... ok
test tests::hello_names_the_pod_and_the_version ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```text
p21-hello 0.1.0 listening on port 8090
```

In a second tab, `curl -s http://localhost:8090/` prints `hello from localhost (version 0.1.0)` (on your Mac `HOSTNAME` is usually not set, so the default is used). Press Ctrl+C in the first tab:

```text
received Ctrl+C, shutting down
all connections closed, bye
```

## 2. The image

Create a `Dockerfile` in `code/p21-hello`:

```dockerfile
# Stage 1: build the program with the full Rust toolchain.
FROM rust:1.100 AS build
WORKDIR /src
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release --locked

# Stage 2: a small runtime image with only the program.
FROM gcr.io/distroless/cc-debian13:nonroot
COPY --from=build /src/target/release/p21-hello /usr/local/bin/p21-hello
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/p21-hello"]
```

and a `.dockerignore` with two lines, `target` and `.git`, so that Docker does not send gigabytes of build output into the build.

**What happens:**

1. The first stage uses the official `rust:1.100` image (Debian 13) and builds a release binary. `--locked` makes the build fail if `Cargo.lock` does not match `Cargo.toml`, so the image is built from exactly the dependency versions you tested.
2. The second stage starts from Google's **distroless** image `cc-debian13`, which has the C library that Rust programs link against and nothing else: no shell, no package manager. The `:nonroot` variant runs as the user with id 65532, not as root.
3. `ENTRYPOINT` in the JSON (JavaScript Object Notation) or "exec" form starts our program directly as process number 1 of the container. Kubernetes sends `SIGTERM` to process 1, so our handler receives it. (In the shell form, `ENTRYPOINT p21-hello`, a shell would be process 1 and might not pass the signal on.)

Build and check:

```bash
docker build -t course-p21-hello:0.1.0 .
docker image ls --filter reference=course-p21-hello --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}'
```

```text
REPOSITORY         TAG       SIZE
course-p21-hello   0.1.0     56.4MB
```

The tag `0.1.0` matches the version in `Cargo.toml`: one exact tag per version, never `latest`. (If you like, check the image with plain Docker first: `docker run --rm -p 127.0.0.1:8090:8080 course-p21-hello:0.1.0`, `curl` it from another tab, and stop it with Ctrl+C.)

> 🔒 **Security:** the distroless image has no shell, so `docker exec ... sh` and `kubectl exec ... sh` do not work in it. That is a feature: an attacker who finds a bug in your program has no shell and no tools to download more. To debug, use logs, metrics and `kubectl debug` (a temporary helper container), not a shell in the production image.

## 3. Getting the image into the cluster

Create the cluster for this lesson:

```bash
kind create cluster --name course-21-04
kubectl wait --for=condition=Ready node --all --timeout=120s
```

Normally a node pulls images from a **registry**. Your image exists only in Docker on your Mac, and the kind node is a separate container with its own containerd and its own image store. `kind load` copies the image into the node:

```bash
kind load docker-image course-p21-hello:0.1.0 --name course-21-04
docker exec course-21-04-control-plane crictl images | grep p21-hello
```

```text
Image: "course-p21-hello:0.1.0" with ID "sha256:ab5aca63e76d8c263f1a75920e98a1d0fba8bb79c7ab071989ec99c80dbdc8dc" not yet present on node "course-21-04-control-plane", loading...
docker.io/library/course-p21-hello              0.1.0                135d15789c7a6       11.7MB
```

`crictl` is the command-line tool of the container runtime inside the node. The image is there, under the full name `docker.io/library/course-p21-hello:0.1.0` (a name without a registry is short for Docker Hub). In Lesson 21.7, the CI (Continuous Integration) pipeline pushes images to a real registry instead.

## 4. The manifests

Keep the manifests next to the code, in `code/p21-hello/k8s/`. `k8s/deployment.yaml`:

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
      maxSurge: 1        # at most 1 extra pod during an update
      maxUnavailable: 0  # never go below 3 ready pods
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: course-p21-hello:0.1.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
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
```

`k8s/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello
spec:
  selector:
    app: hello
  ports:
    - name: http
      port: 80
      targetPort: http
```

**What happens:**

1. `strategy` describes the rolling update: with `maxSurge: 1` and `maxUnavailable: 0`, Kubernetes first starts **one new** pod, waits until it is ready, then stops **one old** pod, and repeats. There are always at least 3 ready pods.
2. `imagePullPolicy: IfNotPresent` uses the image that `kind load` put on the node. (This is already the default for an exact tag; the default for `latest` is `Always`, which would try Docker Hub and fail.)
3. The container port has a **name**, `http`. The Service uses `targetPort: http`, so if the port number ever changes, only the Deployment needs editing.
4. `resources.requests` is what the scheduler *reserves* on a node for each pod: 50 millicores of CPU and 16 Mi of memory. `limits.memory: 64Mi` is a hard ceiling: above it the kernel kills the container (an OOM kill). There is deliberately no CPU limit: a CPU limit does not kill, it slows the program down (*throttling*), and many teams prefer to set only a request.
5. The `securityContext` follows the principle of least privilege (Phase 10): the kubelet refuses to start the container if it would run as root, the process can never gain more rights, the root file system is read-only (our program writes nothing to disk), and all Linux capabilities are dropped.
6. A file can hold several objects separated by a line `---`; here we use one file per object, and `kubectl apply -f k8s/` applies the whole folder.

> 📏 **Convention:** keep the manifests in Git next to the code they deploy (`k8s/` or `deploy/`), one object per file, named after the kind. Lesson 21.6 turns this folder into a Helm chart and a Kustomize base.

## 5. Deploy and talk to it

```bash
kubectl apply -f k8s/
kubectl rollout status deployment/hello
kubectl get pods -l app=hello -o wide
```

```text
deployment.apps/hello created
service/hello created
Waiting for deployment "hello" rollout to finish: 0 of 3 updated replicas are available...
Waiting for deployment "hello" rollout to finish: 1 of 3 updated replicas are available...
Waiting for deployment "hello" rollout to finish: 2 of 3 updated replicas are available...
deployment "hello" successfully rolled out
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE                         NOMINATED NODE   READINESS GATES
hello-64bccb4655-5z2fn   1/1     Running   0          1s    10.244.0.5   course-21-04-control-plane   <none>           <none>
hello-64bccb4655-kkgfb   1/1     Running   0          1s    10.244.0.7   course-21-04-control-plane   <none>           <none>
hello-64bccb4655-nqc95   1/1     Running   0          1s    10.244.0.6   course-21-04-control-plane   <none>           <none>
```

Send six requests from inside the cluster, through the Service:

```bash
kubectl run client -q --rm -i --restart=Never --image=busybox:1.37 -- \
  sh -c 'for i in 1 2 3 4 5 6; do wget -qO- http://hello; done'
```

```text
hello from hello-64bccb4655-5z2fn (version 0.1.0)
hello from hello-64bccb4655-kkgfb (version 0.1.0)
hello from hello-64bccb4655-nqc95 (version 0.1.0)
hello from hello-64bccb4655-5z2fn (version 0.1.0)
hello from hello-64bccb4655-kkgfb (version 0.1.0)
hello from hello-64bccb4655-kkgfb (version 0.1.0)
```

Each `wget` opens a new connection, and the Service sends each connection to one of the three pods. The choice is random, so your order will differ. (If you see fewer than six lines, start the command with `sleep 2;`; Exercise 1 explains why.) From your Mac, `kubectl port-forward service/hello 8088:80` works as in Lesson 21.3; it prints `Forwarding from 127.0.0.1:8088 -> 8080` (kubectl resolved the Service port 80 to the named container port) and always talks to the same single pod. The logs are on standard output, so `kubectl logs` finds them:

```bash
kubectl logs deployment/hello
```

```text
Found 3 pods, using pod/hello-64bccb4655-5z2fn
p21-hello 0.1.0 listening on port 8080
```

## 6. A rolling update

Release version 0.2.0. Change `version = "0.1.0"` to `version = "0.2.0"` in `Cargo.toml`, then **run the tests first**: this also updates `Cargo.lock`, which the `--locked` build needs (see "Common mistakes"). Then build, load and change the image tag in `k8s/deployment.yaml` to `course-p21-hello:0.2.0`:

```bash
cargo test
docker build -t course-p21-hello:0.2.0 .
kind load docker-image course-p21-hello:0.2.0 --name course-21-04
kubectl diff -f k8s/deployment.yaml
```

The interesting part of the diff:

```text
-      - image: course-p21-hello:0.1.0
+      - image: course-p21-hello:0.2.0
```

Apply it and watch the rollout:

```bash
kubectl apply -f k8s/deployment.yaml
kubectl rollout status deployment/hello
```

```text
deployment.apps/hello configured
Waiting for deployment "hello" rollout to finish: 1 out of 3 new replicas have been updated...
Waiting for deployment "hello" rollout to finish: 2 out of 3 new replicas have been updated...
Waiting for deployment "hello" rollout to finish: 1 old replicas are pending termination...
deployment "hello" successfully rolled out
```

(Each "Waiting" line appeared several times; repeated lines are left out.) Now look at the ReplicaSets and the history:

```bash
kubectl get replicasets -l app=hello
kubectl rollout history deployment/hello
```

```text
NAME               DESIRED   CURRENT   READY   AGE
hello-64bccb4655   0         0         0       43s
hello-7d689ffb7b   3         3         3       2s
deployment.apps/hello 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```

This is the two-level design from Lesson 21.2 at work. The Deployment created a second ReplicaSet (`7d689ffb7b`) for the new template and moved the count across, one pod at a time; the old ReplicaSet (`64bccb4655`) is kept with 0 pods, so that a rollback is quick. The requests now reach version 0.2.0:

```text
hello from hello-7d689ffb7b-s49gl (version 0.2.0)
hello from hello-7d689ffb7b-jvw4p (version 0.2.0)
hello from hello-7d689ffb7b-s49gl (version 0.2.0)
```

## 7. Rolling back

Suppose 0.2.0 has a bug. The quick emergency button is `rollout undo`:

```bash
kubectl rollout undo deployment/hello
kubectl get replicasets -l app=hello
```

```text
Warning: resource deployments/hello was previously managed with 'kubectl apply'. Rolling back will not update the kubectl.kubernetes.io/last-applied-configuration annotation, which may cause unexpected behavior on future 'kubectl apply' operations. Consider using 'kubectl apply' with your previous configuration file instead.
deployment.apps/hello rolled back
NAME               DESIRED   CURRENT   READY   AGE
hello-64bccb4655   3         3         3       47s
hello-7d689ffb7b   0         0         0       6s
```

The old ReplicaSet is back to 3 pods. Read the warning carefully: the cluster now runs 0.1.0, but your file in Git still says 0.2.0. The cluster and Git disagree, and the next `kubectl apply` would bring 0.2.0 back. So `rollout undo` is for emergencies only; afterwards, also change the file back (in Git: `git revert` of the commit that changed the tag) and apply it. The everyday way to roll back *is* to apply the previous file.

## 8. How Kubernetes stops a pod

Every update, scale-down and node maintenance stops pods. The steps:

```text
 kubectl delete pod / rolling update / scale down
        │
        ▼
 1. pod marked "Terminating"  ──►  removed from the Service's EndpointSlice
        │                          (no new connections are sent to it)
        ▼
 2. SIGTERM to process 1 of each container
        │   your program: stop accepting, finish requests, exit
        ▼
 3. waits up to terminationGracePeriodSeconds (default 30 s)
        │
        ▼
 4. still running? ──► SIGKILL (cannot be caught; requests in flight are lost)
```

Watch it with a real pod. Follow the logs of one pod in a second tab (`kubectl logs -f <pod>`), then delete that pod and time it:

```bash
/usr/bin/time -p kubectl delete pod hello-64bccb4655-b4krd
```

```text
pod "hello-64bccb4655-b4krd" deleted from default namespace
real 1.37
user 0.03
sys 0.01
```

The second tab showed:

```text
p21-hello 0.1.0 listening on port 8080
received SIGTERM, shutting down
all connections closed, bye
```

The pod was gone in about a second, because our program exited cleanly on `SIGTERM`. A program that ignores `SIGTERM` would keep `kubectl delete` waiting for the full 30 seconds and then be killed, which makes every rolling update slow and drops requests.

> ⚡ **Performance:** steps 1 and 2 happen at about the same time, not one after the other. For a short moment, other nodes may still send new connections to a pod that already got `SIGTERM`. Busy services therefore wait a few seconds after `SIGTERM` before they stop accepting (or use a `preStop` hook that waits). For this small service the default behaviour is fine.

## 9. Clean up

```bash
kubectl delete -f k8s/
kind delete cluster --name course-21-04
docker image rm course-p21-hello:0.1.0 course-p21-hello:0.2.0
```

Keep the code and the `k8s/` folder: Lesson 21.5 extends them.

## Common mistakes

**Changing the version but not `Cargo.lock`.** If you edit `version` in `Cargo.toml` and go straight to `docker build`, the `--locked` build fails, because the lock file still records the old version of your own package:

```text
error: cannot update the lock file /src/Cargo.lock because --locked was passed to prevent this
help: to generate the lock file without accessing the network, remove the --locked flag and use --offline instead.
```

Fix: run `cargo test` (or `cargo build`) on your Mac first; it updates `Cargo.lock`. Commit both files together.

**Forgetting `kind load`.** The node does not have the image and tries Docker Hub, where it does not exist. The new pod stays in `ImagePullBackOff`, and its events say:

```text
Failed to pull image "course-p21-hello:0.2.0": failed to pull and unpack image "docker.io/library/course-p21-hello:0.2.0": failed to resolve reference "docker.io/library/course-p21-hello:0.2.0": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
```

Notice what the rolling update did meanwhile:

```text
NAME                     READY   STATUS             RESTARTS   AGE
hello-64bccb4655-2wp99   1/1     Running            0          9m45s
hello-64bccb4655-692f7   1/1     Running            0          9m45s
hello-64bccb4655-fmn9n   1/1     Running            0          9m45s
hello-7d689ffb7b-f5q8v   0/1     ImagePullBackOff   0          9m30s
```

Because of `maxUnavailable: 0`, no old pod was stopped: the service kept working with three pods of 0.1.0. A broken update stalls instead of taking the service down. Fix: `kind load docker-image ...`; the kubelet retries and the rollout continues.

**Using `latest`.** With the tag `latest`, the default `imagePullPolicy` is `Always`, so the kubelet ignores the loaded image and goes to Docker Hub. Even with a registry, `latest` makes it impossible to say which code runs or to roll back. Use exact tags.

## Try it yourself

1. Repeat Sections 3–6 on a fresh cluster, but this time keep `kubectl get pods -l app=hello --watch` open in a second tab during the rolling update. Count how many pods exist at most at the same time. (Answer: 4, because `maxSurge: 1`.)
2. Change `maxUnavailable` to `1` and `maxSurge` to `0`, apply, and do another update (0.3.0). What is different in the watch?
3. Clean up as in Section 9.

## Exercises

### Exercise 1 — Five replicas (easy)

Scale `hello` to 5 replicas and send 20 requests from inside the cluster. Count how many answers each pod gave.

<details>
<summary>Hint</summary>

`kubectl scale`, then a `kubectl run ... -- sh -c 'sleep 2; for i in $(seq 20); do wget -qO- http://hello; done'` piped into `sort | uniq -c` on your Mac.

</details>

<details>
<summary>Answer</summary>

```bash
kubectl scale deployment hello --replicas=5
kubectl rollout status deployment/hello
kubectl run client -q --rm -i --restart=Never --image=busybox:1.37 -- \
  sh -c 'sleep 2; for i in $(seq 20); do wget -qO- http://hello; done' | sort | uniq -c
```

```text
   1 hello from hello-64bccb4655-2mjbx (version 0.1.0)
   7 hello from hello-64bccb4655-8mmg5 (version 0.1.0)
   2 hello from hello-64bccb4655-9xrkn (version 0.1.0)
   4 hello from hello-64bccb4655-l49gk (version 0.1.0)
   6 hello from hello-64bccb4655-lrg5v (version 0.1.0)
```

All 20 answers arrived, spread over the five pods. The distribution is random per connection, not perfectly even. Why `sleep 2`? `kubectl run -i` attaches to the pod only after the pod has started. Without the pause, the first answers can be printed before kubectl is listening and are lost: in one run without it, only 10 of 20 lines arrived. Scale back with `kubectl scale deployment hello --replicas=3`.

</details>

### Exercise 2 — A `/version` endpoint, with a test (medium)

Add `GET /version` that answers with the version and a newline (for example `0.1.0`). Write a test `version_matches_cargo_toml` first, watch it fail with `404`, then make it pass.

<details>
<summary>Hint</summary>

`concat!(env!("CARGO_PKG_VERSION"), "\n")` builds a `&'static str` at compile time, so the handler can return `&'static str`. In the test, compare `body.trim()` with `env!("CARGO_PKG_VERSION")`.

</details>

<details>
<summary>Answer</summary>

The route and the handler:

```rust
        .route("/version", get(version))
```

```rust
async fn version() -> &'static str {
    concat!(env!("CARGO_PKG_VERSION"), "\n")
}
```

The test, in the `tests` module:

```rust
    #[tokio::test]
    async fn version_matches_cargo_toml() {
        let (status, body) = get_text("/version").await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body.trim(), env!("CARGO_PKG_VERSION"));
    }
```

```text
running 3 tests
test tests::version_matches_cargo_toml ... ok
test tests::healthz_says_ok ... ok
test tests::hello_names_the_pod_and_the_version ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the version is part of the binary, so `/version` tells you exactly which build a pod runs. During a rolling update, a script can call `/version` to see the old and the new version side by side.

</details>

### Exercise 3 — Broken resources (hard)

Make two separate broken changes to `k8s/deployment.yaml`, apply each one, find out with kubectl what is wrong, and undo it:

1. `requests.cpu: "100"` (one hundred whole cores).
2. `requests.memory: 2Mi` and `limits.memory: 2Mi`.

For each, say which component refuses, what status the new pod gets, and why the service keeps working.

<details>
<summary>Hint</summary>

Look at `kubectl get pods -l app=hello` and at the events in `kubectl describe pod` of the new pod. Think about who reads requests (the scheduler) and who enforces limits (the kernel, via the kubelet and the container runtime).

</details>

<details>
<summary>Answer</summary>

**1. CPU request too big.** The new pod stays `Pending`:

```text
NAME                     READY   STATUS    RESTARTS   AGE
hello-64bccb4655-5wkr6   1/1     Running   0          13s
hello-64bccb4655-qjppq   1/1     Running   0          16s
hello-64bccb4655-v294n   1/1     Running   0          16s
hello-695865b764-jptf7   0/1     Pending   0          8s
```

```text
  Warning  FailedScheduling  8s (x3 over 8s)  default-scheduler  0/1 nodes are available: 1 Insufficient cpu. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling.
```

The **scheduler** refuses: no node has 100 free cores, so the pod is never placed. Requests are a promise, and the scheduler only places a pod where the promise can be kept.

**2. Memory limit too small.** The new pod is placed, but its container cannot even start:

```text
hello-6b77c7bb66-fhfs8   0/1     RunContainerError   2 (10s ago)   26s
```

```text
      Message:      failed to create containerd task: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: container init was OOM-killed (memory limit too low?)
```

The **kernel** killed the container's first process for going over 2 Mi while it was being set up (an OOM kill), and the kubelet keeps retrying with growing delays. OCI (Open Container Initiative) is the standard for container images and runtimes; `runc` is the program that starts the container.

In both cases the service kept working, because `maxUnavailable: 0` stopped no old pod before a new one was ready. Undo by putting the old values back into the file and applying it (or, in a hurry, `kubectl rollout undo deployment/hello`).

</details>

## Quiz

1. Why does the service bind to `0.0.0.0` and not to `127.0.0.1`?

<details>
<summary>Answer</summary>

`127.0.0.1` accepts connections only from inside the same container (the same network namespace). The Service forwards connections to the pod's IP (Internet Protocol) address, which works only if the program listens on all interfaces.

</details>

2. What does `kind load docker-image` do, and why is it needed?

<details>
<summary>Answer</summary>

It copies an image from Docker on your Mac into the image store of the kind node's container runtime. Without it the node would try to pull the image from a registry, where your local image does not exist.

</details>

3. With `replicas: 3`, `maxSurge: 1` and `maxUnavailable: 0`, how many pods exist at most during an update, and how many are ready at least?

<details>
<summary>Answer</summary>

At most 4 pods (3 + 1 surge), and at least 3 ready pods (3 − 0 unavailable).

</details>

4. What is the difference between a resource request and a resource limit for memory?

<details>
<summary>Answer</summary>

The request is reserved for the pod; the scheduler uses it to choose a node. The limit is a ceiling enforced at run time: a container that goes above its memory limit is killed (OOM kill).

</details>

5. Why is `kubectl rollout undo` only an emergency tool when you work with files in Git?

<details>
<summary>Answer</summary>

It changes the cluster but not the files. The cluster then disagrees with Git, and the next `kubectl apply` of the files would bring the bad version back. Revert the change in Git and apply the files.

</details>

6. List the steps Kubernetes takes to stop a pod, and say what happens to a program that ignores `SIGTERM`.

<details>
<summary>Answer</summary>

The pod is marked Terminating and removed from the Service's endpoints; `SIGTERM` goes to process 1 of each container; Kubernetes waits up to the grace period (30 s by default); then it sends `SIGKILL`. A program that ignores `SIGTERM` runs until the grace period ends and is then killed, losing the requests in flight.

</details>

## Summary

- A cluster-friendly service reads its port from the environment, binds to `0.0.0.0`, logs to stdout (standard output) and handles `SIGTERM` with a graceful shutdown.
- Build a small, non-root image (multi-stage, distroless, exec-form `ENTRYPOINT`) with an exact tag that matches the version; build with `--locked` and update `Cargo.lock` before you build.
- kind nodes do not see your local Docker images: `kind load docker-image` copies them in.
- A Deployment for your service sets requests (for scheduling), a memory limit (a hard ceiling), a strict security context and a rolling-update strategy; a Service with a named target port sits in front of it.
- A rolling update creates a new ReplicaSet and moves pods across; with `maxUnavailable: 0` a broken update stalls instead of breaking the service.
- Roll back by applying the previous file; `kubectl rollout undo` is the emergency button.
- Stopping a pod: Terminating, removed from endpoints, `SIGTERM`, grace period, `SIGKILL`.

## Sources and further reading

- [Kubernetes documentation — Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) — rolling updates, `maxSurge`, `maxUnavailable`, history and rollbacks.
- [Kubernetes documentation — Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) — requests, limits and units.
- [Kubernetes documentation — Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) — every field of `securityContext`.
- [Kubernetes documentation — Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) — the exact termination sequence and the grace period.
- [Kubernetes documentation — Images](https://kubernetes.io/docs/concepts/containers/images/) — image names, tags and `imagePullPolicy`.
- [kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/) — the section "Loading an Image Into Your Cluster".
- [distroless images](https://github.com/GoogleContainerTools/distroless) — what the images contain and which variants exist.
- [axum — `serve` module](https://docs.rs/axum/latest/axum/serve/index.html) — `serve` and `with_graceful_shutdown`.

---

[🏠 Start](../START-HERE.md) · [← 21.3 A local cluster with kind and kubectl](03-a-local-cluster-with-kind.md) · [📋 Phase 21](README.md) · [21.5 ConfigMaps, Secrets and health probes →](05-configmaps-secrets-and-probes.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
