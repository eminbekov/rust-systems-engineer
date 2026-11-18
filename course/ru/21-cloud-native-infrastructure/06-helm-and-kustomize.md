<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 21.5 ConfigMap, Secret и пробы работоспособности](05-configmaps-secrets-and-probes.md) · [📋 Этап 21](README.md) · [21.7 CI/CD: от коммита до продакшена →](07-ci-cd-from-commit-to-production.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/21-cloud-native-infrastructure/06-helm-and-kustomize.md) <!-- nav -->

# Урок 21.6 — Упаковка с Helm и Kustomize

> **Этап 21 — Облачная (cloud-native) инфраструктура** · Урок 6 из 14 · ⏱ около 3 часов (2 занятия)

## Что вы узнаете

- Почему обычными файлами манифестов становится трудно управлять, когда один сервис работает в нескольких окружениях.
- Как работает Helm: чарты, значения, шаблоны, релизы и ревизии. Вы напишете небольшой чарт для своего сервиса.
- Как проверить чарт до установки (`helm lint`, `helm template`) и как установить, обновить, откатить и удалить релиз.
- Как работает Kustomize: база плюс оверлеи, с патчами вместо шаблонов, встроенный в kubectl.
- Когда какой инструмент выбирать.

## Перед началом

- Вы прошли [Урок 21.5 — ConfigMap, Secret и пробы работоспособности](05-configmaps-secrets-and-probes.md), и у вас есть `code/p21-hello` версии 0.3.0 с папкой `k8s/` и файлом `admin-token.txt`.
- Вы помните рекомендуемые метки `app.kubernetes.io/*` из [Урока 21.2](02-kubernetes-concepts.md).

## Новые слова

| Слово | Значение |
|---|---|
| **Helm** | Менеджер пакетов для Kubernetes. Устанавливает набор манифестов как единое целое и помнит каждую его версию. |
| **чарт** (chart) | Пакет Helm: папка с `Chart.yaml`, `values.yaml` и шаблонами манифестов. |
| **значения** (values) | Настройки чарта (`values.yaml`), которые можно переопределить при установке через `--set` или `-f file.yaml`. |
| **шаблон** (template) | Манифест с заполнителями вроде `{{ .Values.replicaCount }}`, которые подставляет Helm. Helm использует язык шаблонов Go. |
| **релиз** (release) | Одна установленная копия чарта в кластере с именем (например, `hello`). Один чарт можно установить много раз под разными именами. |
| **ревизия** (revision, Helm) | Пронумерованная версия релиза. Каждая установка, обновление и откат создают новую. |
| **Kustomize** | Инструмент, встроенный в kubectl, который собирает манифесты из *базы* и *оверлеев*, которые её изменяют. Без шаблонов. |
| **база** (base) | В Kustomize — общий набор манифестов, который используют все окружения. |
| **оверлей** (overlay) | В Kustomize — папка для одного окружения, которая ссылается на базу и перечисляет изменения для этого окружения. |
| **генератор** (generator) | Возможность Kustomize создать ConfigMap или Secret и добавить к его имени хеш содержимого. |
| **окружение** (environment) | Одно место, где сервис работает со своими настройками, например *dev*, *staging* или *production*. |
| **JSON Schema** (JSON: JavaScript Object Notation, формат записи объектов JavaScript) | Стандартный способ описать, какие поля и каких типов может содержать документ JSON или YAML; Helm проверяет по нему значения. |

## 1. Проблема копий

