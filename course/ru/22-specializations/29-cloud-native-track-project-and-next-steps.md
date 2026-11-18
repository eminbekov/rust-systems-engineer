<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 22.28 Трек cloud native: тестирование и запуск операторов](28-cloud-native-track-testing-and-running-operators.md) · [📋 Этап 22](README.md) · [22.30 Контрольная точка: ваша специализация →](30-checkpoint-your-specialization.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/22-specializations/29-cloud-native-track-project-and-next-steps.md) <!-- nav -->

# Урок 22.29 — Трек cloud native: проект и следующие шаги

> **Этап 22 — Специализации** · Урок 29 из 30 · ⏱ около 20 часов (10 занятий)

## Что вы узнаете

- Как спроектировать оператор, который управляет несколькими встроенными объектами (ConfigMap, Deployment и Service)
  для одного пользовательского ресурса.
- Как сообщать полезный статус на основе принадлежащих вам объектов и как перезапускать поды, когда изменилась
  только ConfigMap.
- Как самостоятельно, веха за вехой, собрать, протестировать и выпустить такой оператор.
- Куда двигаться дальше в этом направлении: документация, книги, open-source-проекты на Rust и сообщества.

## Перед началом

- Вы прошли [Урок 22.28 — Трек cloud native: тестирование и запуск операторов](28-cloud-native-track-testing-and-running-operators.md).
  Этот проект использует всё из Уроков 22.26–22.28; держите `code/p22-greeter-operator` открытым как образец.
- Кластер kind `course-22-operators` работает. Оператор приветствий больше не нужен: удалите его Deployment командой
  `kubectl delete -f k8s/operator.yaml` (в `code/p22-greeter-operator`), чтобы сэкономить память.
- **Не ваше направление?** Переходите к [Уроку 22.30 — Контрольная точка: ваша специализация](30-checkpoint-your-specialization.md).

## Новые слова

| Слово | Значение |
|---|---|
| **дочерний объект** (child object) | Объект, который оператор создаёт для пользовательского ресурса и которым владеет через ссылку на владельца; здесь это ConfigMap, Deployment и Service. |
| **селектор меток** (label selector) | Правило вида «все поды с `app.kubernetes.io/name: hello`». Deployment с его помощью находит свои поды, Service — куда отправлять трафик. |
| **рекомендуемые метки** (recommended labels) | Ключи меток `app.kubernetes.io/...`, которые предлагает документация Kubernetes, например `name` и `managed-by`. Инструменты их понимают. |
| **выкатка** (rollout) | Пошаговая замена подов Deployment новыми после изменения шаблона пода. |
| **аннотация шаблона пода** (pod template annotation) | Заметка в `spec.template.metadata.annotations` у Deployment. Её изменение меняет шаблон, и начинается выкатка. |
| **проброс порта** (port-forward) | `kubectl port-forward` связывает порт на вашем Mac с Service или подом в кластере — для проверки без ingress (точки входа для внешнего веб-трафика). |
| **фаза** (phase) | Короткое слово в статусе, которое подытоживает состояние; здесь `Pending` или `Ready`. |
| **CRD** (CustomResourceDefinition, определение пользовательского ресурса) | Объект, который добавляет новый тип в API Kubernetes (Урок 22.26). |
| **RBAC** (Role-Based Access Control, управление доступом на основе ролей) | Система прав Kubernetes (Урок 22.28). |
| **HTML** (HyperText Markup Language, язык гипертекстовой разметки) | Текстовый формат веб-страниц, например `<h1>Hello</h1>`. |

## 1. Проектирование оператора

Прежде чем писать код, решите, **что должно существовать** для одного пользовательского ресурса. Это решение и есть
весь оператор; остальное — та же механика, что и в операторе приветствий.

```text
StaticSite default/hello               (spec: html, replicas)
│
├── ConfigMap  hello-html              data: index.html = spec.html
├── Deployment hello                   replicas = spec.replicas, image nginx:1.30-alpine,
│                                      mounts hello-html at /usr/share/nginx/html
└── Service    hello                   port 80 → the pods of the Deployment

status: readyReplicas (from the Deployment), phase (Pending/Ready), url
```

Четыре проектных решения стоит записать в README:

1. **Имена выводятся** из имени StaticSite (`hello`, `hello-html`). Оператору никогда не нужно хранить, какие объекты
   он создал: он всегда может вычислить их заново. Именно это делает согласование «срабатывающим по уровню».
