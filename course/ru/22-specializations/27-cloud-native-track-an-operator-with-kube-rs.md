<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.26 Трек cloud native: операторы Kubernetes](26-cloud-native-track-kubernetes-operators.md) · [📋 Этап 22](README.md) · [22.28 Трек cloud native: тестирование и запуск операторов →](28-cloud-native-track-testing-and-running-operators.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/27-cloud-native-track-an-operator-with-kube-rs.md) <!-- nav -->

# Урок 22.27 — Трек cloud native: оператор на kube-rs

> **Этап 22 — Специализации** · Урок 27 из 30 · ⏱ около 2,5 часа (2 занятия)

## Что вы узнаете

- Как описать пользовательский ресурс структурой Rust с `#[derive(CustomResource)]` и сгенерировать его CRD.
- Как оператор читает и записывает объекты через `kube::Client` и `Api<T>`.
- Как ссылки на владельца заставляют Kubernetes удалять дочерние объекты за вас (сборка мусора).
- Как написать функцию согласования с server-side apply и обновлением статуса.
- Как `Controller` из kube-runtime превращает эту функцию в работающий цикл управления.

## Перед началом

- Вы прошли [Урок 22.26 — Трек cloud native: операторы Kubernetes](26-cloud-native-track-kubernetes-operators.md).
  Кластер kind `course-22-operators` работает (если вы его удалили, создайте заново, как показано там), а
  `kubectl config current-context` выводит `kind-course-22-operators`.
- Вы знаете serde и `thiserror` ([Урок 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md)),
  derive-макросы ([Урок 06.8](../06-rust-advanced/08-procedural-macros.md)) и `Arc`
  ([Урок 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md)).
- Вы знаете асинхронные функции и Tokio ([Урок 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)).

## Новые слова

| Слово | Значение |
|---|---|
| **kube-rs** | Проект Rust для Kubernetes. Его главный крейт `kube` реэкспортирует клиент (`kube-client`), рантайм контроллеров (`kube-runtime`) и derive-макросы (`kube-derive`). |
| **`Client`** | Соединение с API-сервером (API — Application Programming Interface, программный интерфейс). `Client::try_default()` читает ваш kubeconfig на Mac или сервисный аккаунт пода внутри кластера. |
| **`Api<T>`** | Дескриптор для одного типа объектов в одном пространстве имён (или во всех): `get`, `list`, `patch`, `delete`, … |
| **подресурс status** (status subresource) | Отдельная часть пользовательского ресурса, `/status`, в которую пишет только оператор. Пользователи пишут `spec`, оператор отчитывается в `status`. |
| **ссылка на владельца** (owner reference) | Поле в метаданных дочернего объекта, которое называет его владельца (вид, имя и uid — уникальный идентификатор, который API-сервер даёт каждому объекту). Когда владельца удаляют, Kubernetes удаляет и дочерний объект. |
| **сборка мусора** (garbage collection) | Контроллер Kubernetes, который удаляет объекты, чьего владельца больше нет. |
| **SSA** (Server-Side Apply, применение на стороне сервера) | Способ записать объект, отправив *весь желаемый объект*. API-сервер сливает его с сохранённым и запоминает, какими полями владеете вы. `kubectl apply --server-side` делает то же самое. |
| **менеджер полей** (field manager) | Имя, которым пишущий подписывает свои изменения в SSA (здесь `greeter-operator`), чтобы API-сервер знал, кто владеет каким полем. |
| **merge patch** | Частичное обновление: вы отправляете только поля, которые хотите изменить, в виде JSON (JavaScript Object Notation, нотация объектов JavaScript). |
| **повторная постановка в очередь** (requeue) | Просьба к контроллеру снова запустить согласование для этого объекта через некоторое время, даже если ничего не изменилось. |
| **политика ошибок** (error policy) | Функция, которая решает, что делать, когда согласование вернуло ошибку; обычно «попробовать снова через несколько секунд». |
| **наблюдение / watcher** (watch) | Долгий запрос к API, который передаёт потоком каждое изменение некоторых объектов. `watcher` — обёртка над ним из kube-runtime. |
| **reflector / хранилище** (store) | Локальная копия наблюдаемых объектов внутри оператора, которую watcher держит актуальной. Контроллер читает из неё, а не спрашивает каждый раз API-сервер. |
| **CRD** (CustomResourceDefinition, определение пользовательского ресурса) | Объект, который добавляет новый тип в API Kubernetes (Урок 22.26). |