После Урока 21.5 у вашего сервиса три файла манифестов на YAML (YAML Ain't Markup Language, «YAML — не язык разметки»). Теперь он нужен в двух окружениях: *dev* с 1 репликой и приветствием `privet` и *production* с 3 репликами и `salam`. У очевидного решения — двух копий папки — известные проблемы:

- Исправление (например, новую пробу) нужно вносить в каждую копию, и рано или поздно одну копию забудут.
- Ревьюеру трудно увидеть, чем *отличаются* dev и production.
- На вопрос «что именно установлено в кластере и что было установлено до этого?» нет ответа; `kubectl apply` не хранит историю всего набора.

Два инструмента решают это двумя разными способами:

```text
  Helm (templates)                         Kustomize (patches)

  values.yaml ─┐                           base/  (complete, valid YAML)
  --set ...  ──┼─► templates/ ──► YAML        │
  -f dev.yaml ─┘   {{ .Values.x }}           overlays/dev/   "replicas: 1,
                                             overlays/prod/   greeting: privet"
                                                   │
                                                   ▼
                                                  YAML
```

Helm заполняет заполнители в шаблонах; Kustomize начинает с готовых файлов и меняет некоторые поля. Оба в итоге выдают обычные манифесты, которые получает сервер API (Application Programming Interface, программный интерфейс).

## 2. Helm: установка и первый взгляд

Установите Helm через Homebrew и проверьте:

```bash
brew install helm
helm version --short
```

```text
v4.3.2+g8346aaa
```

В этом уроке используется Helm 4. Helm работает с кластером вашего текущего контекста kubectl, с тем же kubeconfig. В отличие от старых версий, у него нет компонента внутри кластера; историю каждого релиза он хранит как Secret в пространстве имён релиза.

Чарт — это папка. Создайте такую структуру внутри `code/p21-hello` (чарт называется `hello`):

```text
chart/hello/
├── Chart.yaml          ← имя и версии чарта
├── values.yaml         ← настройки по умолчанию
└── templates/
    ├── _helpers.tpl    ← именованные куски шаблонов (не манифест)
    ├── configmap.yaml
    ├── deployment.yaml
    └── service.yaml
```

> 💡 **Совет:** `helm create <имя>` создаёт полный пример чарта с гораздо большим числом файлов (Ingress, автомасштабирование, учётная запись сервиса, тесты). Это хороший образец, но для учёбы понятнее самим написать несколько файлов ниже.

`chart/hello/Chart.yaml`:

```yaml
apiVersion: v2
name: hello
description: The p21-hello service from the Rust course
type: application
version: 0.1.0
appVersion: "0.3.0"
```

`version` — версия **чарта** (упаковки); `appVersion` — версия **программы** внутри. Они меняются независимо: новая проба в чарте — новая версия чарта с той же программой.

`chart/hello/values.yaml`:

```yaml
# How many pods to run.
replicaCount: 3

image:
  repository: course-p21-hello
  # Empty means: use appVersion from Chart.yaml.
  tag: ""
  pullPolicy: IfNotPresent

# Configuration for the service (goes into the ConfigMap).
greeting: salam
startupDelayMs: 3000

# The Secret with the key "admin-token". It is created outside the chart.
adminSecretName: hello-admin

resources:
  requests:
    cpu: 50m
    memory: 16Mi
  limits:
    memory: 64Mi
```

Secret остаётся вне чарта и создаётся через `kubectl create secret`, как в Уроке 21.5: секретным значениям не место в `values.yaml`, который лежит в Git.

## 3. Шаблоны

`chart/hello/templates/_helpers.tpl` определяет два переиспользуемых куска:

```text
{{/* Labels that select the pods. They must never change after the first install. */}}
{{- define "hello.selectorLabels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/* All labels: the selector labels plus the version. */}}
{{- define "hello.labels" -}}
{{ include "hello.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
```

`chart/hello/templates/configmap.yaml` и `service.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-config
  labels:
    {{- include "hello.labels" . | nindent 4 }}
data:
  GREETING: {{ .Values.greeting | quote }}
  STARTUP_DELAY_MS: {{ .Values.startupDelayMs | quote }}
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Release.Name }}
  labels:
    {{- include "hello.labels" . | nindent 4 }}
spec:
  selector:
    {{- include "hello.selectorLabels" . | nindent 4 }}
  ports:
    - name: http
      port: 80
      targetPort: http
```

`chart/hello/templates/deployment.yaml` — это Deployment из Урока 21.5 с заполнителями:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}
  labels:
    {{- include "hello.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "hello.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        {{- include "hello.labels" . | nindent 8 }}
      annotations:
        # A new checksum after every config change starts a rolling update.
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
        - name: hello
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 8080
          envFrom:
            - configMapRef:
                name: {{ .Release.Name }}-config
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
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          securityContext:
            runAsNonRoot: true
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop: ["ALL"]
      volumes:
        - name: admin-secret
          secret:
            secretName: {{ .Values.adminSecretName }}
```

**Что происходит:**

1. Всё между `{{` и `}}` — код шаблона; остальное копируется как есть. `{{-` и `-}}` ещё и убирают пробелы (и переводы строк) рядом с собой.
2. `.Values` — объединённые значения (умолчания из `values.yaml`, затем файлы `-f`, затем `--set`). `.Release.Name` — имя, которое вы даёте при установке, поэтому два релиза одного чарта не конфликтуют. `.Chart` — это `Chart.yaml`.
3. `define "hello.labels"` даёт имя куску шаблона; `include "hello.labels" .` вставляет его, а `| nindent 4` ставит его с новой строки с отступом в 4 пробела. `.` передаёт куску текущие данные. Конвейер `|` отправляет результат левой части в функцию справа, как конвейер в оболочке.
4. `| quote` берёт значение в двойные кавычки. Значения ConfigMap должны быть строками, а без кавычек значение вроде `3000` (число) или `yes` могло бы прочитаться как другой тип.
5. `{{ .Values.image.tag | default .Chart.AppVersion }}` берёт тег из значений или `appVersion`, если тег пуст. Поэтому по умолчанию версия образа следует за `appVersion` чарта.
6. `toYaml .Values.resources | nindent 12` записывает целый блок значений как YAML, и пользователи могут менять `resources`, а шаблону не нужно знать каждое поле.
7. `checksum/config` — приём из Урока 21.5: Helm отрисовывает шаблон ConfigMap, вычисляет его контрольную сумму SHA-256 (Secure Hash Algorithm, безопасный алгоритм хеширования, 256 бит) и записывает её в шаблон пода. Новое приветствие даёт новую сумму, шаблон пода меняется, и Deployment автоматически обновляет поды.
8. Селектор использует только `hello.selectorLabels` (имя и экземпляр), которые никогда не меняются. Метки подов содержат ещё и версию, которая меняется с каждым релизом.

## 4. Проверьте чарт до установки

`helm lint` находит ошибки в структуре чарта; `helm template` отрисовывает манифесты на вашем Mac без всякого кластера:

```bash
helm lint chart/hello
helm template hello chart/hello
```

```text
==> Linting chart/hello
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

Отрисованный ConfigMap и начало Deployment (сокращено):

```text
---
# Source: hello/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-config
  labels:
    app.kubernetes.io/name: hello
    app.kubernetes.io/instance: hello
    app.kubernetes.io/version: "0.3.0"
data:
  GREETING: "salam"
  STARTUP_DELAY_MS: "3000"
...
# Source: hello/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
...
      annotations:
        # A new checksum after every config change starts a rolling update.
        checksum/config: 95cd63356bcb64007f0daf66cbd4adb92542467f0b8e8e87a69a4725c5f5cfb8
    spec:
      containers:
        - name: hello
          image: "course-p21-hello:0.3.0"
```

Читайте отрисованный вывод при каждом изменении шаблона: большинство ошибок чарта видны здесь — неверный отступ или отсутствующее значение.

> 🧪 **Тестирование:** вывод `helm template` — обычный YAML, поэтому чарт можно тестировать без кластера: передать его в `kubectl apply --dry-run=server -f -` на тестовом кластере, проверить инструментом политик или сравнить с сохранённой копией (снапшот-тест, Урок 09.5). Упражнение 2 показывает собственную возможность тестирования Helm.

## 5. Установка, обновление, откат

Создайте кластер и Secret, загрузите образ (как в Уроке 21.5), затем установите чарт как релиз `hello`:

```bash
docker build -t course-p21-hello:0.3.0 .
kind create cluster --name course-21-06
kind load docker-image course-p21-hello:0.3.0 --name course-21-06
kubectl create secret generic hello-admin --from-file=admin-token=admin-token.txt
helm install hello chart/hello --wait
```

```text
NAME: hello
LAST DEPLOYED: Wed Nov 18 13:57:45 2026
NAMESPACE: default
STATUS: deployed
REVISION: 1
DESCRIPTION: Install complete
TEST SUITE: None
```

`--wait` заставляет Helm ждать, пока новые объекты не станут готовыми, поэтому статус `deployed` значит, что поды действительно работают. У подов новые метки:

```bash
kubectl get pods -l app.kubernetes.io/instance=hello
```

```text
NAME                     READY   STATUS    RESTARTS   AGE
hello-76cf754449-2zp79   1/1     Running   0          6s
hello-76cf754449-k6gk2   1/1     Running   0          6s
hello-76cf754449-lw55n   1/1     Running   0          6s
```

Теперь поменяйте две настройки только для этого релиза, не трогая файлы:

```bash
helm upgrade hello chart/hello --set greeting=privet --set replicaCount=2 --wait
```

```text
Release "hello" has been upgraded. Happy Helming!
NAME: hello
LAST DEPLOYED: Wed Nov 18 13:58:00 2026
NAMESPACE: default
STATUS: deployed
REVISION: 2
DESCRIPTION: Upgrade complete
TEST SUITE: None
```

Приветствие изменило ConfigMap, контрольная сумма изменила шаблон пода, и Deployment обновил поды без `rollout restart`: запрос теперь печатает `privet from hello-59f549fc88-df4jd (version 0.3.0)`. Helm помнит каждую ревизию:

```bash
helm history hello
```

```text
REVISION	UPDATED                 	STATUS    	CHART      	APP VERSION	DESCRIPTION     
1       	Wed Nov 18 13:57:45 2026	superseded	hello-0.1.0	0.3.0      	Install complete
2       	Wed Nov 18 13:58:00 2026	deployed  	hello-0.1.0	0.3.0      	Upgrade complete
```

Откатитесь к ревизии 1:

```bash
helm rollback hello 1 --wait
helm history hello
```

```text
Rollback was a success! Happy Helming!
REVISION	UPDATED                 	STATUS    	CHART      	APP VERSION	DESCRIPTION     
1       	Wed Nov 18 13:57:45 2026	superseded	hello-0.1.0	0.3.0      	Install complete
2       	Wed Nov 18 13:58:00 2026	superseded	hello-0.1.0	0.3.0      	Upgrade complete
3       	Wed Nov 18 13:58:14 2026	deployed  	hello-0.1.0	0.3.0      	Rollback to 1   
```

Откат — это **новая** ревизия (3) со значениями ревизии 1; история никогда не переписывается. В отличие от `kubectl rollout undo` (Урок 21.4), он восстанавливает весь релиз: ConfigMap, Service и Deployment вместе. `helm get values hello --all` печатает все значения, с которыми работает релиз, включая умолчания.

> 📏 **Соглашение:** в скриптах и в CI (Continuous Integration, непрерывная интеграция) используйте `helm upgrade --install <релиз> <чарт> --wait`: он устанавливает релиз, если его нет, и обновляет, если есть, поэтому одна и та же команда работает и в первый раз, и во все последующие. Для окружения храните переопределения в файле в Git (`-f values-dev.yaml`), а не в длинных списках `--set`.

Когда релиз больше не нужен, `helm uninstall hello` удаляет все созданные им объекты (Secret, сделанный вами вручную, остаётся).

## 6. Kustomize: база и оверлей

Kustomize не требует нового инструмента: `kubectl kustomize` отрисовывает, а `kubectl apply -k` применяет. Создайте такую структуру внутри `code/p21-hello`, скопировав `deployment.yaml` и `service.yaml` из `k8s/` в `kustomize/base/` (но не `configmap.yaml`: его заменит генератор):

```text
kustomize/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml     ← скопирован из k8s/
│   └── service.yaml        ← скопирован из k8s/
└── overlays/
    └── dev/
        └── kustomization.yaml
```

`kustomize/base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
configMapGenerator:
  - name: hello-config
    literals:
      - GREETING=salam
      - STARTUP_DELAY_MS=3000
```

`kustomize/overlays/dev/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: hello-dev
resources:
  - ../../base
replicas:
  - name: hello
    count: 1
configMapGenerator:
  - name: hello-config
    behavior: merge
    literals:
      - GREETING=privet
images:
  - name: course-p21-hello
    newTag: "0.3.0"
```

**Что происходит:**

1. База перечисляет свои файлы в `resources` и **генерирует** ConfigMap `hello-config` из буквальных значений.
2. Оверлей берёт всю базу (`../../base`) и меняет её: каждый объект попадает в пространство имён `hello-dev`; Deployment `hello` получает 1 реплику; сгенерированный ConfigMap **объединяется** с новым `GREETING`; а каждый образ с именем `course-p21-hello` получает тег `0.3.0` (так каждое окружение выбирает свою версию).
3. Файлы базы остаются полными, корректными манифестами. Их по-прежнему можно применить через `kubectl apply -f`; Kustomize лишь меняет поля.

Отрисуйте оверлей, чтобы увидеть результат (сокращено):

```bash
kubectl kustomize kustomize/overlays/dev
```

```text
apiVersion: v1
data:
  GREETING: privet
  STARTUP_DELAY_MS: "3000"
kind: ConfigMap
metadata:
  name: hello-config-bg8b5cd5bg
  namespace: hello-dev
---
...
spec:
  replicas: 1
...
        envFrom:
        - configMapRef:
            name: hello-config-bg8b5cd5bg
        image: course-p21-hello:0.3.0
```

Генератор добавил к имени ConfigMap **хеш содержимого** (`-bg8b5cd5bg`) и поменял ссылку в Deployment, чтобы она совпадала. Примените это к кластеру (пространству имён нужна собственная копия Secret):

```bash
kubectl create namespace hello-dev
kubectl create secret generic hello-admin -n hello-dev --from-file=admin-token=admin-token.txt
kubectl apply -k kustomize/overlays/dev
kubectl rollout status deployment/hello -n hello-dev
```

```text
namespace/hello-dev created
secret/hello-admin created
configmap/hello-config-bg8b5cd5bg created
service/hello created
deployment.apps/hello created
Waiting for deployment "hello" rollout to finish: 0 of 1 updated replicas are available...
deployment "hello" successfully rolled out
```

Запрос в этом пространстве имён печатает `privet from hello-7d7994f67b-h84fv (version 0.3.0)`. Теперь поменяйте в оверлее `GREETING=privet` на `GREETING=zdravstvuy` и примените снова:

```bash
kubectl apply -k kustomize/overlays/dev
kubectl get pods,configmaps -n hello-dev
```

```text
configmap/hello-config-f42cb6hh84 created
service/hello unchanged
deployment.apps/hello configured
NAME                         READY   STATUS        RESTARTS   AGE
pod/hello-7d7994f67b-h84fv   1/1     Terminating   0          14s
pod/hello-98ff5f8d-hnfmn     1/1     Running       0          5s

NAME                                DATA   AGE
configmap/hello-config-bg8b5cd5bg   2      14s
configmap/hello-config-f42cb6hh84   2      5s
configmap/kube-root-ca.crt          1      14s
```

Новое содержимое, новое имя, новая ссылка в шаблоне пода: Deployment сам обновил под — тот же эффект, что у контрольной суммы в Helm. Старый ConfigMap остаётся, неиспользуемый, и это делает возврат к предыдущей версии безопасным; время от времени убирайте такие остатки. Удалите окружение dev командой `kubectl delete namespace hello-dev`.

## 7. Helm или Kustomize?

| | Helm | Kustomize |
|---|---|---|
| Идея | Шаблоны + значения | База + патчи |
| Дополнительный инструмент | Да (`helm`) | Нет (встроен в kubectl) |
| История релизов и откат | Да (`helm history`, `helm rollback`) | Нет (используйте Git и `kubectl apply`) |
| Как делиться с другими | Чарты в репозиториях (в том числе как OCI-образы в реестрах; OCI — Open Container Initiative) | Папки в Git |
| Читаемость | Шаблоны могут стать трудночитаемыми | Файлы остаются обычным YAML |
| Типичное применение | Установка стороннего ПО; один чарт для многих команд | Ваши собственные сервисы с несколькими окружениями |

Многие команды используют оба: стороннее ПО (например, Prometheus, с которым вы встретитесь в Уроке 21.9) они ставят официальным чартом Helm, а свои сервисы ведут через Kustomize или небольшой собственный чарт. OCI (открытая инициатива по контейнерам) — стандарт образов из Урока 21.4; Helm может хранить чарты в тех же реестрах, что и образы.

## 8. Уборка

```bash
helm uninstall hello
kind delete cluster --name course-21-06
docker image rm course-p21-hello:0.3.0
```

Папки `chart/` и `kustomize/` сохраните: в Уроке 21.7 чарт развёртывается из конвейера.

## Частые ошибки

**Изменить метки селектора у установленного релиза.** Селектор Deployment нельзя менять после создания. Если добавить метку в `hello.selectorLabels` и обновиться, API-сервер откажет:

```text
Error: UPGRADE FAILED: server-side apply failed for object default/hello apps/v1, Kind=Deployment: Deployment.apps "hello" is invalid: spec.selector: Invalid value: {"matchLabels":{"app.kubernetes.io/component":"web","app.kubernetes.io/instance":"hello","app.kubernetes.io/name":"hello"}}: field is immutable
```

Неудачное обновление остаётся в `helm history` как ревизия со статусом `failed`, а работающая версия не затрагивается. Исправление: никогда не меняйте метки селектора; новые метки добавляйте только в `hello.labels`. Если без этого никак, удалите Deployment и установите заново (с коротким простоем).

**Любой под с метками селектора получает трафик.** Service выбирает *поды*, а не «поды моего Deployment». В ранней версии Упражнения 2 тестовый под получил полный набор `hello.labels`, и Service стал отправлять запросы и самому тестовому поду, который не слушает ни одного порта. EndpointSlice показали адрес тестового пода как четвёртый эндпоинт, и часть запросов падала:

```text
NAME          ADDRESSTYPE   PORTS     ENDPOINTS                             AGE
hello-jvldb   IPv4          8080      10.244.0.16,10.244.0.14,10.244.0.15   82s
hello-qv956   IPv4          <unset>   10.244.0.17                           1s
```

```text
wget: download timed out
```

Исправление: давайте вспомогательным подам (тестам, заданиям, отладочным подам) метки, которые не совпадают ни с одним селектором Service.

**Секреты в `values.yaml`.** `values.yaml` лежит в Git, а `helm get values` показывает значения любому, кто может читать релиз. Передавайте только *имя* Secret, как делает этот чарт, а сам Secret создавайте отдельно.

## Попробуйте сами

1. Отрисуйте чарт с другим именем релиза: `helm template shop chart/hello`. Имена каких объектов изменились?
2. Создайте `values-dev.yaml` с `greeting: privet` и `replicaCount: 1` и отрисуйте с `-f values-dev.yaml`. Сравните с Упражнением 1.
3. Установите чарт дважды в один кластер, как релизы `hello` и `shop` (оба могут использовать Secret `hello-admin`). Посмотрите на имена объектов и метки селекторов: конфликтуют ли два релиза?

## Упражнения

### Упражнение 1 — Отрисовка с переопределениями (легко)

Без кластера отрисуйте чарт с `greeting=privet` и `replicaCount=1` и покажите только строки с `GREETING`, `replicas:` и контрольной суммой. Сравните контрольную сумму со значением по умолчанию из раздела 4.

<details>
<summary>Подсказка</summary>

`helm template hello chart/hello --set ... --set ... | grep -E 'GREETING|replicas:|checksum'`.

</details>

<details>
<summary>Ответ</summary>

```bash
helm template hello chart/hello --set greeting=privet --set replicaCount=1 | grep -E 'GREETING|replicas:|checksum'
```

```text
  GREETING: "privet"
  replicas: 1
        # A new checksum after every config change starts a rolling update.
        checksum/config: 164c134ea9938e8ee94c0f5dd2ce33cd8e033e7f24abb3f0cd4067342f4f231e
```

Контрольная сумма отличается от значения по умолчанию (`95cd6335...`), потому что изменилось содержимое ConfigMap; `replicaCount` в ConfigMap не входит, поэтому сам по себе сумму не изменил бы. Тот же вывод даёт и файл: `-f values-dev.yaml` с этими двумя значениями.

</details>

### Упражнение 2 — Тест Helm (средне)

Добавьте **тест Helm**: под в `templates/tests/test-connection.yaml` с аннотацией `"helm.sh/hook": test`, который запрашивает `http://<имя релиза>/` и проходит, только если ответ начинается с настроенного приветствия. Запустите его через `helm test hello --logs`, затем обновитесь с `--set greeting=privet` и запустите снова. Наконец, специально уроните его через `kubectl scale deployment hello --replicas=0`.

<details>
<summary>Подсказка</summary>

У тестового пода `restartPolicy: Never` и один контейнер, команда которого завершается с `0` при успехе. Используйте `busybox:1.37` и `sh -c "wget -qO- -T 5 http://... | grep '... from'"`. Прежде чем выбирать метки пода, прочитайте «Частые ошибки». Аннотация `"helm.sh/hook-delete-policy": before-hook-creation` удаляет старый тестовый под перед следующим запуском.

</details>

<details>
<summary>Ответ</summary>

`chart/hello/templates/tests/test-connection.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ .Release.Name }}-test
  labels:
    # Not the selector labels: the Service must not send traffic to this pod.
    app.kubernetes.io/instance: {{ .Release.Name }}
    app.kubernetes.io/component: test
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  restartPolicy: Never
  containers:
    - name: wget
      image: busybox:1.37
      command:
        - sh
        - -c
        - wget -qO- -T 5 http://{{ .Release.Name }} | grep '{{ .Values.greeting }} from'
```

```bash
helm upgrade hello chart/hello --wait
helm test hello --logs
```

Конец вывода:

```text
TEST SUITE:     hello-test
Last Started:   Wed Nov 18 14:04:32 2026
Last Completed: Wed Nov 18 14:04:34 2026
Phase:          Succeeded

POD LOGS: hello-test (wget)
salam from hello-76cf754449-q27jh (version 0.3.0)
```

После `helm upgrade hello chart/hello --set greeting=privet --wait` тест проверяет `privet from` и снова проходит (`Phase: Succeeded`, `privet from hello-59f549fc88-mgbk8 (version 0.3.0)`). С 0 реплик он падает, и `helm test` завершается с ненулевым кодом:

```text
Phase:          Failed

POD LOGS: hello-test (wget)
wget: can't connect to remote host (10.96.1.205): Connection refused

Error: resource Pod/default/hello-test not ready. status: Failed, message: pod hello-test failed
```

Почему это работает: Helm создаёт поды с хуком `test` только когда вы запускаете `helm test`, ждёт их и сообщает результат. Поскольку ожидаемый текст берётся из `.Values.greeting`, тест всегда проверяет конфигурацию текущей ревизии. Запускайте его в CI сразу после `helm upgrade` (Урок 21.7). Учтите, что `helm uninstall` не удаляет поды хуков; оставшийся удалите через `kubectl delete pod hello-test`.

</details>

### Упражнение 3 — Проверка значений (сложно)

Пользователь запускает `helm upgrade hello chart/hello --set replicaCount=0` или `--set replicaCount=two`. Сделайте так, чтобы Helm отказывал в обоих случаях *до того*, как что-либо дойдёт до кластера, добавив `chart/hello/values.schema.json`: `replicaCount` должен быть целым числом не меньше 1, `greeting` — непустой строкой, `startupDelayMs` — целым числом не меньше 0.

<details>
<summary>Подсказка</summary>

Helm проверяет объединённые значения по `values.schema.json` (JSON Schema) в `lint`, `template`, `install` и `upgrade`. Используйте `"type": "integer"`, `"minimum"`, `"minLength"` и `"required"`.

</details>

<details>
<summary>Ответ</summary>

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["replicaCount", "greeting"],
  "properties": {
    "replicaCount": { "type": "integer", "minimum": 1 },
    "greeting": { "type": "string", "minLength": 1 },
    "startupDelayMs": { "type": "integer", "minimum": 0 }
  }
}
```

```bash
helm upgrade hello chart/hello --set replicaCount=0
helm upgrade hello chart/hello --set replicaCount=two
```

```text
Error: UPGRADE FAILED: values don't meet the specifications of the schema(s) in the following chart(s):
hello:
- at '/replicaCount': minimum: got 0, want 1

Error: UPGRADE FAILED: values don't meet the specifications of the schema(s) in the following chart(s):
hello:
- at '/replicaCount': got string, want integer
```

`helm lint chart/hello --set replicaCount=0` сообщает о той же проблеме как `[ERROR]`, поэтому CI может поймать её до любого развёртывания. С корректными значениями `helm lint chart/hello` проходит, как раньше.

Почему это работает: схема превращает `values.yaml` в проверяемый контракт, как типы для параметров функции. Без неё `replicaCount: two` дошёл бы до API-сервера и упал бы там с менее понятным сообщением, а `0` — хуже того — был бы принят и тихо остановил бы сервис.

</details>

## Вопросы для самопроверки

1. Чем отличаются чарт, релиз и ревизия?

<details>
<summary>Ответ</summary>

Чарт — пакет (шаблоны и значения по умолчанию). Релиз — одна установленная копия чарта в кластере с именем. Ревизия — одна пронумерованная версия релиза; каждая установка, обновление и откат добавляют новую.

</details>

2. В каком порядке объединяются значения и что побеждает?

<details>
<summary>Ответ</summary>

Сначала `values.yaml` чарта, затем файлы, переданные через `-f` (по порядку), затем `--set`. Более поздние источники переопределяют более ранние, поэтому побеждает `--set`.

</details>

3. Зачем чарт кладёт контрольную сумму ConfigMap в шаблон пода?

<details>
<summary>Ответ</summary>

Потому что поды читают ConfigMap только при запуске. Когда конфигурация меняется, меняется сумма, меняется шаблон пода, и Deployment автоматически обновляет поды.

</details>

4. Что создаёт `helm rollback hello 1` и чем это отличается от `kubectl rollout undo`?

<details>
<summary>Ответ</summary>

Он создаёт новую ревизию с конфигурацией ревизии 1. Он восстанавливает все объекты релиза (ConfigMap, Service, Deployment), а `kubectl rollout undo` меняет только шаблон пода одного Deployment.

</details>

5. Что такое база и что такое оверлей в Kustomize?

<details>
<summary>Ответ</summary>

База — общий полный набор манифестов. Оверлей — папка для одного окружения, которая ссылается на базу и перечисляет изменения (пространство имён, реплики, образы, объединённые значения ConfigMap, патчи).

</details>

6. Почему у ConfigMap, сделанного генератором Kustomize, имя вида `hello-config-bg8b5cd5bg`?

<details>
<summary>Ответ</summary>

Суффикс — хеш содержимого. Изменение содержимого даёт новое имя, Kustomize обновляет ссылки в шаблоне пода, и Deployment обновляет поды; старый ConfigMap остаётся для безопасного возврата.

</details>

7. Назовите одну причину выбрать Helm и одну — Kustomize.

<details>
<summary>Ответ</summary>

Helm: история релизов и откат или установка стороннего ПО из репозитория чартов. Kustomize: нет лишнего инструмента и языка шаблонов; файлы остаются обычным YAML, что подходит для ваших собственных сервисов с несколькими окружениями.

</details>

## Итоги

- Копии манифестов для каждого окружения расходятся. Helm (шаблоны + значения) и Kustomize (база + оверлеи) получают обычные манифесты из одного источника.
- В чарте Helm есть `Chart.yaml` (`version` чарта и `appVersion` программы), `values.yaml` и шаблоны; `.Values`, `.Release` и `.Chart` заполняют заполнители; `include`, `nindent`, `quote`, `default` и `toYaml` делают большую часть работы.
- Проверяйте через `helm lint` и `helm template`; развёртывайте через `helm upgrade --install --wait`; смотрите через `helm history` и `helm get values`; возвращайтесь через `helm rollback` (новая ревизия).
- Аннотация с контрольной суммой (Helm) или имя с хешем от генератора (Kustomize) заставляют изменения конфигурации автоматически обновлять поды.
- Никогда не меняйте метки селектора; держите метки вспомогательных подов подальше от селекторов Service; не храните секретные значения в `values.yaml`.
- `values.schema.json` заставляет Helm отклонять плохие значения до того, как они дойдут до кластера.

## Источники и что почитать

- [Helm — Quickstart Guide](https://helm.sh/docs/intro/quickstart/) — установка Helm и первые команды.
- [Helm — Charts](https://helm.sh/docs/topics/charts/) — структура чарта, `Chart.yaml` и `values.schema.json`.
- [Helm — Chart Template Guide](https://helm.sh/docs/chart_template_guide/getting_started/) — язык шаблонов шаг за шагом.
- [Helm — Chart Tips and Tricks](https://helm.sh/docs/howto/charts_tips_and_tricks/) — в том числе аннотация с контрольной суммой для автоматических обновлений.
- [Helm — Chart Tests](https://helm.sh/docs/topics/chart_tests/) — как писать и запускать `helm test`.
- [Helm — Labels and Annotations](https://helm.sh/docs/chart_best_practices/labels/) — какие метки должен ставить чарт.
- [Документация Kubernetes — Declarative Management of Kubernetes Objects Using Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) — базы, оверлеи и генераторы.
- [Справочник Kustomize — kustomization](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/) — каждое поле `kustomization.yaml`.

---

[🏠 Начало](../START-HERE.md) · [← 21.5 ConfigMap, Secret и пробы работоспособности](05-configmaps-secrets-and-probes.md) · [📋 Этап 21](README.md) · [21.7 CI/CD: от коммита до продакшена →](07-ci-cd-from-commit-to-production.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
