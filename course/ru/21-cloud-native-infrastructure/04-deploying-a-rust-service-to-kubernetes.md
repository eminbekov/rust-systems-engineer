<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 21.3 Локальный кластер с kind и kubectl](03-a-local-cluster-with-kind.md) · [📋 Этап 21](README.md) · [21.5 ConfigMap, Secret и пробы работоспособности →](05-configmaps-secrets-and-probes.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/21-cloud-native-infrastructure/04-deploying-a-rust-service-to-kubernetes.md) <!-- nav -->

# Урок 21.4 — Развёртывание сервиса на Rust в Kubernetes

> **Этап 21 — Облачная (cloud-native) инфраструктура** · Урок 4 из 14 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Как написать небольшой сервис на axum, который правильно ведёт себя в Kubernetes: берёт порт из окружения, пишет логи в стандартный вывод и корректно останавливается по `SIGTERM`.
- Как собрать для него маленький образ без root и загрузить образ в кластер kind без реестра.
- Как написать Deployment и Service для своей программы с запросами и лимитами ресурсов и строгим контекстом безопасности.
- Как постепенное (rolling) обновление заменяет поды по одному и как откатиться.
- Что именно происходит, когда Kubernetes останавливает под.

## Перед началом

- Вы прошли [Урок 21.3 — Локальный кластер с kind и kubectl](03-a-local-cluster-with-kind.md).
- Вы умеете писать сервис на axum и тестировать его маршрутизатор через `oneshot` ([Урок 16.3 — Веб-сервисы на axum](../16-backend-services/03-web-services-with-axum.md)).
- Вы знаете многоэтапные Dockerfile для Rust ([Урок 11.13](../11-linux-shell-and-containers/13-dockerfiles-for-rust.md)) и корректное завершение с Tokio ([Урок 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **реестр образов** (registry) | Сервер, который хранит образы, например Docker Hub или `ghcr.io` от GitHub. Обычно узлы скачивают (*pull*) образы из реестра. |
| **`imagePullPolicy`** | Когда kubelet скачивает образ: `IfNotPresent` (только если его нет на узле), `Always` или `Never`. |
| **distroless-образ** (distroless image) | Базовый образ, в котором есть только то, что нужно программе для работы (библиотека C, сертификаты, часовые пояса), без оболочки и без менеджера пакетов. |
| **постепенное обновление** (rolling update) | Замена подов Deployment по несколько штук, чтобы сервис продолжал работать во время обновления. |
| **`maxSurge` / `maxUnavailable`** | Два числа постепенного обновления: сколько подов может быть *сверх* желаемого числа и сколько может не хватать *до* него. |
| **ревизия** (revision) | Пронумерованная версия шаблона пода Deployment. Каждое изменение шаблона создаёт новую ревизию. |
| **откат** (rollback) | Возврат к более ранней ревизии. |
| **запрос ресурсов** (resource request) | Сколько CPU (Central Processing Unit, центральный процессор) и памяти контейнеру гарантировано. Планировщик размещает поды по их запросам. |
| **лимит ресурсов** (resource limit) | Максимум, который контейнер может использовать. Превысив лимит памяти, он будет убит. |
| **милликор** (millicore, `m`) | Одна тысячная ядра CPU: `50m` = 5% одного ядра. |
| **Mi** (мебибайт, mebibyte) | 1 048 576 байт (2²⁰). Kubernetes записывает память как `16Mi`, `1Gi`. |
| **OOM-kill** (Out Of Memory, нехватка памяти) | Ядро Linux останавливает процесс, превысивший свой лимит памяти. |
| **контекст безопасности** (security context) | Настройки пода или контейнера, ограничивающие, что ему можно: работать не от root, файловая система только для чтения, никаких дополнительных возможностей (capabilities) Linux. |
| **период ожидания** (grace period) | Время, которое Kubernetes ждёт после `SIGTERM`, прежде чем отправить `SIGKILL` (по умолчанию 30 секунд). |

## 1. Сервис, который подходит платформе

В Уроке 21.1 перечислено, чего платформа ждёт от сервиса. Вот небольшой сервис, который делает всё это. Создайте пакет:

```bash
cd ~/projects/rust
cargo new code/p21-hello
cd code/p21-hello
cargo add axum@0.8
cargo add tokio@1 --features macros,rt-multi-thread,net,signal
cargo add --dev tower@0.5 --features util
cargo add --dev http-body-util@0.1
```

Теперь в `Cargo.toml`:

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

Замените `src/main.rs` на:

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

**Что происходит:**

1. `GET /` отвечает именем пода и версией. `env!("CARGO_PKG_VERSION")` подставляется **во время компиляции** из `version` в `Cargo.toml`, поэтому программа всегда знает, какая она версия. Имя пода в каждом ответе покажет нам, как Service распределяет запросы.
2. `GET /healthz` отвечает `ok`. В Уроке 21.5 его будет вызывать Kubernetes.
3. Порт берётся из переменной окружения `PORT` (по умолчанию 8080), а имя пода — из `HOSTNAME`. Kubernetes в каждом контейнере записывает в `HOSTNAME` имя пода.
4. Слушатель привязывается к `0.0.0.0`, то есть «ко всем сетевым интерфейсам». `127.0.0.1` принимал бы соединения только изнутри того же контейнера, и Service не смог бы достучаться до программы.
5. `shutdown_signal` ждёт `SIGTERM` или Ctrl+C. `with_graceful_shutdown` заставляет axum, когда future завершится, перестать принимать новые соединения, дать текущим запросам закончиться и затем вернуться. Каждая строка лога идёт в стандартный вывод через `println!`.
6. Тесты отправляют запросы маршрутизатору в памяти, без сети (Урок 16.3). Ожидаемый текст использует тот же макрос `env!`, поэтому тест продолжит проходить после смены версии.

Запустите тесты, а затем сервис на другом порту, чтобы увидеть, как работает переменная `PORT`:

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

Во второй вкладке `curl -s http://localhost:8090/` печатает `hello from localhost (version 0.1.0)` (на вашем Mac `HOSTNAME` обычно не задана, поэтому используется значение по умолчанию). Нажмите Ctrl+C в первой вкладке:

```text
received Ctrl+C, shutting down
all connections closed, bye
```

## 2. Образ

Создайте `Dockerfile` в `code/p21-hello`:

```dockerfile
# Этап 1: собрать программу полным тулчейном Rust.
FROM rust:1.100 AS build
WORKDIR /src
COPY Cargo.toml Cargo.lock ./
COPY src ./src
RUN cargo build --release --locked

# Этап 2: маленький образ для запуска, в котором только программа.
FROM gcr.io/distroless/cc-debian13:nonroot
COPY --from=build /src/target/release/p21-hello /usr/local/bin/p21-hello
EXPOSE 8080
ENTRYPOINT ["/usr/local/bin/p21-hello"]
```

и `.dockerignore` из двух строк, `target` и `.git`, чтобы Docker не отправлял в сборку гигабайты результатов сборки.

**Что происходит:**

1. Первый этап использует официальный образ `rust:1.100` (Debian 13) и собирает релизную программу. `--locked` заставляет сборку упасть, если `Cargo.lock` не соответствует `Cargo.toml`, поэтому образ собирается ровно из тех версий зависимостей, которые вы тестировали.
2. Второй этап начинается с **distroless**-образа Google `cc-debian13`, в котором есть библиотека C, с которой компонуются программы на Rust, и больше ничего: ни оболочки, ни менеджера пакетов. Вариант `:nonroot` работает от пользователя с id 65532, а не от root.
3. `ENTRYPOINT` в форме JSON (JavaScript Object Notation, формат записи объектов JavaScript), или «exec»-форме, запускает нашу программу напрямую как процесс номер 1 контейнера. Kubernetes отправляет `SIGTERM` процессу 1, поэтому его получает наш обработчик. (В форме оболочки, `ENTRYPOINT p21-hello`, процессом 1 была бы оболочка, и она могла бы не передать сигнал дальше.)

Соберите и проверьте:

```bash
docker build -t course-p21-hello:0.1.0 .
docker image ls --filter reference=course-p21-hello --format 'table {{.Repository}}\t{{.Tag}}\t{{.Size}}'
```

```text
REPOSITORY         TAG       SIZE
course-p21-hello   0.1.0     56.4MB
```

Тег `0.1.0` совпадает с версией в `Cargo.toml`: один точный тег на версию, никогда не `latest`. (При желании сначала проверьте образ обычным Docker: `docker run --rm -p 127.0.0.1:8090:8080 course-p21-hello:0.1.0`, вызовите `curl` из другой вкладки и остановите через Ctrl+C.)

> 🔒 **Безопасность:** в distroless-образе нет оболочки, поэтому `docker exec ... sh` и `kubectl exec ... sh` в нём не работают. Это достоинство: атакующий, нашедший ошибку в вашей программе, не получит ни оболочки, ни инструментов, чтобы скачать что-то ещё. Для отладки используйте логи, метрики и `kubectl debug` (временный вспомогательный контейнер), а не оболочку в продакшен-образе.

## 3. Как доставить образ в кластер

Создайте кластер для этого урока:

```bash
kind create cluster --name course-21-04
kubectl wait --for=condition=Ready node --all --timeout=120s
```

Обычно узел скачивает образы из **реестра**. Ваш образ существует только в Docker на вашем Mac, а узел kind — отдельный контейнер со своим containerd и своим хранилищем образов. `kind load` копирует образ в узел:

```bash
kind load docker-image course-p21-hello:0.1.0 --name course-21-04
docker exec course-21-04-control-plane crictl images | grep p21-hello
```

```text
Image: "course-p21-hello:0.1.0" with ID "sha256:ab5aca63e76d8c263f1a75920e98a1d0fba8bb79c7ab071989ec99c80dbdc8dc" not yet present on node "course-21-04-control-plane", loading...
docker.io/library/course-p21-hello              0.1.0                135d15789c7a6       11.7MB
```

`crictl` — инструмент командной строки среды выполнения контейнеров внутри узла. Образ на месте, под полным именем `docker.io/library/course-p21-hello:0.1.0` (имя без реестра — сокращение для Docker Hub). В Уроке 21.7 конвейер CI (Continuous Integration, непрерывная интеграция) вместо этого отправляет образы в настоящий реестр.

## 4. Манифесты

Храните манифесты рядом с кодом, в `code/p21-hello/k8s/`. `k8s/deployment.yaml`:

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
      maxSurge: 1        # не больше 1 лишнего пода во время обновления
      maxUnavailable: 0  # никогда не меньше 3 готовых подов
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

**Что происходит:**

1. `strategy` описывает постепенное обновление: при `maxSurge: 1` и `maxUnavailable: 0` Kubernetes сначала запускает **один новый** под, ждёт его готовности, затем останавливает **один старый** и повторяет. Готовых подов всегда не меньше 3.
2. `imagePullPolicy: IfNotPresent` использует образ, который `kind load` положил на узел. (Для точного тега это и так значение по умолчанию; для `latest` по умолчанию `Always`, и тогда kubelet пошёл бы в Docker Hub и потерпел неудачу.)
3. У порта контейнера есть **имя** — `http`. Service использует `targetPort: http`, поэтому, если номер порта когда-нибудь поменяется, править придётся только Deployment.
4. `resources.requests` — то, что планировщик *резервирует* на узле для каждого пода: 50 милликоров CPU и 16 Mi памяти. `limits.memory: 64Mi` — жёсткий потолок: выше него ядро убивает контейнер (OOM-kill). Лимита CPU сознательно нет: лимит CPU не убивает, а замедляет программу (*троттлинг*, throttling), и многие команды предпочитают задавать только запрос.
5. `securityContext` следует принципу наименьших привилегий (Этап 10): kubelet откажется запускать контейнер, если тот будет работать от root, процесс никогда не сможет получить больше прав, корневая файловая система только для чтения (наша программа ничего не пишет на диск), а все возможности (capabilities) Linux сброшены.
6. В одном файле может быть несколько объектов, разделённых строкой `---`; здесь мы используем по файлу на объект, а `kubectl apply -f k8s/` применяет всю папку.

> 📏 **Соглашение:** храните манифесты в Git рядом с кодом, который они развёртывают (`k8s/` или `deploy/`), по одному объекту на файл, называя файл по виду объекта. В Уроке 21.6 эта папка превратится в чарт Helm и базу Kustomize.

## 5. Развернуть и поговорить с сервисом

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

Отправьте шесть запросов изнутри кластера через Service:

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

Каждый `wget` открывает новое соединение, и Service отправляет каждое соединение одному из трёх подов. Выбор случайный, поэтому ваш порядок будет другим. (Если строк меньше шести, начните команду с `sleep 2;`; Упражнение 1 объясняет почему.) С вашего Mac `kubectl port-forward service/hello 8088:80` работает, как в Уроке 21.3; он печатает `Forwarding from 127.0.0.1:8088 -> 8080` (kubectl превратил порт Service 80 в именованный порт контейнера) и всегда говорит с одним и тем же подом. Логи идут в стандартный вывод, поэтому `kubectl logs` их находит:

```bash
kubectl logs deployment/hello
```

```text
Found 3 pods, using pod/hello-64bccb4655-5z2fn
p21-hello 0.1.0 listening on port 8080
```

## 6. Постепенное обновление

Выпустите версию 0.2.0. Поменяйте `version = "0.1.0"` на `version = "0.2.0"` в `Cargo.toml`, затем **сначала запустите тесты**: это заодно обновит `Cargo.lock`, который нужен сборке с `--locked` (см. «Частые ошибки»). Затем соберите, загрузите и поменяйте тег образа в `k8s/deployment.yaml` на `course-p21-hello:0.2.0`:

```bash
cargo test
docker build -t course-p21-hello:0.2.0 .
kind load docker-image course-p21-hello:0.2.0 --name course-21-04
kubectl diff -f k8s/deployment.yaml
```

Интересная часть diff:

```text
-      - image: course-p21-hello:0.1.0
+      - image: course-p21-hello:0.2.0
```

Примените и следите за выкаткой:

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

(Каждая строка «Waiting» появлялась несколько раз; повторы опущены.) Теперь посмотрите на ReplicaSet и на историю:

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

Здесь работает двухуровневая схема из Урока 21.2. Deployment создал второй ReplicaSet (`7d689ffb7b`) для нового шаблона и перенёс количество, по одному поду за раз; старый ReplicaSet (`64bccb4655`) сохранён с 0 подов, чтобы откат был быстрым. Запросы теперь доходят до версии 0.2.0:

```text
hello from hello-7d689ffb7b-s49gl (version 0.2.0)
hello from hello-7d689ffb7b-jvw4p (version 0.2.0)
hello from hello-7d689ffb7b-s49gl (version 0.2.0)
```

## 7. Откат

Допустим, в 0.2.0 есть ошибка. Быстрая аварийная кнопка — `rollout undo`:

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

Старый ReplicaSet снова держит 3 пода. Внимательно прочитайте предупреждение: кластер теперь выполняет 0.1.0, а ваш файл в Git всё ещё говорит 0.2.0. Кластер и Git расходятся, и следующий `kubectl apply` вернёт 0.2.0. Поэтому `rollout undo` — только для аварий; после него верните и файл (в Git: `git revert` коммита, который поменял тег) и примените его. Повседневный способ отката — *это и есть* применение предыдущего файла.

## 8. Как Kubernetes останавливает под

Каждое обновление, уменьшение числа копий и обслуживание узла останавливают поды. Шаги:

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

Посмотрите на это на настоящем поде. Во второй вкладке следите за логами одного пода (`kubectl logs -f <под>`), затем удалите этот под и замерьте время:

```bash
/usr/bin/time -p kubectl delete pod hello-64bccb4655-b4krd
```

```text
pod "hello-64bccb4655-b4krd" deleted from default namespace
real 1.37
user 0.03
sys 0.01
```

Во второй вкладке появилось:

```text
p21-hello 0.1.0 listening on port 8080
received SIGTERM, shutting down
all connections closed, bye
```

Под исчез примерно за секунду, потому что наша программа корректно вышла по `SIGTERM`. Программа, игнорирующая `SIGTERM`, заставила бы `kubectl delete` ждать все 30 секунд, а потом была бы убита; из-за этого каждое постепенное обновление становится медленным и теряет запросы.

> ⚡ **Производительность:** шаги 1 и 2 происходят почти одновременно, а не один за другим. Короткое время другие узлы ещё могут отправлять новые соединения поду, который уже получил `SIGTERM`. Поэтому нагруженные сервисы после `SIGTERM` ждут несколько секунд, прежде чем перестать принимать соединения (или используют хук `preStop`, который ждёт). Для этого небольшого сервиса поведения по умолчанию достаточно.

## 9. Уборка

```bash
kubectl delete -f k8s/
kind delete cluster --name course-21-04
docker image rm course-p21-hello:0.1.0 course-p21-hello:0.2.0
```

Код и папку `k8s/` сохраните: Урок 21.5 их расширяет.

## Частые ошибки

**Поменять версию, но не `Cargo.lock`.** Если отредактировать `version` в `Cargo.toml` и сразу запустить `docker build`, сборка с `--locked` упадёт, потому что в файле блокировки всё ещё записана старая версия вашего собственного пакета:

```text
error: cannot update the lock file /src/Cargo.lock because --locked was passed to prevent this
help: to generate the lock file without accessing the network, remove the --locked flag and use --offline instead.
```

Исправление: сначала выполните на Mac `cargo test` (или `cargo build`); он обновит `Cargo.lock`. Коммитьте оба файла вместе.

**Забыть `kind load`.** У узла нет образа, и он пытается скачать его из Docker Hub, где его нет. Новый под остаётся в `ImagePullBackOff`, а его события говорят:

```text
Failed to pull image "course-p21-hello:0.2.0": failed to pull and unpack image "docker.io/library/course-p21-hello:0.2.0": failed to resolve reference "docker.io/library/course-p21-hello:0.2.0": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
```

Посмотрите, что тем временем сделало постепенное обновление:

```text
NAME                     READY   STATUS             RESTARTS   AGE
hello-64bccb4655-2wp99   1/1     Running            0          9m45s
hello-64bccb4655-692f7   1/1     Running            0          9m45s
hello-64bccb4655-fmn9n   1/1     Running            0          9m45s
hello-7d689ffb7b-f5q8v   0/1     ImagePullBackOff   0          9m30s
```

Благодаря `maxUnavailable: 0` ни один старый под не был остановлен: сервис продолжал работать на трёх подах версии 0.1.0. Сломанное обновление застревает, а не роняет сервис. Исправление: `kind load docker-image ...`; kubelet повторит попытку, и выкатка продолжится.

**Тег `latest`.** С тегом `latest` `imagePullPolicy` по умолчанию — `Always`, поэтому kubelet игнорирует загруженный образ и идёт в Docker Hub. Даже с реестром `latest` не позволяет сказать, какой код работает, и не даёт откатиться. Используйте точные теги.

## Попробуйте сами

1. Повторите разделы 3–6 на свежем кластере, но на этот раз во время постепенного обновления держите открытым `kubectl get pods -l app=hello --watch` во второй вкладке. Посчитайте, сколько подов существует одновременно в худший момент. (Ответ: 4, потому что `maxSurge: 1`.)
2. Поменяйте `maxUnavailable` на `1`, а `maxSurge` на `0`, примените и сделайте ещё одно обновление (0.3.0). Что изменилось в наблюдении?
3. Уберите за собой, как в разделе 9.

## Упражнения

### Упражнение 1 — Пять реплик (легко)

Увеличьте `hello` до 5 реплик и отправьте 20 запросов изнутри кластера. Посчитайте, сколько ответов дал каждый под.

<details>
<summary>Подсказка</summary>

`kubectl scale`, затем `kubectl run ... -- sh -c 'sleep 2; for i in $(seq 20); do wget -qO- http://hello; done'`, вывод которого на Mac передаётся в `sort | uniq -c`.

</details>

<details>
<summary>Ответ</summary>

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

Пришли все 20 ответов, распределённые по пяти подам. Распределение случайное для каждого соединения и не идеально ровное. Зачем `sleep 2`? `kubectl run -i` подключается к поду только после того, как под запустился. Без паузы первые ответы могут быть напечатаны раньше, чем kubectl начнёт слушать, и потеряются: в одном запуске без паузы пришло только 10 строк из 20. Верните число реплик: `kubectl scale deployment hello --replicas=3`.

</details>

### Упражнение 2 — Эндпоинт `/version` с тестом (средне)

Добавьте `GET /version`, который отвечает версией и переводом строки (например, `0.1.0`). Сначала напишите тест `version_matches_cargo_toml`, посмотрите, как он падает с `404`, затем сделайте так, чтобы он проходил.

<details>
<summary>Подсказка</summary>

`concat!(env!("CARGO_PKG_VERSION"), "\n")` собирает `&'static str` во время компиляции, поэтому обработчик может возвращать `&'static str`. В тесте сравнивайте `body.trim()` с `env!("CARGO_PKG_VERSION")`.

</details>

<details>
<summary>Ответ</summary>

Маршрут и обработчик:

```rust
        .route("/version", get(version))
```

```rust
async fn version() -> &'static str {
    concat!(env!("CARGO_PKG_VERSION"), "\n")
}
```

Тест в модуле `tests`:

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

Почему это работает: версия — часть программы, поэтому `/version` точно говорит, какую сборку выполняет под. Во время постепенного обновления скрипт может вызывать `/version`, чтобы видеть старую и новую версии рядом.

</details>

### Упражнение 3 — Сломанные ресурсы (сложно)

Сделайте два отдельных сломанных изменения в `k8s/deployment.yaml`, примените каждое, выясните с помощью kubectl, что не так, и отмените его:

1. `requests.cpu: "100"` (сто целых ядер).
2. `requests.memory: 2Mi` и `limits.memory: 2Mi`.

Для каждого скажите, какой компонент отказывает, какой статус получает новый под и почему сервис продолжает работать.

<details>
<summary>Подсказка</summary>

Смотрите на `kubectl get pods -l app=hello` и на события в `kubectl describe pod` нового пода. Подумайте, кто читает запросы (планировщик) и кто следит за лимитами (ядро, через kubelet и среду выполнения контейнеров).

</details>

<details>
<summary>Ответ</summary>

**1. Слишком большой запрос CPU.** Новый под остаётся в `Pending`:

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

Отказывает **планировщик**: ни на одном узле нет 100 свободных ядер, поэтому под так и не размещается. Запросы — это обещание, и планировщик размещает под только там, где обещание можно выполнить.

**2. Слишком маленький лимит памяти.** Под размещается, но его контейнер не может даже запуститься:

```text
hello-6b77c7bb66-fhfs8   0/1     RunContainerError   2 (10s ago)   26s
```

```text
      Message:      failed to create containerd task: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: container init was OOM-killed (memory limit too low?)
```

**Ядро** убило первый процесс контейнера за превышение 2 Mi ещё во время его подготовки (OOM-kill), а kubelet продолжает повторять попытки с растущими паузами. OCI (Open Container Initiative, открытая инициатива по контейнерам) — стандарт для образов и сред выполнения контейнеров; `runc` — программа, которая запускает контейнер.

В обоих случаях сервис продолжал работать, потому что `maxUnavailable: 0` не позволил остановить ни одного старого пода, пока новый не готов. Отменяйте, вернув старые значения в файл и применив его (или, в спешке, `kubectl rollout undo deployment/hello`).

</details>

## Вопросы для самопроверки

1. Почему сервис привязывается к `0.0.0.0`, а не к `127.0.0.1`?

<details>
<summary>Ответ</summary>

`127.0.0.1` принимает соединения только изнутри того же контейнера (того же сетевого пространства имён). Service передаёт соединения на IP-адрес пода (IP — Internet Protocol, межсетевой протокол), а это работает, только если программа слушает все интерфейсы.

</details>

2. Что делает `kind load docker-image` и зачем он нужен?

<details>
<summary>Ответ</summary>

Он копирует образ из Docker на вашем Mac в хранилище образов среды выполнения контейнеров узла kind. Без него узел попытался бы скачать образ из реестра, где вашего локального образа нет.

</details>

3. При `replicas: 3`, `maxSurge: 1` и `maxUnavailable: 0` сколько подов существует максимум во время обновления и сколько минимум готовы?

<details>
<summary>Ответ</summary>

Максимум 4 пода (3 + 1 сверх), и минимум 3 готовых (3 − 0 недоступных).

</details>

4. Чем запрос ресурсов отличается от лимита для памяти?

<details>
<summary>Ответ</summary>

Запрос резервируется для пода; по нему планировщик выбирает узел. Лимит — потолок, который соблюдается во время работы: контейнер, превысивший лимит памяти, убивается (OOM-kill).

</details>

5. Почему `kubectl rollout undo` — только аварийный инструмент, когда вы работаете с файлами в Git?

<details>
<summary>Ответ</summary>

Он меняет кластер, но не файлы. Кластер начинает расходиться с Git, и следующий `kubectl apply` файлов вернёт плохую версию. Отмените изменение в Git и примените файлы.

</details>

6. Перечислите шаги, которыми Kubernetes останавливает под, и скажите, что происходит с программой, игнорирующей `SIGTERM`.

<details>
<summary>Ответ</summary>

Под помечается как Terminating и удаляется из эндпоинтов Service; `SIGTERM` уходит процессу 1 каждого контейнера; Kubernetes ждёт до конца периода ожидания (по умолчанию 30 с); затем отправляет `SIGKILL`. Программа, игнорирующая `SIGTERM`, работает до конца периода ожидания, а потом её убивают, и запросы, которые она обрабатывала, теряются.

</details>

## Итоги

- Сервис, дружелюбный к кластеру, берёт порт из окружения, привязывается к `0.0.0.0`, пишет логи в stdout (стандартный вывод) и обрабатывает `SIGTERM` корректным завершением.
- Собирайте маленький образ без root (многоэтапная сборка, distroless, `ENTRYPOINT` в exec-форме) с точным тегом, совпадающим с версией; собирайте с `--locked` и обновляйте `Cargo.lock` до сборки.
- Узлы kind не видят ваших локальных образов Docker: `kind load docker-image` копирует их внутрь.
- Deployment вашего сервиса задаёт запросы (для планирования), лимит памяти (жёсткий потолок), строгий контекст безопасности и стратегию постепенного обновления; перед ним стоит Service с именованным целевым портом.
- Постепенное обновление создаёт новый ReplicaSet и переносит поды; с `maxUnavailable: 0` сломанное обновление застревает, а не ломает сервис.
- Откатывайтесь, применяя предыдущий файл; `kubectl rollout undo` — аварийная кнопка.
- Остановка пода: Terminating, удаление из эндпоинтов, `SIGTERM`, период ожидания, `SIGKILL`.

## Источники и что почитать

- [Документация Kubernetes — Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) — постепенные обновления, `maxSurge`, `maxUnavailable`, история и откаты.
- [Документация Kubernetes — Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) — запросы, лимиты и единицы измерения.
- [Документация Kubernetes — Configure a Security Context for a Pod or Container](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/) — каждое поле `securityContext`.
- [Документация Kubernetes — Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) — точная последовательность остановки и период ожидания.
- [Документация Kubernetes — Images](https://kubernetes.io/docs/concepts/containers/images/) — имена образов, теги и `imagePullPolicy`.
- [kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/) — раздел «Loading an Image Into Your Cluster».
- [distroless-образы](https://github.com/GoogleContainerTools/distroless) — что содержат образы и какие есть варианты.
- [axum — модуль `serve`](https://docs.rs/axum/latest/axum/serve/index.html) — `serve` и `with_graceful_shutdown`.

---

[🏠 Начало](../START-HERE.md) · [← 21.3 Локальный кластер с kind и kubectl](03-a-local-cluster-with-kind.md) · [📋 Этап 21](README.md) · [21.5 ConfigMap, Secret и пробы работоспособности →](05-configmaps-secrets-and-probes.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