## 1. Проект

Оператор из этого урока для каждого `Greeting` создаёт ConfigMap `<name>-greeting` с сообщением внутри. Это
симуляция из Урока 22.26, но теперь с настоящим API-сервером.

```bash
cd ~/projects/rust
cargo new code/p22-greeter-operator
cd code/p22-greeter-operator
mkdir -p src/bin k8s
```

Добавьте зависимости в `Cargo.toml`:

```toml
[dependencies]
kube = { version = "5.1", features = ["runtime", "derive"] }
k8s-openapi = { version = "0.29", features = ["latest", "schemars"] }
schemars = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde-saphyr = "2"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
futures = "0.3"
thiserror = "2"
```

Фича `schemars` крейта `k8s-openapi` позволяет schemars описывать и встроенные типы. `serde-saphyr` пишет YAML (YAML Ain't Markup
Language, формат манифестов; старый крейт `serde_yaml` устарел (deprecated)). В проекте три исходных файла:

```text
code/p22-greeter-operator/
├── Cargo.toml
├── k8s/                  ← манифесты (CRD, примеры Greeting)
└── src/
    ├── lib.rs            ← тип Greeting и чистая логика «что должно существовать»
    ├── main.rs           ← оператор: согласование + контроллер
    └── bin/crdgen.rs     ← крошечная программа, которая печатает CRD в YAML
```

📏 **Соглашение:** держите *чистую* часть (типы, «построить желаемый объект») в `lib.rs`, без сетевых вызовов. Её
используют оба бинарника, и её можно тестировать модульными тестами без кластера (вы сделаете это в Упражнении 2).

## 2. Тип Greeting на Rust

В Уроке 22.26 вы писали схему вручную на YAML. Теперь единственный источник истины — структура Rust.
Напишите `src/lib.rs`:

```rust
use std::collections::BTreeMap;

use k8s_openapi::api::core::v1::ConfigMap;
use kube::api::ObjectMeta;
use kube::{CustomResource, Resource, ResourceExt};
use schemars::JsonSchema;
use serde::{Deserialize, Serialize};

/// The `spec` of a Greeting: what the user wants.
#[derive(CustomResource, Clone, Debug, Deserialize, Serialize, JsonSchema)]
#[kube(group = "course.example.com", version = "v1", kind = "Greeting")]
#[kube(namespaced, shortname = "gr", status = "GreetingStatus")]
#[kube(printcolumn(name = "Message", type_ = "string", json_path = ".spec.message"))]
pub struct GreetingSpec {
    /// The text to put into the ConfigMap.
    #[schemars(length(min = 1, max = 100))]
    pub message: String,
    /// Write the message in capital letters.
    #[serde(default)]
    pub shout: bool,
}

/// The `status` of a Greeting: what the operator did.
#[derive(Clone, Debug, Default, Deserialize, Serialize, JsonSchema)]
#[serde(rename_all = "camelCase")]
pub struct GreetingStatus {
    pub config_map: Option<String>,
}

/// The name of the ConfigMap that belongs to a Greeting.
pub fn config_map_name(greeting: &Greeting) -> String {
    format!("{}-greeting", greeting.name_any())
}

/// Builds the ConfigMap that should exist for this Greeting.
pub fn desired_config_map(greeting: &Greeting) -> ConfigMap {
    let text = if greeting.spec.shout {
        greeting.spec.message.to_uppercase()
    } else {
        greeting.spec.message.clone()
    };
    ConfigMap {
        metadata: ObjectMeta {
            name: Some(config_map_name(greeting)),
            namespace: greeting.namespace(),
            owner_references: greeting.controller_owner_ref(&()).map(|r| vec![r]),
            ..ObjectMeta::default()
        },
        data: Some(BTreeMap::from([("message".to_string(), text)])),
        ..ConfigMap::default()
    }
}
```

**Что происходит:**