2. **Метки связывают всё вместе.** Каждый дочерний объект получает `app.kubernetes.io/name: <site>` и
   `app.kubernetes.io/managed-by: site-operator`. Их используют селекторы Deployment и Service. Тогда
   `kubectl get deploy,svc,cm -l app.kubernetes.io/managed-by=site-operator` показывает всё, что создал оператор.
3. **Выкатка при изменении содержимого.** Поды читают файлы из смонтированной ConfigMap. Когда меняется только
   ConfigMap, Deployment не меняется, и новые поды не стартуют (kubelet — агент на каждом узле, который запускает
   поды, — обновляет смонтированные файлы с задержкой, но вы не видите когда). Стандартный приём: положить хеш HTML в
   **аннотацию шаблона пода**. Новый HTML → новый хеш → новый шаблон пода → выкатка.
4. **Статус берётся из дочерних объектов.** `status.readyReplicas` у Deployment говорит, сколько подов обслуживают
   трафик. Оператор копирует это в статус StaticSite, так что пользователь всегда смотрит только на один объект.

> 📏 **Соглашение:** соглашения API Kubernetes (API conventions) говорят: пользователи пишут `spec`, контроллеры пишут
> `status`, а статус описывает то, что *есть*, а не то, чего хочется. Прочитайте их, прежде чем проектировать API
> побольше (ссылка в источниках).

## Проект

### Что вы создадите

`code/p22-site-operator`: оператор для пользовательского ресурса `StaticSite`. Примените этот файл — и получите
работающий сайт из двух подов за Service:

```yaml
apiVersion: course.example.com/v1
kind: StaticSite
metadata:
  name: hello
spec:
  replicas: 2
  html: |
    <h1>Hello from a StaticSite</h1>
```

### Контракт

**Пользовательский ресурс** — группа `course.example.com`, версия `v1`, вид `StaticSite`, в пространстве имён,
короткое имя `site`:

| Поле | Тип | Правила |
|---|---|---|
| `spec.html` | строка | обязательное, от 1 до 10 000 символов; становится `index.html` |
| `spec.replicas` | целое | необязательное, по умолчанию `1`, от `0` до `5` |
| `status.readyReplicas` | целое | копируется из Deployment |
| `status.phase` | строка | `Ready`, когда `readyReplicas >= spec.replicas`, иначе `Pending` |
| `status.url` | строка | `http://<name>.<namespace>.svc.cluster.local` |

`kubectl get staticsites` показывает столбцы `REPLICAS`, `READY` и `PHASE`.

**Дочерние объекты** StaticSite `<name>` — все в его пространстве имён, все со ссылкой на владельца на него и с
метками `app.kubernetes.io/name: <name>` и `app.kubernetes.io/managed-by: site-operator`:

| Вид | Имя | Содержимое |
|---|---|---|
| ConfigMap | `<name>-html` | ключ `index.html` = `spec.html` |
| Deployment | `<name>` | `spec.replicas` подов с образом `nginx:1.30-alpine`, порт 80, ConfigMap смонтирована только для чтения в `/usr/share/nginx/html`, аннотация шаблона пода `course.example.com/html-hash` = хеш `spec.html` |
| Service | `<name>` | тип ClusterIP, порт 80, селектор = метки выше |

**Поведение:** создание, изменение или удаление StaticSite отражается в его дочерних объектах за секунды; удаление
или ручная правка дочернего объекта исправляется; удаление StaticSite удаляет все дочерние объекты.

**Структура проекта и планка качества** (как в каждом проекте курса): `src/lib.rs` (типы и чистые функции
`desired_*` с модульными тестами), `src/main.rs` (согласование и контроллер), `src/bin/crdgen.rs`, `tests/e2e.rs`
(с `#[ignore]`), `k8s/` (CRD, пример, манифесты оператора), `Dockerfile`, `README.md`. `cargo fmt`, `cargo clippy
--all-targets` без предупреждений, `cargo test` зелёный. Один коммит Git на каждую веху.

### Пошаговый план

#### M1 — Пользовательский ресурс и его CRD

**Цель:** `StaticSiteSpec` и `StaticSiteStatus` в `src/lib.rs`, `crdgen` и CRD в кластере.

