<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 21.4 Развёртывание сервиса на Rust в Kubernetes](04-deploying-a-rust-service-to-kubernetes.md) · [📋 Этап 21](README.md) · [21.6 Упаковка с Helm и Kustomize →](06-helm-and-kustomize.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/21-cloud-native-infrastructure/05-configmaps-secrets-and-probes.md) <!-- nav -->

# Урок 21.5 — ConfigMap, Secret и пробы работоспособности

> **Этап 21 — Облачная (cloud-native) инфраструктура** · Урок 5 из 14 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Как держать конфигурацию вне образа с помощью ConfigMap и передавать её сервису через переменные окружения.
- Как передать сервису секрет через Secret, смонтированный как файл, и почему Secret по умолчанию *не* зашифрован.
- Три вида проб (готовности, живости, запуска), что делает Kubernetes, когда каждая из них не проходит, и какой эндпоинт должна вызывать каждая.
- Как безопасно менять конфигурацию: почему работающие поды не видят изменённый ConfigMap и как это исправляет `kubectl rollout restart`.
- Как неправильная проба живости заставляет Kubernetes перезапускать здоровую программу.

## Перед началом

- Вы прошли [Урок 21.4 — Развёртывание сервиса на Rust в Kubernetes](04-deploying-a-rust-service-to-kubernetes.md), и у вас есть его папка `code/p21-hello` с `k8s/`.
- Вы знаете конфигурацию через окружение и разницу между «жив» и «готов» ([Урок 16.13 — Конфигурация, проверки работоспособности и контейнеры](../16-backend-services/13-configuration-health-checks-and-containers.md)) и bearer-токены ([Урок 16.8 — Аутентификация в веб-сервисе](../16-backend-services/08-authentication-in-a-web-service.md)).
- Вы знаете атомарные типы вроде `AtomicBool` ([Урок 14.5 — Атомарные операции](../14-concurrency-and-async/05-atomics.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **ConfigMap** | Объект Kubernetes с несекретной конфигурацией в виде пар «ключ-значение». Поды могут читать его как переменные окружения или как файлы. |
| **Secret** | Объект Kubernetes для небольших секретных значений (пароли, токены, ключи). Похож на ConfigMap, но обращаются с ним осторожнее: по умолчанию значения не показываются, и в etcd его можно шифровать. |
| **base64** | Способ записать любые байты 64 печатными символами. Это *кодирование*, а не шифрование: раскодировать может любой. |
| **проба** (probe) | Проверка, которую kubelet регулярно выполняет над контейнером: запрос HTTP (HyperText Transfer Protocol, протокол передачи гипертекста), соединение TCP (Transmission Control Protocol, протокол управления передачей) или команда. |
| **проба готовности** (readiness probe) | «Может ли этот под сейчас принимать трафик?» Если не проходит, под убирается из эндпоинтов Service, но продолжает работать. |
| **проба живости** (liveness probe) | «Работает ли программа вообще?» Если не проходит несколько раз подряд, kubelet убивает и перезапускает контейнер. |
| **проба запуска** (startup probe) | «Закончила ли программа запускаться?» Пока она не прошла, две другие пробы ждут. Для медленного старта. |
| **счётчик перезапусков** (restart count) | Сколько раз kubelet перезапускал контейнер пода; столбец `RESTARTS` в `kubectl get pods`. |
| **`rollout restart`** | Команда kubectl, которая заменяет все поды Deployment новыми (постепенное обновление без изменения кода). |
| **быстрый отказ** (fail fast) | Сразу остановиться с понятной ошибкой, когда не хватает чего-то важного, вместо того чтобы работать в сломанном состоянии. |

## 1. Почему конфигурация живёт вне образа

В Уроке 21.4 образ был всей программой. Но одной и той же программе в разных местах нужны разные настройки: тестовый кластер и продакшен используют разные приветствия, тайм-ауты, адреса баз данных и пароли. Если настройки внутри образа, нужен отдельный образ на каждое окружение, и образ, который вы тестировали, — не тот, что работает.

Правило cloud native (Урок 21.1 и Twelve-Factor App): **соберите один образ и настраивайте его снаружи**. Для этого в Kubernetes есть два объекта:

| | ConfigMap | Secret |
|---|---|---|
| Для чего | Обычные настройки: приветствие, уровень логирования, переключатели возможностей | Пароли, токены, ключи, сертификаты |
| Как показывает `kubectl get -o yaml` | Обычный текст | base64 (закодировано, не зашифровано) |
| Как хранится в etcd | Как есть | Как есть по умолчанию; можно включить шифрование при хранении |
| Как под его читает | Переменные окружения или файлы | Переменные окружения или файлы (файлы безопаснее) |
| Наибольший размер | 1 МиБ (MiB, мебибайт) | 1 МиБ |

## 2. Версия 0.3.0 сервиса

Новая версия читает ещё три настройки и предлагает ещё два эндпоинта:

- `GREETING` (из ConfigMap): первое слово ответа, по умолчанию `hello`.
- `STARTUP_DELAY_MS` (из ConfigMap): имитация прогрева, например загрузки кэша. Пока он не закончился, сервис жив, но **не готов**.
- `ADMIN_TOKEN_FILE`: путь к файлу с секретным токеном (из Secret). `GET /admin` отвечает только тем, кто присылает этот токен.
- `GET /readyz`: `200 ready` после прогрева, `503 warming up` до него.

В `code/p21-hello` поставьте `version = "0.3.0"` в `Cargo.toml`, добавьте фичу `time` в Tokio (`cargo add tokio@1 --features macros,rt-multi-thread,net,signal,time`) и замените `src/main.rs` на:

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

**Что происходит:**

1. В `AppState` три новых поля. `ready` — `Arc<AtomicBool>`: его делят задача прогрева и каждый обработчик запросов, читая и записывая без блокировки. `Ordering::Relaxed` достаточно, потому что это одиночный флаг, который не защищает никаких других данных (Урок 14.6).
2. `readyz` возвращает пару `(StatusCode, &str)`; axum превращает её в ответ с этим статусом. Kubernetes смотрит только на статус: 200–399 — успех, всё остальное — неудача.
3. `admin` читает заголовок `Authorization` и сравнивает его с `Bearer <токен>`. Нет заголовка или он другой — `401 Unauthorized`.
4. `main` читает токен **до** открытия порта. Если файла нет, программа печатает причину и завершается с кодом 1. Это **быстрый отказ**: под, который не может запуститься, показывает понятную ошибку в логах, а не работает, чтобы потом проваливать каждый запрос к admin.
5. Запущенная задача спит `STARTUP_DELAY_MS`, а затем устанавливает `ready`. Во время прогрева порт открыт, поэтому `/healthz` уже отвечает: программа *жива*, но не *готова*. Именно эту разницу измеряют две пробы.
6. Тесты собирают `AppState` вручную с фейковым токеном, поэтому им не нужны ни файлы, ни окружение.

> 🔒 **Безопасность:** сравнение токена через `==` заканчивается, как только один байт отличается, поэтому его время немного выдаёт, сколько байтов было верным. В настоящем сервисе сравнивайте секреты за постоянное время (например, крейтом `subtle`) и предпочитайте полноценную аутентификацию (Урок 16.8). Обратите внимание и на `text.trim()`: он убирает перевод строки, который большинство инструментов пишут в конце файла; если об этом забыть, получается классическая ошибка «токен правильный, но не работает».

Попробуйте на Mac. Создайте файл с токеном для разработки и держите его вне Git (добавьте `dev-token.txt` и `admin-token.txt` в `.gitignore`):

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

Во второй вкладке `curl -s -i http://localhost:8090/readyz` в первые две секунды печатает строку статуса `HTTP/1.1 503 Service Unavailable` и `warming up`; после них — `HTTP/1.1 200 OK` и `ready`. `curl -s -H 'Authorization: Bearer dev-token' http://localhost:8090/admin` печатает `welcome, admin`. Остановите сервис через Ctrl+C. Без `ADMIN_TOKEN_FILE` он отказывает сразу:

```text
cannot read the admin token from /etc/hello/admin-token: No such file or directory (os error 2)
```

## 3. ConfigMap

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

Значения в ConfigMap — всегда строки, поэтому числа взяты в кавычки. Deployment в разделе 5 читает все ключи как переменные окружения через `envFrom: - configMapRef: name: hello-config`. Есть ещё два способа, которые вы встретите в других проектах: одна переменная из одного ключа (`env: - name: X valueFrom: configMapKeyRef: ...`) и ConfigMap, смонтированный как папка, по файлу на ключ (удобно для целых файлов конфигурации вроде `config.toml`).

## 4. Secret

Создайте случайный токен и Secret из него. Не записывайте токен в YAML-файл (YAML Ain't Markup Language) в Git; создайте Secret прямо из файла:

```bash
openssl rand -hex 16 > admin-token.txt
kubectl create secret generic hello-admin --from-file=admin-token=admin-token.txt
```

(Сначала вы создадите кластер, в разделе 6; здесь команды показаны, чтобы их объяснить.) `generic` — обычный вид Secret; `--from-file=admin-token=admin-token.txt` создаёт один ключ `admin-token`, значение которого — содержимое файла. Теперь посмотрите, что хранит Kubernetes:

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

Значение выглядит нечитаемым, но это всего лишь **base64**. Любой, кто может читать Secret, раскодирует его:

```bash
kubectl get secret hello-admin -o jsonpath='{.data.admin-token}' | base64 -d
```

```text
cc129aa4da4b564c3cb14cb3ea7b45cc
```

Это ровно содержимое `admin-token.txt` (включая перевод строки в конце). Значит, Secret защищают **права на чтение** (правила доступа в сервере API, Application Programming Interface; принцип наименьших привилегий с Этапа 10), шифрование etcd, если администратор кластера его включил, и то, что секрет не попадает в Git, — а не base64.

Deployment в разделе 5 монтирует Secret как **папку только для чтения** (`volumes` и `volumeMounts`): каждый ключ становится файлом. Тогда в поде есть файл `/etc/hello/admin-token`, и на него указывает `ADMIN_TOKEN_FILE`. Файл безопаснее переменной окружения: переменные окружения легко раскрыть случайно (отчёт о сбое, печатающий окружение; дочерний процесс, который их наследует; `kubectl describe pod`, печатающий каждую переменную, заданную через `value:`), а смонтированные файлы kubelet может обновить при изменении Secret.

> 🔒 **Безопасность:** никогда не коммитьте манифест Secret с настоящими значениями, даже «всего лишь в base64». Настоящие команды хранят секреты в менеджере секретов или коммитят их только в зашифрованном виде; для этого есть инструменты Sealed Secrets, SOPS (Secrets OPerationS, «операции с секретами») и External Secrets Operator. Для этого курса достаточно `kubectl create secret` из файла, который есть в `.gitignore`.

## 5. Пробы: жив ли, готов ли?

kubelet на каждом узле может регулярно проверять каждый контейнер. Есть три вида проверок, и они отвечают на разные вопросы:

| Проба | Вопрос | Когда не проходит | Наш эндпоинт |
|---|---|---|---|
| **готовности** (readiness) | Может ли принимать трафик сейчас? | Убирается из эндпоинтов Service; продолжает работать; возвращается, когда проба снова проходит | `/readyz` |
| **живости** (liveness) | Работает ли вообще или завис? | После `failureThreshold` неудач: контейнер убивают и перезапускают | `/healthz` |
| **запуска** (startup) | Закончил ли запускаться? | Пока не пройдёт, пробы готовности и живости не выполняются; если не пройдёт никогда, контейнер перезапускают | (здесь не нужна) |

Полный `k8s/deployment.yaml` этого урока:

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

**Что происходит:**

1. `envFrom` превращает каждый ключ `hello-config` в переменную окружения; `env` добавляет ещё одну, `ADMIN_TOKEN_FILE`.
2. `volumes` объявляет том пода, сделанный из Secret; `volumeMounts` кладёт его в `/etc/hello` в контейнере. Поскольку корневая файловая система только для чтения (Урок 21.4), смонтированная папка — единственный способ дать программе файлы.
3. `readinessProbe` каждые 2 секунды отправляет `GET /readyz` на порт с именем `http`. Пока не придёт ответ 2xx или 3xx, под готов `0/1` и не получает трафика от Service.
4. `livenessProbe` каждые 10 секунд отправляет `GET /healthz`; после 3 неудач подряд (`failureThreshold`) kubelet перезапускает контейнер. Другие полезные поля: `initialDelaySeconds` (подождать перед первой проверкой) и `timeoutSeconds` (по умолчанию 1 секунда).
5. Тег образа — `0.3.0`; всё остальное не изменилось с Урока 21.4.

> ⚠️ **Частая ошибка:** проба живости, которая проверяет зависимости, например «могу ли я достучаться до базы данных?». Когда база недоступна, пробу живости проваливает *каждый* под, и Kubernetes по кругу перезапускает их все; это не чинит базу и добавляет нагрузку, когда она возвращается. Проба живости должна отвечать только на вопрос «завис ли этот процесс?». Зависимостям место в пробе готовности, и даже там осторожно (если не готов весь сервис, он вообще не получает трафика).

## 6. Развернуть и понаблюдать

Соберите образ, создайте кластер, загрузите образ, создайте Secret, примените:

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

(Вывод предыдущих команд такой же, как в Уроке 21.4 и разделе 4.) Посмотрите на поды сразу и ещё раз после выкатки:

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

`Running`, но `0/1`: контейнеры работают, но во время 3-секундного прогрева не готовы. События пода рассказывают, что произошло (подставьте имя своего пода):

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

(Показана только часть `Events`.) Три неудачи пробы готовности с `503`, затем успех. Это то, что нужно: во время прогрева ни один запрос не доходит до пода. Теперь, когда готовы все три, их перечисляет EndpointSlice, а логи программы показывают прогрев:

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

Вызовите сервис изнутри кластера, сначала без токена, затем с ним. `--env` передаёт токен во временный под как переменную, а благодаря одинарным кавычкам `$TOKEN` раскрывает оболочка пода, а не ваша:

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

Приветствие пришло из ConfigMap, токен — из Secret, и тот же образ работал бы в любом другом кластере с другими значениями.

## 7. Изменение конфигурации

Поменяйте `GREETING: "salam"` на `GREETING: "privet"` в `k8s/configmap.yaml` и примените только этот файл. Затем спросите сервис:

```bash
kubectl apply -f k8s/configmap.yaml
kubectl run client -q --rm -i --restart=Never --image=busybox:1.37 -- sh -c 'sleep 2; wget -qO- http://hello'
```

```text
configmap/hello-config configured
salam from hello-c6cf94684-b5hp4 (version 0.3.0)
```

Всё ещё `salam`. Переменные окружения задаются **один раз**, когда контейнер запускается; процесс не видит их последующих изменений. ConfigMap изменился, а шаблон пода Deployment — нет, поэтому ни один контроллер ничего не заменил. Попросите новые поды:

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

(Повторяющиеся строки «Waiting» опущены.) `rollout restart` добавляет в шаблон пода аннотацию с отметкой времени; это считается изменением шаблона, поэтому Deployment выполняет обычное постепенное обновление с проверками готовности и без простоя. Чарт Helm (Урок 21.6) может делать это автоматически: он кладёт в шаблон контрольную сумму ConfigMap, и тогда каждое изменение конфигурации меняет и шаблон.

> 💡 **Совет:** файлы смонтированного ConfigMap или Secret kubelet *обновляет* после изменения (с задержкой примерно до минуты), но только если программа перечитывает файл. Наша программа читает токен один раз при запуске, поэтому ей всё равно нужен перезапуск. Прочитать один раз и перезапускаться при изменении — проще и понятнее.

## 8. Когда проба живости неправильная

Сделайте реалистичную ошибку: поменяйте путь пробы живости в `k8s/deployment.yaml` с `/healthz` на `/nope` (эндпоинта не существует), примените и подождите около минуты:

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

Выкатка прошла (проба готовности проходит), но каждый под уже **перезапускался** один раз, и это будет повторяться примерно каждые 30 секунд. Последние события пода объясняют почему:

```text
  Normal   Killing    25s                kubelet            spec.containers{hello}: Container hello failed liveness probe, will be restarted
  Warning  Unhealthy  22s (x6 over 55s)  kubelet            spec.containers{hello}: Readiness probe failed: HTTP probe failed with statuscode: 503
  Warning  Unhealthy  5s (x5 over 45s)   kubelet            spec.containers{hello}: Liveness probe failed: HTTP probe failed with statuscode: 404
```

`404` от `/nope` три раза подряд — и kubelet убил контейнер. Логи **предыдущего** (убитого) контейнера показывают, что сама программа была совершенно здорова и просто получила `SIGTERM`:

```bash
kubectl logs hello-7d97657c47-49c7s --previous
```

```text
p21-hello 0.3.0 listening on port 8080
warm-up finished, ready for traffic
received SIGTERM, shutting down
all connections closed, bye
```

Это урок из реальной жизни: растущий счётчик `RESTARTS` при чистых логах обычно означает неправильную пробу, а не падающую программу. (Падающая программа показала бы панику в `--previous` и статус `CrashLoopBackOff`.) Верните путь `/healthz` и примените снова; постепенное обновление заменит поды, и перезапуски прекратятся.

## 9. Уборка

```bash
kind delete cluster --name course-21-05
docker image rm course-p21-hello:0.3.0
```

Держите `admin-token.txt` вне Git и сохраните код: Урок 21.6 упаковывает его.

## Частые ошибки

**Ждать, что изменение ConfigMap дойдёт до работающих подов.** Как показал раздел 7, переменные окружения в работающем процессе не меняются. Исправление: `kubectl rollout restart deployment/<имя>` или контрольная сумма конфигурации в шаблоне пода, которую добавляет Helm.

**Считать base64 защитой.** Манифест Secret в Git с `data: admin-token: Y2Mx...` — это утёкший секрет: `base64 -d` раскрывает его за секунду. Исправление: создавайте Secret из файлов, которые есть в `.gitignore`, или используйте инструмент для зашифрованных секретов.

**Отсутствующий секрет, обнаруженный поздно.** Сервис, который читает токен только при первом запросе к admin, запускается «нормально» и падает позже, в неудачный момент. Исправление: читайте и проверяйте каждую обязательную настройку при запуске и выходите с понятным сообщением, как делает здесь `main` (`cannot read the admin token from ...`). Тогда под никогда не станет готовым, и постепенное обновление остановится, не дойдя до пользователей.

## Попробуйте сами

1. Поставьте `STARTUP_DELAY_MS: "0"` в ConfigMap, примените, перезапустите Deployment и проверьте, что поды становятся готовыми почти сразу (событий `Readiness probe failed` нет).
2. Удалите Secret (`kubectl delete secret hello-admin`) и перезапустите Deployment. Новый под остаётся в `ContainerCreating`, а его события говорят `MountVolume.SetUp failed for volume "admin-secret" : secret "hello-admin" not found`. Почему сервис продолжает отвечать? (Потом создайте Secret снова; под запустится сам.)
3. Уберите за собой, как в разделе 9.

## Упражнения

### Упражнение 1 — Медленный прогрев (легко)

Поставьте `STARTUP_DELAY_MS` в `"10000"` (10 секунд), примените ConfigMap, выполните `kubectl rollout restart deployment/hello` и измерьте, сколько длится `kubectl rollout status`. Объясните число через `maxSurge: 1` и `maxUnavailable: 0`.

<details>
<summary>Подсказка</summary>

Используйте `/usr/bin/time -p kubectl rollout status deployment/hello`. Подумайте: сколько подов заменяется одновременно и сколько времени нужно каждому новому поду, чтобы стать готовым?

</details>

<details>
<summary>Ответ</summary>

```text
deployment "hello" successfully rolled out
real 32.84
user 0.04
sys 0.02
```

Около 33 секунд. При `maxSurge: 1` и `maxUnavailable: 0` Kubernetes заменяет по одному поду и ждёт, пока новый под станет **готовым**, прежде чем остановить старый. Каждому новому поду нужно около 10 секунд прогрева, плюс немного на запуск и на следующую проверку готовности (каждые 2 секунды): 3 пода × около 11 секунд ≈ 33 секунды. Более быстрый запуск ускоряет каждое обновление и каждое увеличение числа копий; больший `maxSurge` заменяет больше подов параллельно. Потом верните значение `"3000"`.

</details>

### Упражнение 2 — Тест эндпоинта готовности (средне)

Напишите тест `readyz_follows_the_ready_flag`: при `ready`, равном `false`, `/readyz` отвечает `503`; после установки `true` он отвечает `200` с телом `ready\n`.

<details>
<summary>Подсказка</summary>

`test_state()` даёт состояние с `ready: true`. Поскольку `ready` — это `Arc<AtomicBool>`, клон состояния делит тот же флаг: поменяйте его через `state.ready.store(false, Ordering::Relaxed)`, отправьте запрос с `state.clone()`, поменяйте снова, отправьте снова.

</details>

<details>
<summary>Ответ</summary>

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

Почему это работает: `Clone` у `AppState` клонирует `Arc`, а не `AtomicBool`, поэтому маршрутизатор и тест смотрят на один и тот же флаг. Это то же разделение, благодаря которому задача прогрева меняет то, что видят обработчики.

</details>

### Упражнение 3 — Перестать быть готовым до остановки (сложно)

Урок 21.4 упоминал гонку: сразу после `SIGTERM` ещё могут приходить соединения. Измените `shutdown_signal` так, чтобы после сигнала она сначала устанавливала `ready` в `false`, печатала строку, ждала 2 секунды и только потом позволяла axum остановиться. Выпустите это как версию 0.3.1, разверните, удалите один под, следя за его логами, и замерьте время удаления.

<details>
<summary>Подсказка</summary>

`shutdown_signal` нужен флаг: дайте ей параметр `ready: Arc<AtomicBool>`, а в `main` склонируйте `state.ready` в переменную **до** того, как `app(state)` переместит `state`. Помните про `cargo test` перед `docker build` (версия изменилась).

</details>

<details>
<summary>Ответ</summary>

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

В `main`:

```rust
    let ready_flag = state.ready.clone();
    axum::serve(listener, app(state))
        .with_graceful_shutdown(shutdown_signal(ready_flag))
```

После сборки, загрузки и `kubectl set image deployment/hello hello=course-p21-hello:0.3.1` удаление пода заняло около 3 секунд вместо 1:

```text
pod "hello-7bfbb55b66-447nr" deleted from default namespace
real 2.93
user 0.03
sys 0.02
```

а его логи показали новый порядок:

```text
p21-hello 0.3.1 listening on port 8080
warm-up finished, ready for traffic
received SIGTERM, shutting down
not ready any more, waiting 2 s for the endpoints to update
all connections closed, bye
```

Почему это работает: Kubernetes и так убирает завершающийся под из эндпоинтов, но это изменение доходит до kube-proxy на каждом узле не мгновенно. Во время 2-секундной паузы под всё ещё принимает и обслуживает опоздавшие соединения и сообщает «не готов» любой пробе. Только после этого axum перестаёт принимать соединения. Пауза должна быть намного меньше периода ожидания (30 с).

</details>

## Вопросы для самопроверки

1. Почему конфигурацию не стоит встраивать в образ?

<details>
<summary>Ответ</summary>

Потому что один и тот же образ должен работать в разных окружениях с разными настройками. Когда конфигурация снаружи (ConfigMap, Secret), образ, который вы тестировали, — ровно тот, что работает.

</details>

2. Зашифрован ли Secret в Kubernetes? Что на самом деле его защищает?

<details>
<summary>Ответ</summary>

По умолчанию нет: значения только закодированы в base64, а etcd хранит их как есть, если администратор не включил шифрование при хранении. Защищают управление доступом (кто может читать Secret), необязательное шифрование etcd и то, что секреты не попадают в Git.

</details>

3. Что происходит с подом, у которого не проходит проба готовности? А проба живости три раза подряд?

<details>
<summary>Ответ</summary>

Проба готовности: под убирается из эндпоинтов Service и не получает трафика, но продолжает работать. Проба живости (после `failureThreshold` раз): kubelet убивает контейнер и запускает его снова; счётчик перезапусков растёт.

</details>

4. Почему проба живости не должна проверять базу данных?

<details>
<summary>Ответ</summary>

Если база недоступна, пробу живости проваливает каждый под, и их перезапускают снова и снова. Перезапуск не чинит базу и добавляет нагрузку. Проба живости должна проверять только, не завис ли сам процесс.

</details>

5. Вы изменили ConfigMap, который используется через `envFrom`. Почему поды не видят изменения и что нужно выполнить?

<details>
<summary>Ответ</summary>

Переменные окружения задаются при запуске контейнера и в работающем процессе не меняются; шаблон пода не изменился, поэтому ничего не заменили. Выполните `kubectl rollout restart deployment/<имя>`.

</details>

6. Столбец `RESTARTS` ваших подов всё растёт, но `kubectl logs --previous` показывает чистое завершение после `SIGTERM`. Какая вероятная причина?

<details>
<summary>Ответ</summary>

Не проходящая проба живости (неверный путь, неверный порт, слишком короткий тайм-аут): kubelet убивает здоровую программу. Падающая программа показала бы в предыдущих логах панику или ошибку.

</details>

## Итоги

- Соберите один образ и настраивайте его снаружи: обычные настройки — в ConfigMap, секреты — в Secret.
- `envFrom` превращает ConfigMap в переменные окружения; Secret безопаснее монтировать как файлы только для чтения.
- base64 — не шифрование. Защищайте Secret управлением доступом, шифрованием etcd и тем, что не храните их в Git.
- Проба готовности решает судьбу трафика (не прошла → убран из эндпоинтов); проба живости решает о перезапусках (не прошла → контейнер перезапущен); проба запуска защищает медленный старт.
- Держите `/healthz` тривиальным; логику «могу ли я обслуживать сейчас?» кладите в `/readyz`; отказывайте сразу при запуске, если не хватает обязательной конфигурации.
- Работающие поды не видят изменённый ConfigMap в своём окружении: используйте `kubectl rollout restart`.
- Растущие перезапуски при чистых предыдущих логах указывают на неправильную пробу.

## Источники и что почитать

- [Документация Kubernetes — ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) — все способы использовать ConfigMap.
- [Документация Kubernetes — Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) — виды Secret и как их использовать.
- [Документация Kubernetes — Good practices for Kubernetes Secrets](https://kubernetes.io/docs/concepts/security/secrets-good-practices/) — управление доступом, шифрование при хранении и другие советы.
- [Документация Kubernetes — Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) — каждое поле проб с примерами.
- [Документация Kubernetes — kubectl rollout restart](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_rollout/kubectl_rollout_restart/) — справка по команде.
- [The Twelve-Factor App — III. Config](https://12factor.net/config) — почему конфигурации место в окружении.

---

[🏠 Начало](../START-HERE.md) · [← 21.4 Развёртывание сервиса на Rust в Kubernetes](04-deploying-a-rust-service-to-kubernetes.md) · [📋 Этап 21](README.md) · [21.6 Упаковка с Helm и Kustomize →](06-helm-and-kustomize.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