1. Вы пишете только `GreetingSpec`. `#[derive(CustomResource)]` **генерирует** вторую структуру, `Greeting`, — весь
   объект: `metadata` (имя, пространство имён, uid, метки, …), ваш `spec` и `status`. Ещё он даёт ей методы
   `Greeting::crd()` и `Greeting::new(name, spec)`.
2. Строки `#[kube(...)]` несут ту же информацию, что и YAML CRD: группу, версию, вид, `namespaced`, короткое имя и
   столбец вывода. `status = "GreetingStatus"` добавляет подресурс status.
3. `JsonSchema` (из schemars) строит схему OpenAPI из структуры. Doc-комментарии (`///`) становятся описаниями полей,
   `#[schemars(length(min = 1, max = 100))]` становится `minLength`/`maxLength`, а `#[serde(default)]` делает `shout`
   необязательным со значением по умолчанию `false`.
4. `#[serde(rename_all = "camelCase")]` превращает имя Rust `config_map` в `configMap` — обычный для Kubernetes стиль
   полей JSON.
5. `desired_config_map` строит ConfigMap, который *должен* существовать. Она не обращается к кластеру. `ConfigMap` и
   `ObjectMeta` — структуры из `k8s-openapi`; `..ObjectMeta::default()` заполняет все остальные поля значением `None`.
6. `controller_owner_ref(&())` строит ссылку на владельца, указывающую на этот Greeting (подробнее в разделе 3). `&()`
   означает «без дополнительной информации о типе» — так всегда для типов, известных во время компиляции.

Сгенерированный `Greeting` выглядит так:

```text
Greeting
├── metadata: ObjectMeta              ← имя, пространство имён, uid, метки, ownerReferences, …
├── spec:     GreetingSpec            ← ваша структура
└── status:   Option<GreetingStatus>  ← None, пока оператор его не запишет
```

Теперь крошечный генератор CRD, `src/bin/crdgen.rs`:

```rust
use kube::CustomResourceExt;
use p22_greeter_operator::Greeting;

fn main() {
    let crd = Greeting::crd();
    let yaml = serde_saphyr::to_string(&crd).expect("the CRD can be written as YAML");
    print!("{yaml}");
}
```

Сгенерируйте CRD, посмотрите на него и примените (он заменит написанный вручную в Уроке 22.26):

```bash
cargo run -q --bin crdgen > k8s/greeting-crd.yaml
cat k8s/greeting-crd.yaml
kubectl apply -f k8s/greeting-crd.yaml
```

```text
…
              shout:
                default: false
                description: Write the message in capital letters.
                type: boolean
            required:
            - message
            type: object
          status:
            description: "The `status` of a Greeting: what the operator did."
            nullable: true
            properties:
              configMap:
                nullable: true
                type: string
            type: object
        required:
        - spec
        title: Greeting
        type: object
    served: true
    storage: true
    subresources:
      status: {}
customresourcedefinition.apiextensions.k8s.io/greetings.course.example.com configured
```

Выше показан конец файла (`…` отмечает пропущенную часть). Это CRD из Урока 22.26 плюс три вещи: описания из ваших
doc-комментариев, часть схемы для `status` и `subresources: status: {}`. `configured` означает, что существующий CRD
обновлён.

> 🧪 **Тестирование:** закоммитьте `k8s/greeting-crd.yaml` в Git. Когда меняете структуру, снова запустите `crdgen`;
> diff в Git точно покажет, как изменился API вашего оператора. Задание CI (continuous integration, непрерывная
> интеграция, Урок 09.12) может падать, если закоммиченный файл устарел.

## 3. Ссылки на владельца и сборка мусора

Кто удалит ConfigMap, когда пользователь удалит Greeting? Не ваш оператор. ConfigMap несёт **ссылку на владельца** в
`metadata.ownerReferences`: `apiVersion`, `kind`, `name` и `uid` владельца (поэтому она указывает именно на этот
объект, а не на любой Greeting с именем `hello`) плюс `controller: true` (этот владелец ею управляет). Вы увидите
её в настоящей ConfigMap в разделе 6.

Когда Greeting удаляют, **сборщик мусора** Kubernetes видит ConfigMap без владельца и удаляет её. Это работает,
даже если ваш оператор в этот момент не запущен. Два правила: владелец и дочерний объект должны быть в одном
пространстве имён, и владелец уже должен существовать (ему нужен `uid`) — внутри согласования это всегда так.