**Подсказки к работе:** скопируйте `Cargo.toml` из оператора приветствий (переименуйте пакет). Значению по умолчанию,
которое не `false`/`0`, нужна функция: `#[serde(default = "one")]` с `fn one() -> i32 { 1 }`. Ограничения чисел
задаются через `#[schemars(range(min = 0, max = 5))]`. Напишите два примера манифестов: `k8s/hello-site.yaml` (выше)
и `k8s/too-many.yaml` с `replicas: 9`.

<details>
<summary>Подсказка</summary>

Три столбца вывода, два из них целочисленные: если компилятор предупреждает `duplicated attribute` на втором
`printcolumn(..., type_ = "integer", ...)`, запишите этот столбец в старой форме «сырого JSON», которую kube-derive
по-прежнему принимает: `printcolumn = r#"{"name":"Ready","type":"integer","jsonPath":".status.readyReplicas"}"#`.
Сгенерированный CRD получается тем же.

</details>

**Готово, когда:**

```bash
cargo run -q --bin crdgen > k8s/staticsite-crd.yaml
kubectl apply -f k8s/staticsite-crd.yaml
kubectl apply -f k8s/too-many.yaml
```

```text
customresourcedefinition.apiextensions.k8s.io/staticsites.course.example.com created
The StaticSite "too-many" is invalid: spec.replicas: Invalid value: 9: spec.replicas in body should be less than or equal to 5
```

#### M2 — Чистые построители с модульными тестами

**Цель:** `desired_config_map`, `desired_deployment`, `desired_service` и `status_for(site, ready_replicas)` в
`src/lib.rs`, без единого сетевого вызова.

**Подсказки к работе:** стройте объекты из структур `k8s-openapi` (`Deployment`, `DeploymentSpec`, `PodTemplateSpec`,
`PodSpec`, `Container`, `Volume`, `ConfigMapVolumeSource`, `Service`, `ServiceSpec`, `ServicePort`) и
`..Default::default()` для всего, что вы не задаёте. Напишите одну вспомогательную функцию, которая строит общий для
всех дочерних объектов `ObjectMeta` (имя, пространство имён, метки, ссылка на владельца). Для хеша достаточно
`std::hash::DefaultHasher`: отформатируйте `finish()` в шестнадцатеричном виде через `format!("{:016x}", …)`.

<details>
<summary>Подсказка</summary>

Прежде чем писать структуру, прочитайте типы полей на docs.rs: в `k8s-openapi` 0.29 `Deployment.spec` — это просто
`DeploymentSpec` (не `Option`), как и `DeploymentSpec.selector` и `DeploymentSpec.template`, а `Service.spec` — всё
ещё `Option<ServiceSpec>`. Если по ошибке обернуть такое поле в `Some(...)`, компилятор скажет
`error[E0308]: mismatched types`. Хорошие тесты: число реплик скопировано; том указывает на `<name>-html`; два разных
текста HTML дают две разные аннотации; селектор Service равен меткам; `status_for` даёт `Pending` и `Ready` при нужных
числах; дочерние объекты несут ссылку на владельца вида `StaticSite`.

</details>

**Готово, когда:** `cargo test --lib` заканчивается строкой `test result: ok. 6 passed` (или больше), а
`cargo clippy --all-targets` не печатает предупреждений.

#### M3 — Согласование трёх дочерних объектов

**Цель:** оператор, который создаёт и обновляет все дочерние объекты через server-side apply.

**Подсказки к работе:** согласование применяет ConfigMap, затем Deployment, затем Service, каждый с
`PatchParams::apply("site-operator").force()`. Контроллер наблюдает за StaticSite и вызывает `.owns(...)` для всех трёх
видов дочерних объектов. Сначала запускайте его с Mac через `cargo run`.

<details>
<summary>Подсказка</summary>

Применяйте ConfigMap **до** Deployment: поды, стартовавшие раньше своей ConfigMap, ждут в `ContainerCreating`, пока
она не появится. В обратном порядке всё тоже работает, только медленнее и с непонятными событиями.

</details>

**Готово, когда** (возраст и IP-адреса у вас другие; IP — Internet Protocol, межсетевой протокол):

```bash
kubectl apply -f k8s/hello-site.yaml
kubectl get deploy,svc,cm -l app.kubernetes.io/managed-by=site-operator
kubectl port-forward svc/hello 8080:80
```

```text
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello   2/2     2            2           10s

NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/hello   ClusterIP   10.96.49.218   <none>        80/TCP    10s

NAME                   DATA   AGE
configmap/hello-html   1      10s
```