## 4. Функция согласования

Теперь `src/main.rs`. Сначала функция согласования и политика ошибок:

```rust
use std::sync::Arc;
use std::time::Duration;

use futures::StreamExt;
use k8s_openapi::api::core::v1::ConfigMap;
use kube::api::{Patch, PatchParams};
use kube::runtime::controller::Action;
use kube::runtime::{Controller, watcher};
use kube::{Api, Client, ResourceExt};
use p22_greeter_operator::{Greeting, GreetingStatus, desired_config_map};

/// The name we sign our changes with (the "field manager").
const MANAGER: &str = "greeter-operator";

#[derive(Debug, thiserror::Error)]
enum Error {
    #[error("Kubernetes API error: {0}")]
    Kube(#[from] kube::Error),
}

/// Things every reconcile call needs.
struct Context {
    client: Client,
}

async fn reconcile(greeting: Arc<Greeting>, ctx: Arc<Context>) -> Result<Action, Error> {
    let ns = greeting
        .namespace()
        .unwrap_or_else(|| "default".to_string());
    let name = greeting.name_any();
    println!("reconcile Greeting {ns}/{name}");

    // 1. Make the ConfigMap look exactly like we want it.
    let cm = desired_config_map(&greeting);
    let cm_name = cm.name_any();
    let config_maps: Api<ConfigMap> = Api::namespaced(ctx.client.clone(), &ns);
    let params = PatchParams::apply(MANAGER).force();
    config_maps
        .patch(&cm_name, &params, &Patch::Apply(&cm))
        .await?;

    // 2. Tell the user what we did, in the status.
    let greetings: Api<Greeting> = Api::namespaced(ctx.client.clone(), &ns);
    let status = GreetingStatus {
        config_map: Some(cm_name),
    };
    let patch = serde_json::json!({ "status": status });
    greetings
        .patch_status(&name, &PatchParams::default(), &Patch::Merge(&patch))
        .await?;

    // 3. Look again in 5 minutes, even if nothing changes.
    Ok(Action::requeue(Duration::from_secs(300)))
}

fn error_policy(greeting: Arc<Greeting>, error: &Error, _ctx: Arc<Context>) -> Action {
    eprintln!("reconcile of {} failed: {error}", greeting.name_any());
    Action::requeue(Duration::from_secs(5))
}
```

**Что происходит:**

1. `reconcile` получает Greeting как `Arc<Greeting>` (контроллер делится одной копией из своего хранилища) и общий
   `Context` с `Client`. Возвращает либо `Action` (что делать дальше), либо нашу `Error`.
2. `Api::namespaced(client, &ns)` — дескриптор для ConfigMap в пространстве имён Greeting. Клонировать `Client` дёшево:
   все клоны делят один пул соединений.
3. `Patch::Apply` с `PatchParams::apply(MANAGER)` — это **server-side apply**. Мы отправляем *всю* желаемую ConfigMap.
   Если её нет, API-сервер её создаёт; если она отличается — обновляет; если совпадает — ничего не меняется. Это
   ровно идемпотентное «сделай так» из Урока 22.26, которое за нас выполняет API-сервер. `.force()` забирает поля,
   которые изменил другой менеджер (скажем, человек через `kubectl edit`): оператор побеждает.
4. `patch_status` пишет только подресурс status с помощью **merge patch** (`Patch::Merge`): только JSON
   `{"status": {"configMap": "hello-greeting"}}`. Пользователи не могут изменить `status` обычной записью в Greeting.
5. `Action::requeue(300 s)` просит ещё одно согласование через пять минут. Это ловит расхождения, о которых не сообщает
   ни одно наблюдение.
6. `?` после каждого вызова превращает `kube::Error` в нашу `Error` (благодаря `#[from]`) и выходит раньше. Тогда
   контроллер вызывает `error_policy`, которая пишет в лог и просит повторить через пять секунд.

> ⚡ **Производительность:** согласование строит полный желаемый объект и позволяет API-серверу сравнить. В нашем коде
> нет шага «прочитать, сравнить, потом решить», поэтому один вызов делает всю работу и нет гонки между чтением и
> записью.

## 5. Контроллер

Остальная часть `src/main.rs` запускает цикл управления:

```rust
#[tokio::main]
async fn main() -> Result<(), kube::Error> {
    let client = Client::try_default().await?;
    let greetings: Api<Greeting> = Api::all(client.clone());
    let config_maps: Api<ConfigMap> = Api::all(client.clone());

    println!("greeter-operator started");
    Controller::new(greetings, watcher::Config::default())
        .owns(config_maps, watcher::Config::default())
        .shutdown_on_signal()
        .run(reconcile, error_policy, Arc::new(Context { client }))
        .for_each(|result| async move {
            match result {
                Ok((object, _action)) => println!("done: {}", object.name),
                Err(error) => eprintln!("controller error: {error}"),
            }
        })
        .await;
    println!("greeter-operator stopped");
    Ok(())
}
```

**Что происходит:**

1. `Client::try_default()` находит кластер: на вашем Mac читает kubeconfig и его текущий контекст; внутри пода
   использует сервисный аккаунт пода (Урок 22.28).
2. `Api::all` следит за Greeting во **всех** пространствах имён.
3. `Controller::new` запускает watcher для Greeting и держит их локальное хранилище (reflector). Каждое изменение
   Greeting ставит его в очередь на согласование.
4. `.owns(config_maps, …)` также следит за ConfigMap. Когда ConfigMap со ссылкой на владельца-Greeting меняется или
   исчезает, контроллер согласует **владельца**. Так удалённая вручную ConfigMap возвращается меньше чем за секунду.
5. `.shutdown_on_signal()` корректно останавливается по Ctrl+C (SIGINT) или SIGTERM: новые согласования не
   начинаются, работающие завершаются.
6. `.run(...)` возвращает *поток* (stream) результатов, по одному на каждое завершённое согласование. `for_each` его
   потребляет; программа живёт, пока живёт этот поток.

Контроллер также гарантирует, что согласование никогда не выполняется дважды **одновременно для одного объекта**, и
сливает много быстрых изменений одного объекта в одно согласование.

## 6. Запуск оператора

Оператор работает на вашем Mac и общается с кластером kind через ваш kubeconfig. Откройте **два терминала** в
`code/p22-greeter-operator`. В терминале 1:

```bash
cargo run
```

```text
     Running `target/debug/p22-greeter-operator`
greeter-operator started
```

В терминале 2 скопируйте Greeting `hello` из Урока 22.26 и примените его:

```bash
cp ../p22-reconcile-sim/k8s/hello.yaml k8s/
kubectl apply -f k8s/hello.yaml
kubectl get configmaps
kubectl get greeting hello -o jsonpath='{.status}'; echo
```

```text
greeting.course.example.com/hello created
NAME               DATA   AGE
hello-greeting     1      3s
kube-root-ca.crt   1      6m18s
{"configMap":"hello-greeting"}
```

ConfigMap существует, и статус об этом говорит. Посмотрите на ConfigMap через `kubectl get configmap hello-greeting -o yaml`:

```text
apiVersion: v1
data:
  message: Hello from a custom resource
kind: ConfigMap
metadata:
  creationTimestamp: "2026-11-18T07:50:06Z"
  name: hello-greeting
  namespace: default
  ownerReferences:
  - apiVersion: course.example.com/v1
    controller: true
    kind: Greeting
    name: hello
    uid: 2109072e-b11e-4ee3-b9c2-0638eed3357b
  resourceVersion: "1067"
  uid: a89b332d-f0d8-4e8b-a707-7610526e00e2
```

Теперь измените желание, сломайте что-нибудь и удалите:

```bash
kubectl patch greeting hello --type merge -p '{"spec":{"shout":true}}'
kubectl get configmap hello-greeting -o jsonpath='{.data.message}'; echo
kubectl delete configmap hello-greeting
kubectl get configmaps
kubectl delete greeting hello
kubectl get configmaps
```

```text
greeting.course.example.com/hello patched
HELLO FROM A CUSTOM RESOURCE
configmap "hello-greeting" deleted from default namespace
NAME               DATA   AGE
hello-greeting     1      2s
kube-root-ca.crt   1      6m22s
greeting.course.example.com "hello" deleted from default namespace
NAME               DATA   AGE
kube-root-ca.crt   1      6m25s
```