и во втором терминале, пока работает проброс порта, `curl -s localhost:8080` выводит
`<h1>Hello from a StaticSite</h1>`.

#### M4 — Статус

**Цель:** StaticSite сообщает `readyReplicas`, `phase` и `url`.

**Подсказки к работе:** результат вызова apply для Deployment *и есть* сохранённый Deployment, вместе со `status`;
возьмите из него `ready_replicas` (оно равно `None`, пока ни один под не готов). Записывайте статус через
`patch_status` и merge patch, но только когда он отличается от `site.status`; для этого выведите `PartialEq` для типа
статуса.

<details>
<summary>Подсказка</summary>

Статус меняется, когда поды становятся готовыми, а это изменение **Deployment**. Так как контроллер вызывает `.owns()`
для Deployment, каждое такое изменение снова согласует StaticSite, и статус догоняет реальность без всякого таймера.

</details>

**Готово, когда:**

```bash
kubectl wait --for=jsonpath='{.status.phase}'=Ready staticsite/hello --timeout=180s
kubectl get staticsites
```

```text
staticsite.course.example.com/hello condition met
NAME    REPLICAS   READY   PHASE
hello   2          2       Ready
```

#### M5 — Изменения и самовосстановление

**Цель:** новый HTML доходит до браузера, удалённый Deployment возвращается, масштабирование работает.

**Подсказки к работе:** эта веха в основном проверяет то, что построили M2–M4. Если HTML не меняется, проверьте
аннотацию с хешем в `kubectl get deployment hello -o yaml`.

<details>
<summary>Подсказка</summary>

`kubectl rollout status deployment/hello` ждёт, пока выкатка завершится; запустите его сразу после патча HTML.

</details>

**Готово, когда:**

```bash
kubectl patch staticsite hello --type merge -p '{"spec":{"html":"<h1>Version 2</h1>\n"}}'
kubectl rollout status deployment/hello --timeout=120s
```

заканчивается строкой `deployment "hello" successfully rolled out`, а `curl -s localhost:8080` (через новый проброс
порта) выводит `<h1>Version 2</h1>`. Затем:

```bash
kubectl delete deployment hello
kubectl get deployments
kubectl patch staticsite hello --type merge -p '{"spec":{"replicas":3}}'
kubectl wait --for=jsonpath='{.status.readyReplicas}'=3 staticsite/hello --timeout=120s
```

```text
deployment.apps "hello" deleted from default namespace
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
hello   2/2     2            2           3s
staticsite.course.example.com/hello patched
staticsite.course.example.com/hello condition met
```

#### M6 — Выпуск

**Цель:** оператор работает в кластере в пространстве имён `site-system`, с e2e-тестом и README.

**Подсказки к работе:** e2e-тест с `#[ignore]` создаёт StaticSite `e2e-site`, ждёт (через `await_condition` и
тайм-аут), пока его фаза станет `Ready`, удаляет его и ждёт, пока исчезнет Deployment `e2e-site`. Затем `Dockerfile`
как у оператора приветствий, образ `course-site-operator:0.1.0`, загруженный через `kind load docker-image`, и
`k8s/operator.yaml` с Namespace, ServiceAccount, ClusterRole, ClusterRoleBinding и Deployment.

<details>
<summary>Подсказка</summary>

Перечислите обращения к API, как в Упражнении 3 Урока 22.28: за StaticSite только наблюдают (`list`, `watch`), их
статус патчат (`staticsites/status`: `patch`), а за каждым видом дочерних объектов наблюдают и применяют его
(`list`, `watch`, `create`, `patch`). Deployment живут в группе API `apps`, ConfigMap и Service — в основной группе
`""`. Одно правило может перечислять несколько ресурсов: `resources: ["configmaps", "services"]`. Финализатора в этом
проекте нет, поэтому для StaticSite `patch` не нужен.

</details>

**Готово, когда:** оператор работает только в кластере (остановите свой `cargo run`):

- `kubectl get pods -n site-system` показывает один под в состоянии `Running`;
- `cargo test --test e2e -- --ignored` сообщает `1 passed`;
- `kubectl logs deployment/site-operator -n site-system | grep -ci forbidden` выводит `0`;
- после `kubectl delete staticsite hello` команда `kubectl get deploy,svc,cm -l app.kubernetes.io/managed-by=site-operator`
  выводит `No resources found in default namespace.`;