Обновление, самовосстановление (ConfigMap 2 секунды от роду: её создали заново) и сборка мусора — всё работает.
Нажмите Ctrl+C в терминале 1. Его полный вывод:

```text
greeter-operator started
reconcile Greeting default/hello
done: hello
reconcile Greeting default/hello
done: hello
reconcile Greeting default/hello
done: hello
reconcile Greeting default/hello
done: hello
reconcile Greeting default/hello
done: hello
reconcile Greeting default/hello
done: hello
controller error: tried to reconcile object Greeting.v1.course.example.com/hello.default that was not found in local store
greeter-operator stopped
```

**Что происходит:**

1. Согласований больше, чем команд. Наши собственные записи тоже вызывают события наблюдения: создание ConfigMap
   (принадлежащего объекта) и запись статуса (изменение Greeting) запускают ещё по одному согласованию. Оно не
   находит, что менять, а API-сервер игнорирует записи, которые ничего не меняют, поэтому цикл успокаивается. Вот
   почему идемпотентность обязательна.
2. Строка `controller error` появляется после удаления: сборщик мусора удалил ConfigMap, контроллер хотел согласовать
   её владельца, но владельца уже не было в хранилище. Здесь это безвредно.
3. `greeter-operator stopped` показывает корректное завершение после Ctrl+C.

## Частые ошибки

> ⚠️ **Частая ошибка: запустить оператор до установки CRD.** Watcher не может получить список Greeting и повторяет
> попытки бесконечно:
>
> ```text
> controller error: event queue error: failed to perform initial object list: ApiError: 404 page not found: Failed to parse error data (Status { status: Some(Failure), code: 404, message: "404 page not found", metadata: None, reason: "Failed to parse error data", details: None })
> ```
>
> Сначала выполните `kubectl apply -f k8s/greeting-crd.yaml`. `404` означает «такого URL (API для Greeting) нет».

> ⚠️ **Частая ошибка: нет ссылки на владельца.** Если убрать `owner_references` из `desired_config_map`, всё вроде бы
> работает, но после удаления Greeting его ConfigMap остаётся навсегда, а `.owns()` не может связать изменение
> ConfigMap с её Greeting.

> ⚠️ **Частая ошибка: забыть `.owns(...)`.** Без него удалённая вручную ConfigMap вернётся только при следующем
> requeue, через пять минут. Ссылка на владельца нужна для удаления, `.owns()` — для наблюдения.

## Попробуйте сами

1. Запустите оператор в терминале 1 (`cargo run`).
2. В терминале 2 создайте второе пространство имён и Greeting в нём: `kubectl create namespace demo`, затем
   `kubectl apply -n demo -f k8s/hello.yaml`.
3. `kubectl get configmaps -n demo` показывает `hello-greeting`: один оператор обслуживает все пространства имён.
4. `kubectl edit configmap hello-greeting -n demo`, измените сообщение и сохраните. Меньше чем через секунду
   `kubectl get configmap hello-greeting -n demo -o jsonpath='{.data.message}'` снова показывает исходное сообщение:
   работают `.force()` и `.owns()`.
5. Приберитесь командой `kubectl delete namespace demo` (она удаляет всё внутри) и остановите оператор.

## Упражнения

### Упражнение 1 — Ещё один ключ (легко)

Добавьте в данные ConfigMap второй ключ `length`: число символов сохранённого текста в виде строки. Запустите
оператор и проверьте командой `kubectl get configmap hello-greeting -o jsonpath='{.data}'`.

<details>
<summary>Подсказка</summary>

Значения в данных ConfigMap — всегда строки. `text.chars().count()` считает символы; `.to_string()` превращает
число в `String`. Вычислите длину до того, как `text` будет перемещён в таблицу.

</details>

<details>
<summary>Ответ</summary>

В `desired_config_map`:

```rust
        data: Some(BTreeMap::from([
            ("length".to_string(), text.chars().count().to_string()),
            ("message".to_string(), text),
        ])),
```

```text
{"length":"28","message":"Hello from a custom resource"}
```

Почему это работает: массив строится слева направо, поэтому `text` ещё доступен, когда вычисляется длина, и
перемещается в таблицу только во второй паре. Server-side apply обновит существующую ConfigMap при следующем
согласовании.