- `README.md` объясняет, что делает оператор, как его установить, содержит таблицу контракта и описывает, как
  запускать оба вида тестов.

### Дополнительные цели

- **Условия (conditions).** Замените `phase` списком стандартных условий (`type: Ready`, `status: "True"`, `reason`,
  `message`, `lastTransitionTime`), как описывают соглашения API.
- **События.** Публикуйте события Kubernetes (`kube::runtime::events::Recorder`), например «Rolled out new HTML»,
  чтобы `kubectl describe staticsite hello` рассказывал историю.
- **Более безопасный веб-сервер.** Запускайте nginx не от root, с корневой файловой системой только для чтения (тогда
  ему нужны записываемые тома `emptyDir` для кэша и PID-файла — файла с идентификатором процесса) и добавьте пробу
  готовности на порт 80.
- **Метрики.** Считайте согласования и ошибки и отдавайте их на `/metrics` для Prometheus (Урок 21.9).
- **Проверка через CEL** (Common Expression Language, язык правил API-сервера). Добавьте правило, что `html` должен
  содержать `<h1>`, через `#[x_kube(validation = ...)]`, и посмотрите, как API-сервер отклоняет страницу без заголовка.

## Следующие шаги в направлении cloud native

Теперь вы умеете собирать, тестировать и запускать настоящий оператор. Вот как двигаться дальше.

**Читайте** (в таком порядке):

1. [Документацию kube.rs](https://kube.rs/) до конца: руководство по контроллерам охватывает также оптимизацию
   (наблюдать меньше, кэшировать меньше), доступность (несколько реплик с выбором лидера) и наблюдаемость.
2. [Соглашения API](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
   Kubernetes: как должны выглядеть spec, status и conditions в любом API, которое вы проектируете.
3. [Operator White Paper](https://tag-app-delivery.cncf.io/whitepapers/operator/) от CNCF: что делает оператор хорошим,
   включая безопасность и лучшие практики.
4. Книги: *Programming Kubernetes* Майкла Хаузенбласа (Michael Hausenblas) и Штефана Шиманского (Stefan Schimanski) —
   как контроллеры и механизмы API устроены изнутри, и *Kubernetes Patterns* Билгина Ибряма (Bilgin Ibryam) и Роланда
   Хуса (Roland Huß) — паттерны оператора и контроллера среди многих других. Примеры в них не на Rust, но идеи
   переносятся без изменений.

**Читайте код** (Урок 24.3 показывает, как читать большую кодовую базу):

| Проект | Что это | Зачем читать |
|---|---|---|
| [kube-rs/kube](https://github.com/kube-rs/kube) | крейты, которыми вы пользовались | `kube-runtime/src/controller` и `finalizer.rs` короткие и хорошо прокомментированы |
| [kube-rs/controller-rs](https://github.com/kube-rs/controller-rs) | эталонный оператор | метрики, трассировка, события, тесты и Helm-чарт в одном небольшом репозитории |
| [stackabletech](https://github.com/stackabletech) | много продакшен-операторов на Rust (для Kafka, Spark, ZooKeeper, …) | настоящие операторы на kube-rs |
| [linkerd/linkerd2-proxy](https://github.com/linkerd/linkerd2-proxy) | прокси плоскости данных сервисной сетки Linkerd, на Rust | высокопроизводительная сеть внутри кластеров |
| [kubewarden/policy-server](https://github.com/kubewarden/policy-server) | движок политик Kubernetes на Rust, который выполняет политики как WebAssembly | контроль допуска (admission control) — ещё один способ расширить Kubernetes |
| [youki-dev/youki](https://github.com/youki-dev/youki) | среда выполнения контейнеров на Rust | что происходит ниже пода: пространства имён и cgroup из Этапа 13 |
| [project-akri/akri](https://github.com/project-akri/akri) | находит устройства на периферии (edge) и открывает их кластеру | проект CNCF с контроллерами и агентами на Rust |

**Вносите вклад** (Уроки 24.4 и 24.5 учат процессу): начните с задач с меткой
[good first issue в kube-rs](https://github.com/kube-rs/kube/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22),
исправлений документации kube.rs или примеров. Сначала прочитайте
[CONTRIBUTING.md](https://github.com/kube-rs/kube/blob/main/CONTRIBUTING.md) проекта; там также описаны правила
проекта для вкладов, созданных с помощью ИИ.

**Общайтесь:** канал `#kube` в [Discord Tokio](https://discord.gg/tokio) (ссылка есть на kube.rs),
[Slack Kubernetes](https://slack.k8s.io/) и [Slack CNCF](https://slack.cncf.io/). Группы SIG Kubernetes (special
interest groups, группы по интересам), например SIG API Machinery, проводят открытые видеовстречи.

**Что построить дальше:** оператор для того, что вы запускаете сами, например расписание резервных копий для
PostgreSQL из Этапа 16 (CronJob — объект, запускающий задание по расписанию, — плюс статус со временем последней копии),
или один из итоговых проектов Этапа 23, развёрнутый собственным оператором.

## Вопросы для самопроверки

1. Почему оператор выводит имена дочерних объектов из имени StaticSite, а не хранит сгенерированные имена?

<details>
<summary>Ответ</summary>

Потому что согласование должно уметь восстановить полное желаемое состояние по одному StaticSite — после перезапуска
и без памяти о прошлых запусках. Выведенные имена всегда известны; сохранённые могли бы потеряться или устареть.

</details>

2. Вы меняете только `spec.html`. Что без аннотации с хешем происходит с работающими подами и почему?

<details>
<summary>Ответ</summary>

ConfigMap меняется, а шаблон пода у Deployment — нет, поэтому выкатка не начинается. Поды продолжают работать и
видят новый файл только когда kubelet обновит смонтированную ConfigMap, через непредсказуемое время. Аннотация меняет
шаблон, и выкатка начинается сразу.

</details>

3. Как статус StaticSite узнаёт, что под стал готов, без таймера?

<details>
<summary>Ответ</summary>

Под, ставший готовым, меняет статус Deployment. Контроллер вызывает `.owns()` для Deployment, поэтому это изменение
запускает согласование StaticSite-владельца, который читает `readyReplicas` из Deployment и обновляет свой статус.

</details>

4. Какие глаголы RBAC нужны оператору сайтов для Deployment и зачем каждый?

<details>
<summary>Ответ</summary>

`list` и `watch` — для наблюдения через `.owns()`, `patch` — для server-side apply, а `create` — потому что apply
несуществующего Deployment создаёт его. Он никогда не удаляет (это делает сборщик мусора) и никогда не делает
отдельный `get`.

</details>

## Итоги

- Сначала проектирование: запишите, что должно существовать для одного пользовательского ресурса, с выводимыми
  именами и общими метками. Оператор — это такой проект плюс уже знакомая вам механика согласования.
- Один оператор может владеть несколькими видами дочерних объектов; вызывайте `.owns()` для каждого, чтобы изменения
  и удаления исправлялись сразу.
- Статус берётся из дочерних объектов; записывайте его, только когда он изменился.
- Хеш содержимого в аннотации шаблона пода превращает изменение ConfigMap в выкатку.
- Выпускайте с модульными тестами, e2e-тестом с `#[ignore]`, ClusterRole с наименьшими привилегиями и README.
- Дальше: руководство kube.rs, соглашения API, Operator White Paper, чтение кода kube-rs и других проектов на Rust и
  ваш первый вклад.

## Источники и что почитать

- [kube.rs — Optimization](https://kube.rs/controllers/optimization/) и [kube.rs — Availability](https://kube.rs/controllers/availability/) — следующие главы руководства по контроллерам.
- [Kubernetes — API conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) — spec, status и conditions.
- [CNCF — Operator White Paper](https://tag-app-delivery.cncf.io/whitepapers/operator/) — что делает хороший оператор.
- [Kubernetes — Extending Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/) — все способы расширить кластер, среди них операторы.
- [Kubernetes — Use Port Forwarding to Access Applications in a Cluster](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) — `kubectl port-forward` подробно.
- [docs.rs — k8s-openapi Deployment](https://docs.rs/k8s-openapi/0.29.0/k8s_openapi/api/apps/v1/struct.Deployment.html) — типы полей, которые вы строите в M2.
- [Docker Hub — nginx](https://hub.docker.com/_/nginx) — официальный образ, на котором работают сайты.

---

[🏠 Начало](../START-HERE.md) · [← 22.28 Трек cloud native: тестирование и запуск операторов](28-cloud-native-track-testing-and-running-operators.md) · [📋 Этап 22](README.md) · [22.30 Контрольная точка: ваша специализация →](30-checkpoint-your-specialization.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