</details>

### Упражнение 2 — Модульные тесты без кластера (средне)

Добавьте в `src/lib.rs` модуль `#[cfg(test)] mod tests` с тремя тестами для `desired_config_map`: имя и
пространство имён верны, а сообщение скопировано; `shout` даёт заглавные буквы; ссылка на владельца указывает на
Greeting (вид, имя, uid, `controller: true`). Запустите `cargo test --lib`.

<details>
<summary>Подсказка</summary>

`Greeting::new("hello", spec)` создаёт объект с пустыми `metadata`, кроме имени. Задайте
`greeting.metadata.namespace` и `greeting.metadata.uid` сами: без uid ссылки на владельца нет.

</details>

<details>
<summary>Ответ</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn greeting(message: &str, shout: bool) -> Greeting {
        let spec = GreetingSpec {
            message: message.to_string(),
            shout,
        };
        let mut greeting = Greeting::new("hello", spec);
        greeting.metadata.namespace = Some("default".to_string());
        greeting.metadata.uid = Some("1234".to_string());
        greeting
    }

    #[test]
    fn config_map_has_name_namespace_and_message() {
        let cm = desired_config_map(&greeting("Hi", false));
        assert_eq!(cm.metadata.name.as_deref(), Some("hello-greeting"));
        assert_eq!(cm.metadata.namespace.as_deref(), Some("default"));
        assert_eq!(cm.data.unwrap()["message"], "Hi");
    }

    #[test]
    fn shout_makes_capital_letters() {
        let cm = desired_config_map(&greeting("Hi", true));
        assert_eq!(cm.data.unwrap()["message"], "HI");
    }

    #[test]
    fn config_map_is_owned_by_the_greeting() {
        let cm = desired_config_map(&greeting("Hi", false));
        let owners = cm
            .metadata
            .owner_references
            .expect("owner reference is set");
        assert_eq!(owners.len(), 1);
        assert_eq!(owners[0].kind, "Greeting");
        assert_eq!(owners[0].name, "hello");
        assert_eq!(owners[0].uid, "1234");
        assert_eq!(owners[0].controller, Some(true));
    }
}
```

```text
running 3 tests
test tests::shout_makes_capital_letters ... ok
test tests::config_map_has_name_namespace_and_message ... ok
test tests::config_map_is_owned_by_the_greeting ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `desired_config_map` — чистая функция, поэтому тест лишь строит Greeting в памяти и проверяет
результат. Такие тесты выполняются за миллисекунды и не требуют кластера. (Тесты могут печататься в другом порядке:
они выполняются параллельно.)

</details>

### Упражнение 3 — Наблюдать одно пространство имён (сложно)

Команды часто запускают по одному оператору на пространство имён. Пусть оператор читает переменную окружения
`WATCH_NAMESPACE`: если она задана, наблюдать за Greeting и ConfigMap только в этом пространстве имён и печатать
`watching namespace <ns>`; иначе, как и раньше, наблюдать за всеми. Проверьте: запустите с
`WATCH_NAMESPACE=default cargo run`, создайте пространство имён `other` и примените `hello.yaml` в обоих; ConfigMap
появится только в `default`.

<details>
<summary>Подсказка</summary>

`std::env::var("WATCH_NAMESPACE")` возвращает `Result`. Обе ветки `match` должны давать одинаковые типы, поэтому
возвращайте из каждой ветки кортеж `(Api<Greeting>, Api<ConfigMap>)`.

</details>

<details>
<summary>Ответ</summary>

В `main` замените две строки с `Api::all`:

```rust
    let (greetings, config_maps) = match std::env::var("WATCH_NAMESPACE") {
        Ok(ns) => {
            println!("watching namespace {ns}");
            let greetings: Api<Greeting> = Api::namespaced(client.clone(), &ns);
            let config_maps: Api<ConfigMap> = Api::namespaced(client.clone(), &ns);
            (greetings, config_maps)
        }
        Err(_) => {
            println!("watching all namespaces");
            (Api::all(client.clone()), Api::all(client.clone()))
        }
    };
```

После `kubectl create namespace other` и применения `hello.yaml` в обоих пространствах имён
`kubectl get configmaps -n other` показывает только `kube-root-ca.crt`. Оператор напечатал `watching namespace default`
и согласовал только `default/hello`. Приберитесь: `kubectl delete namespace other` и `kubectl delete greeting hello`.

Почему это работает: `Api`, созданный через `namespaced`, отправляет запросы списка и наблюдения на URL (веб-адрес)
одного пространства имён, поэтому Greeting из других пространств имён никогда не доходят до контроллера. В Уроке 22.28
вы увидите сторону безопасности: такому оператору нужны права только в одном пространстве имён.

</details>

## Вопросы для самопроверки

1. Вы написали только `GreetingSpec`. Откуда берётся структура `Greeting` и что в ней?

<details>
<summary>Ответ</summary>

Её генерирует `#[derive(CustomResource)]`. В ней `metadata` (`ObjectMeta`), ваш `spec` и, благодаря
`status = "GreetingStatus"`, `Option<GreetingStatus>`. Она также реализует `Resource` и имеет `crd()` и `new()`.

</details>

2. Почему оператор использует server-side apply, а не «получить ConfigMap, сравнить, потом создать или обновить»?

<details>
<summary>Ответ</summary>

Один вызов apply идемпотентен: создать, обновить или ничего не менять решает API-сервер. «Прочитать, сравнить,
записать» — это несколько вызовов, и между чтением и записью возможна гонка с другими пишущими. SSA также
записывает оператор как менеджера полей, которые он задал.

</details>

3. Кто удалит `hello-greeting`, когда вы удалите Greeting `hello`, и должен ли для этого работать оператор?

<details>
<summary>Ответ</summary>

Сборщик мусора Kubernetes, потому что у ConfigMap есть ссылка на владельца-Greeting. Оператору работать не нужно.

</details>

4. Один `kubectl apply` вызвал несколько согласований `hello`. Это ошибка?

<details>
<summary>Ответ</summary>

Нет. Собственные записи оператора (ConfigMap, статус) тоже вызывают события наблюдения. Так как согласование
идемпотентно, лишние запуски ничего не меняют, и цикл успокаивается.

</details>

## Итоги

- `#[derive(CustomResource)]` на структуре `…Spec` генерирует полный тип ресурса и его CRD; маленький бинарник
  `crdgen` печатает CRD в YAML, так что единственный источник истины — Rust.
- `Client::try_default()` подключается через ваш kubeconfig; `Api::namespaced` и `Api::all` дают типизированные дескрипторы.
- Дочерние объекты несут **ссылку на владельца**; сборщик мусора удаляет их вместе с владельцем.
- Согласование строит желаемый объект и записывает его через **server-side apply** (идемпотентно), затем отчитывается
  в подресурсе **status** через merge patch и возвращает `Action` (например, `requeue`).
- `Controller::new(...).owns(...).run(reconcile, error_policy, ctx)` — это цикл управления: наблюдения, очередь, никаких
  параллельных согласований одного объекта, повторы через политику ошибок.
- Лишние согласования из-за ваших собственных записей — это нормально; идемпотентность делает их безвредными.

## Источники и что почитать

- [kube.rs — The Application](https://kube.rs/controllers/application/) — официальное пошаговое руководство по контроллеру на kube-rs.
- [kube.rs — The Reconciler](https://kube.rs/controllers/reconciler/) — как думать о согласовании, действиях и ошибках.
- [docs.rs — kube 5.1](https://docs.rs/kube/5.1.0/kube/) — справочник API: `Client`, `Api`, `CustomResource`, `runtime::Controller`.
- [Kubernetes — Server-Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/) — менеджеры полей, конфликты и принудительная запись.
- [Kubernetes — Owners and Dependents](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/) — ссылки на владельца и их правила.
- [Kubernetes — Garbage Collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/) — как удаляются зависимые объекты.
- [kube-rs/controller-rs](https://github.com/kube-rs/controller-rs) — полный пример оператора от мейнтейнеров kube-rs.

---

[🏠 Начало](../START-HERE.md) · [← 22.26 Трек cloud native: операторы Kubernetes](26-cloud-native-track-kubernetes-operators.md) · [📋 Этап 22](README.md) · [22.28 Трек cloud native: тестирование и запуск операторов →](28-cloud-native-track-testing-and-running-operators.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
