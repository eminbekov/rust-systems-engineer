<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 21.12 Инциденты, дежурства и постмортемы](12-incidents-on-call-and-postmortems.md) · [📋 Этап 21](README.md) · [21.14 Контрольная точка →](14-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/21-cloud-native-infrastructure/13-project-run-your-service-on-kubernetes.md) <!-- nav -->

# Урок 21.13 — Проект: ваш сервис в Kubernetes

> **Этап 21 — Облачная (cloud-native) инфраструктура** · Урок 13 из 14 · ⏱ около 12 часов (6 занятий)

## Что вы узнаете

- Как собрать всё, что было на этом этапе, вокруг одного настоящего сервиса — вашего сервиса коротких ссылок (URL shortener; URL — Uniform Resource Locator, веб-адрес) из Этапа 16.
- Как разделить конфигурацию на ConfigMap и Secret, хранить данные в PersistentVolumeClaim и правильно проверять сервис пробами.
- Как упаковать сервис в Helm-чарт, выпускать новые версии и откатываться.
- Как собирать образ и проверять чарт и правила в конвейере CI (Continuous Integration, непрерывная интеграция).
- Как дать сервису метрики Prometheus, один SLO (Service Level Objective, цель уровня обслуживания) с оповещением по скорости расходования, ранбук и README, по которому сможет работать другой инженер.

## Перед началом

- Вы прошли [Урок 21.12 — Инциденты, дежурства и постмортемы](12-incidents-on-call-and-postmortems.md) и все уроки этого этапа до него.
- Вы сделали сервис коротких ссылок в [Уроке 16.15 — Проект: сервис коротких ссылок](../16-backend-services/15-project-url-shortener-service.md): `code/p16-shortener`, с PostgreSQL, необязательным Redis, `/healthz`, `/readyz`, трассировками, Dockerfile и интеграционными тестами. Если вы его пропустили, возьмите `code/p21-orders` из Урока 21.9 и опустите части про базу данных; всё остальное подходит без изменений.
- Docker Desktop запущен, а `kind`, `kubectl`, `helm`, `jq` и `actionlint` работают (Уроки 21.3, 21.6 и 09.12).

## Новые слова

| Слово | Значение |
|---|---|
| **PVC** (PersistentVolumeClaim, заявка на постоянный том) | Запрос хранилища в Kubernetes: «дай мне 1 ГиБ, который переживёт перезапуски подов». Кластер находит или создаёт для него **PersistentVolume**. kind создаёт его на диске узла. |
| **StorageClass** (класс хранилища) | Описывает, *как* создаются тома. В kind есть класс `standard`, он используется по умолчанию. |
| **запросы / лимиты ресурсов** (resource requests / limits) | Сколько CPU (Central Processing Unit, центральный процессор) и памяти контейнеру гарантировано (запросы, их использует планировщик) и сколько он может использовать максимум (лимиты). |
| **релиз** (release, Helm) | Одна установка чарта в кластер с именем и историей ревизий (Урок 21.6). |
| **источник истины** (source of truth) | Единственное место, где хранится желаемое состояние (здесь — репозиторий Git). Кластер всегда приводят к нему, а не наоборот. |
| **GHCR** (GitHub Container Registry, реестр контейнеров GitHub) | Реестр образов GitHub по адресу `ghcr.io`. Workflow может отправлять туда образы со своим встроенным токеном. |

## Проект

### Что вы создадите

```text
  git push ──► GitHub Actions: fmt · clippy · test · helm lint · promtool test · build image
                                                                   │ (main only: push to ghcr.io)
  your Mac: kind cluster "course-21-13"                            ▼
 ┌───────────────────────────── namespace shortener ──────────────────────────────┐
 │  Service shortener ──► Deployment shortener (2 pods, Helm release "shortener")  │
 │                          envFrom: ConfigMap shortener-config + Secret          │
 │                          probes: /readyz (readiness), /healthz (liveness)       │
 │                          /metrics ◄── scraped by Prometheus                     │
 │                              │                                                 │
 │                              ▼                                                 │
 │  Service postgres ──► Deployment postgres (1 pod) ──► PVC postgres-data (1Gi)   │
 └────────────────────────────────────────────────────────────────────────────────┘
 ┌──────────── namespace monitoring ─────────────┐
 │  Prometheus (Helm chart) + SLO rules + alert   │
 └────────────────────────────────────────────────┘
```

Вы берёте сервис коротких ссылок из Урока 16.15 и запускаете его в Kubernetes так, как его запускала бы команда в продакшене: конфигурация и секреты вне образа, данные на томе, две реплики за Service, правильные пробы, Helm-чарт с версионированными релизами, конвейер CI, метрики, SLO с протестированным оповещением, ранбук и README.

### Контракт

**Папки.** Скопируйте проект, чтобы версия из Этапа 16 осталась как есть: `cp -R code/p16-shortener code/p21-shortener`. Всё в этом проекте лежит в `code/p21-shortener`, кроме workflow, который GitHub читает только из корня репозитория:

```text
~/projects/rust/
├── .github/workflows/p21-shortener.yml   ← CI для этого проекта (M5)
└── code/p21-shortener/
    ├── Cargo.toml, src/, migrations/, tests/, Dockerfile   ← из Урока 16.15
    ├── README.md                    ← как развернуть, выпустить, откатить (M6)
    ├── k8s/
    │   ├── namespace.yaml
    │   └── postgres.yaml            ← PVC, Deployment, Service (M2)
    ├── chart/shortener/             ← Helm-чарт сервиса (M4)
    └── deploy/
        ├── prometheus-values.yaml   ← значения чарта с правилами SLO (M3)
        ├── rules/slo-rules.yml
        ├── rules/slo-rules-test.yml
        └── runbooks/shortener-slo-fast-burn.md
```

**Имена.** Кластер `course-21-13`, пространство имён `shortener`, образ `course-p21-shortener:<версия из Cargo.toml>`, Deployment и Service `shortener` (порт Service 80 → порт контейнера 8080), Helm-релиз `shortener`.

**Конфигурация.** Несекретные настройки — в ConfigMap `shortener-config`: `APP_HOST=0.0.0.0` (до значения по умолчанию `127.0.0.1` Service не достучится, Урок 21.4), `APP_PORT=8080`, `PUBLIC_BASE_URL`, `RATE_LIMIT_PER_MINUTE`. Секреты — только в Secret `shortener-secrets`: `DATABASE_URL` и `ADMIN_TOKEN`. Ни одного секретного значения нет в Git, в значениях чарта, в логах и в метках метрик.

**Поды.** 2 реплики; проба готовности `GET /readyz`, проба живости `GET /healthz`; запросы и лимиты CPU и памяти; контейнер работает не от root (пользователь distroless `nonroot` из Урока 16.13). Удаление пода или перезапуск узла не теряет данные.

**Метрики.** `GET /metrics` в формате Prometheus с `http_requests_total{method,path,status}` и гистограммой `http_request_duration_seconds{method,path}`, как в Уроке 21.9 (в метках — шаблоны маршрутов, никогда не сырые пути или коды). `/metrics` не ограничивается лимитом запросов. В шаблоне пода — три аннотации `prometheus.io/*`.

**SLO.** «99,5% переходов по ссылкам (`GET /{code}`) не падают с 5xx, за 30 дней». Правила записи для доли ошибок за 5 минут и за 1 час и оповещение о быстром расходовании (скорость больше 14,4 в обоих окнах, `for: 2m`, `severity: page`), с тестом `promtool`, что оно срабатывает при 10% ошибок и молчит при 1%.

**CI.** Один workflow; задания: проверки Rust (`cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --all-targets`), чарт и правила (`helm lint`, `promtool test rules`), сборка образа (отправка в GHCR только на `main`). `actionlint` ничего не сообщает.

**Качество.** Все тесты из Этапа 16 по-прежнему проходят, `cargo clippy` чист, `cargo fmt` применён. Один коммит Git на веху (`M1: …`).

### Пошаговый план

Держите один терминал для долгих команд (port-forward) и один для всего остального. Создайте кластер один раз: `kind create cluster --name course-21-13`.

#### M1 — Образ, подходящий кластеру

**Цель:** образ сервиса запускается как контейнер не от root, слушает все интерфейсы, когда ему это сказано, и отдаёт `/metrics`.

**Как действовать:** добавьте middleware метрик и маршрут `/metrics` из Урока 21.9 (`metrics` + `metrics-exporter-prometheus` с корзинами для гистограммы длительности). Поставьте `/metrics` рядом с `/healthz` и `/readyz` в список маршрутов, которые пропускает ограничитель запросов. Задайте в `Cargo.toml` версию `0.1.0`, если она другая, соберите `course-p21-shortener:0.1.0` своим Dockerfile из Урока 16.13 и один раз запустите его через Docker Compose из Урока 16.15 (поменяйте там имя образа), чтобы проверить.

<details>
<summary>Подсказка</summary>

Тест вроде Упражнения 2 из Урока 21.9 доказывает, что переход считается с `path="/{code}"`. Запустите его до сборки образа: `MatchedPath` маршрута перехода — это шаблон, который вы написали в маршрутизаторе.

</details>

**Готово, когда:**

```bash
cargo test
docker build -t course-p21-shortener:0.1.0 .
docker image ls --filter reference=course-p21-shortener --format '{{.Repository}}:{{.Tag}}'
curl -s localhost:8080/metrics | grep -c '^http_requests_total'
```

Ожидается: все тесты проходят; список образов показывает `course-p21-shortener:0.1.0`; последняя команда (при запущенном стеке Compose и после одного перехода) печатает число больше 0. Коммит: `M1: metrics endpoint and image 0.1.0`.

#### M2 — Простые манифесты: база данных, конфигурация, секрет, пробы

**Цель:** сервис и PostgreSQL работают в кластере из простых файлов YAML (YAML Ain't Markup Language), а данные переживают удаление подов.

**Как действовать:** сначала `k8s/namespace.yaml` и `k8s/postgres.yaml`: PVC `postgres-data` (1Gi, `ReadWriteOnce`), Deployment `postgres` с официальным образом `postgres`, его пароль из Secret `postgres`, смонтированный том, проба готовности, которая запускает `pg_isready`, и Service `postgres` на порту 5432. Оба Secret создайте командой `kubectl create secret generic` (никогда из файла в Git). Затем вручную напишите Deployment, Service и ConfigMap сервиса — в M4 вы превратите их в чарт, но если один раз сделать это руками, станет видно, что чарт должен получить на выходе. Загрузите образ через `kind load docker-image`. Миграции выполняются при старте сервиса (Урок 16.15), поэтому таблицу создаёт первый готовый под.

<details>
<summary>Подсказка: Deployment для PostgreSQL</summary>

Для единственного пода базы данных используйте `strategy: type: Recreate`, чтобы старый под освободил том до запуска нового. С образом `postgres` задайте `PGDATA` как подпапку точки монтирования, например смонтируйте том в `/var/lib/postgresql/data` и задайте `PGDATA=/var/lib/postgresql/data/pgdata`. Пароль берётся через `valueFrom.secretKeyRef` (Урок 21.5). Тогда `DATABASE_URL` сервиса — `postgres://postgres:<password>@postgres.shortener.svc:5432/postgres`.

</details>

<details>
<summary>Подсказка: ресурсы</summary>

Начните с малого и измерьте потом: например, запросы `cpu: 50m, memory: 64Mi` и лимиты `cpu: 500m, memory: 256Mi` для сервиса. `kubectl top` нужен сервер метрик, которого kind не устанавливает; пока следите за `OOMKilled` в `kubectl describe pod`.

</details>

**Готово, когда:**

```bash
kubectl -n shortener get pvc
kubectl -n shortener get pods
kubectl -n shortener port-forward service/shortener 8080:80
```

Ожидаемый вывод первых двух команд (имена и возраст будут другими):

```text
NAME            STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
postgres-data   Bound    pvc-63cc8967-814b-4708-9cec-417bdbaec44a   1Gi        RWO            standard       <unset>                 24s
```

```text
NAME                         READY   STATUS    RESTARTS   AGE
postgres-86885ff55b-z7cqh    1/1     Running   0          2m
shortener-5d5598c9c7-b9vzq   1/1     Running   0          1m
shortener-5d5598c9c7-flttz   1/1     Running   0          1m
```

Пока работает port-forward, создайте ссылку, как в Уроке 16.15 (`POST /api/links` с вашим токеном администратора и `{"url":"https://www.rust-lang.org/learn","code":"learn"}`), затем удалите **все** поды командой `kubectl -n shortener delete pod --all`, дождитесь, пока они снова станут готовы, перезапустите port-forward и выполните:

```bash
curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' localhost:8080/learn
```

```text
302 https://www.rust-lang.org/learn
```

Ссылка пережила удаление, потому что живёт на томе. Коммит: `M2: plain manifests with PostgreSQL on a volume`.

#### M3 — Prometheus, SLO и его тест

**Цель:** Prometheus собирает метрики с обоих подов сервиса, правила SLO загружены и протестированы.

**Как действовать:** установите чарт Prometheus, как в Уроке 21.9, с `deploy/prometheus-values.yaml`, где группы SLO лежат в `serverFiles.alerting_rules.yml` (Урок 21.11). Адаптируйте правила: SLI (Service Level Indicator, показатель уровня обслуживания) касается `path="/{code}"`; назовите ряды `slo:shortener_redirect_errors:ratio_rate5m` и `..._ratio_rate1h`, а оповещение — `ShortenerRedirectsFastBurn`. Держите копию групп в `deploy/rules/slo-rules.yml` и два теста в `slo-rules-test.yml`: один срабатывает при 10% ошибок, другой молчит при 1%.

<details>
<summary>Подсказка</summary>

Скопируйте два тестовых случая из Урока 21.11 (раздел 6 и Упражнение 2) в один файл: в файле теста может быть несколько записей в `tests:`. Для хороших переходов замените ряд на `http_requests_total{path="/{code}", status="302"}`: успешный статус перехода — 302, а не 200.

</details>

**Готово, когда:**

```bash
curl -s 'localhost:9090/api/v1/targets?state=active' \
  | jq -r '.data.activeTargets[] | select(.labels.namespace == "shortener") | [.labels.pod, .health] | @tsv'
docker run --rm -v "$PWD/deploy/rules":/work -w /work --entrypoint promtool prom/prometheus:v3.16.0 test rules slo-rules-test.yml
```

Ожидается (с port-forward к Prometheus на 9090; имена подов будут другими):

```text
shortener-5d5598c9c7-b9vzq	up
shortener-5d5598c9c7-flttz	up
Processing test file: slo-rules-test.yml
  SUCCESS
```

Коммит: `M3: metrics in Prometheus and a tested SLO`.

#### M4 — Helm-чарт

**Цель:** Deployment, Service и ConfigMap сервиса создаются чартом; релизы и откаты работают.

**Как действовать:** удалите написанные вручную Deployment, Service и ConfigMap сервиса (PostgreSQL и Secret оставьте: данные и секреты живут дольше релизов). Запустите `helm create shortener` в `chart/` и адаптируйте результат (Урок 21.6): репозиторий и тег образа, `replicaCount: 2`, порт контейнера 8080, пробы на `/readyz` и `/healthz`, ресурсы, аннотации Prometheus в `podAnnotations`, `templates/configmap.yaml` на основе карты `config:` из `values.yaml` и `envFrom`, который читает ConfigMap и существующий Secret. Уберите то, чем не пользуетесь (ingress, HTTPRoute, HPA (HorizontalPodAutoscaler, он добавляет поды под нагрузкой), тестовый под), чтобы чарт оставался читаемым.

<details>
<summary>Подсказка: ConfigMap и Secret в чарте</summary>

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "shortener.fullname" . }}-config
  labels:
    {{- include "shortener.labels" . | nindent 4 }}
data:
  {{- range $key, $value := .Values.config }}
  {{ $key }}: {{ $value | quote }}
  {{- end }}
```

В `templates/deployment.yaml` над `ports:` контейнера:

```yaml
          envFrom:
            - configMapRef:
                name: {{ include "shortener.fullname" . }}-config
            - secretRef:
                name: {{ .Values.existingSecret }}
```

а в `values.yaml` — карту `config:` (все значения — строки в кавычках) и `existingSecret: shortener-secrets`. Чарт никогда не содержит секретных значений; он знает только **имя** Secret.

</details>

**Готово, когда:**

```bash
helm lint chart/shortener
helm upgrade --install shortener chart/shortener --namespace shortener --wait
helm list -n shortener
```

Ожидается (даты будут другими; столбец `APP VERSION` показывает `appVersion` из `Chart.yaml`, здесь это ещё значение по умолчанию `1.16.0` от `helm create` — замените его своей версией):

```text
==> Linting chart/shortener
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

```text
NAME     	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART          	APP VERSION
shortener	shortener	1       	2026-11-18 13:58:04.134089 +0500 +05	deployed	shortener-0.1.0	1.16.0     
```

Проверка перехода из M2 по-прежнему печатает `302 https://www.rust-lang.org/learn`. Коммит: `M4: Helm chart`.

#### M5 — Конвейер CI

**Цель:** каждый push и pull request запускает проверки Rust, проверяет чарт, тестирует правила и собирает образ; на `main` образ ещё и отправляется в GHCR.

**Как действовать:** напишите `.github/workflows/p21-shortener.yml` в **корне репозитория** (почему — объясняют Уроки 09.12 и 09.13); образцом служит workflow из Урока 21.7. Запускайте только для изменений в проекте: `paths: ["code/p21-shortener/**", ".github/workflows/p21-shortener.yml"]`. Задайте `defaults: run: working-directory: code/p21-shortener` и передайте `Swatinem/rust-cache@v2` параметр `workspaces: code/p21-shortener`. Задание сборки образа `needs` два других, имеет `permissions: packages: write`, входит в `ghcr.io` через `${{ secrets.GITHUB_TOKEN }}` и отправляет образ только на `main` (для pull request и других веток образ только собирается) и использует `docker/build-push-action@v7` с `context: code/p21-shortener`. Ваши интеграционные тесты запускают контейнеры (Урок 16.14); на Ubuntu-раннерах GitHub есть Docker, поэтому там они тоже работают.

<details>
<summary>Подсказка: задание сборки образа</summary>

```yaml
  image:
    needs: [rust, chart-and-rules]
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v7
      - uses: docker/setup-buildx-action@v4
      - uses: docker/login-action@v4
        if: github.ref == 'refs/heads/main'
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v7
        with:
          context: code/p21-shortener
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ghcr.io/${{ github.repository_owner }}/p21-shortener:${{ github.sha }}
```

Для pull request `github.ref` равен `refs/pull/<номер>/merge`, поэтому образ собирается, но не отправляется. Имена образов в GHCR должны быть в нижнем регистре; если в вашем имени на GitHub есть заглавные буквы, впишите владельца в нижнем регистре вручную. Ставьте тег по коммиту (`github.sha`), чтобы каждый образ можно было связать с его кодом.

</details>

**Готово, когда:** из корня репозитория `actionlint` ничего не печатает и завершается с кодом 0:

```bash
actionlint; echo "exit=$?"
```

```text
exit=0
```

Если репозиторий есть на GitHub, запуск после следующего push зелёный, а на `main` образ появляется в разделе **Packages**. Если репозитория на GitHub нет, workflow запуститься не может (Урок 09.12); ваша проверка — `actionlint`. Коммит: `M5: CI pipeline`.

#### M6 — Учебный выпуск, ранбук и README

**Цель:** вы можете выпустить новую версию, проверить её, откатиться, и другой инженер может сделать то же по вашему README.

**Как действовать:** измените что-то заметное (например, строку лога при запуске), задайте версию `0.2.0` в `Cargo.toml` (и, как хорошая практика, `appVersion` в `Chart.yaml`), соберите и загрузите `course-p21-shortener:0.2.0` и выпустите её командой `helm upgrade ... --set image.tag=0.2.0`. Откатитесь командой `helm rollback shortener 1`. Напишите ранбук для `ShortenerRedirectsFastBurn` (его форму показывает Урок 21.12). README должен отвечать: что делает сервис; как создать кластер, Secret и PostgreSQL; как установить и обновить чарт; как откатиться; где лежат SLO, оповещение и ранбук; и как минимум три проектных решения, например «PostgreSQL не входит в чарт, потому что данные должны переживать релизы» или «чарт только называет Secret; секретные значения никогда не попадают в Git».

<details>
<summary>Подсказка</summary>

`helm history` показывает каждую ревизию. Помните: изменение только ConfigMap не перезапускает поды. Добавьте в шаблон пода вашего чарта аннотацию `checksum/config` из Урока 21.6, чтобы каждое изменение конфигурации перезапускало Deployment.

</details>

**Готово, когда:**

```bash
helm history shortener -n shortener
```

Ожидается после одного обновления и одного отката (даты будут другими):

```text
REVISION	UPDATED                 	STATUS    	CHART          	APP VERSION	DESCRIPTION     
1       	Wed Nov 18 13:58:04 2026	superseded	shortener-0.1.0	1.16.0     	Install complete
2       	Wed Nov 18 13:58:28 2026	superseded	shortener-0.1.0	1.16.0     	Upgrade complete
3       	Wed Nov 18 13:58:29 2026	deployed  	shortener-0.1.0	1.16.0     	Rollback to 1   
```

`kubectl -n shortener get deployment shortener -o jsonpath='{.spec.template.spec.containers[0].image}'` снова печатает `course-p21-shortener:0.1.0`, а проверка перехода из M2 по-прежнему работает. Коммит: `M6: release drill, runbook and README`. Когда закончите, `kind delete cluster --name course-21-13`.

> 🔒 **Безопасность:** перед каждым коммитом проверяйте, что `git diff --cached` не содержит токенов, паролей и `DATABASE_URL` с паролем. Если секрет хоть раз попал в Git, убрать его из следующего коммита недостаточно: он остаётся в истории. Смените (ротируйте) секрет, как описывает [Урок 10.8 — Управление секретами](../10-security-fundamentals/08-secrets-management.md).

> 🧪 **Тестирование:** теперь у проекта пять уровней проверок: модульные и интеграционные тесты (`cargo test`), чарт (`helm lint`), правила (`promtool test rules`), workflow (`actionlint`) и дымовой тест после каждого выпуска (проверка перехода из M2 или цикл дымового теста из Урока 21.12). Запишите последний в README как первое, что нужно сделать после `helm upgrade`.

### Дополнительные цели

1. **Инфраструктура как код:** создайте пространство имён, Secret (из переменных с пометкой `sensitive = true`) и объекты PostgreSQL через OpenTofu и провайдер Kubernetes (Урок 21.8), а чарт установите через провайдер Helm. Состояние держите вне Git.
2. **SLO задержки:** добавьте второй SLO «99% переходов выполняются не дольше чем за 100 мс» (Урок 21.11, Упражнение 3) с тестом.
3. **Дашборд Grafana как код:** подготовьте из ConfigMap дашборд с частотой запросов, долей ошибок, задержкой p95 и остатком бюджета ошибок (Урок 21.9).
4. **Трассировки и логи в кластере:** разверните коллектор и Jaeger из Урока 21.10 и задайте `OTEL_EXPORTER_OTLP_ENDPOINT` в ConfigMap; переведите логи на JSON (JavaScript Object Notation).
5. **Сквозное задание в CI:** добавьте задание, которое создаёт кластер kind на раннере (официальное `helm/kind-action`), устанавливает PostgreSQL и чарт и запускает дымовой тест.
6. **Redis:** запустите в кластере и Redis (официальный образ, тому для кэша не нужен) и проверьте, что на втором переходе появляется `x-cache: hit`.

## Вопросы для самопроверки

1. Почему в этом проекте PostgreSQL не входит в Helm-чарт сервиса?

<details>
<summary>Ответ</summary>

Данные должны переживать релизы. `helm uninstall` или неудачное изменение чарта никогда не должны удалять базу данных или её том. Кроме того, сервис и база меняются с разной скоростью и у них разные владельцы.

</details>

2. Почему в кластере `APP_HOST` должен быть `0.0.0.0`?

<details>
<summary>Ответ</summary>

Программа, которая слушает `127.0.0.1`, принимает соединения только изнутри своего контейнера. Service отправляет трафик на IP-адрес (IP — Internet Protocol, межсетевой протокол) пода, а это работает, только если программа слушает все интерфейсы.

</details>

3. Какая проба вызывает `/readyz`, а какая `/healthz`, и почему не наоборот?

<details>
<summary>Ответ</summary>

Проба готовности вызывает `/readyz` (доступен ли PostgreSQL? если нет — убрать под из Service). Проба живости вызывает `/healthz` (жив ли процесс?). Если бы проба живости проверяла базу, короткий сбой базы заставил бы Kubernetes перезапустить все поды сервиса, а это никому не помогает (Урок 21.5).

</details>

4. Как чарт получает `ADMIN_TOKEN`, если токена нет в Git?

<details>
<summary>Ответ</summary>

Secret `shortener-secrets` создаётся отдельно (`kubectl create secret generic` или OpenTofu с секретной переменной). Чарт содержит только его имя (`existingSecret`) и читает его через `envFrom: secretRef`.

</details>

5. Почему образ помечают тегом версии или коммита, а не `latest`?

<details>
<summary>Ответ</summary>

Точный тег говорит, какой код запущен, делает возможными откаты (`0.1.0` никуда не делся) и позволяет Kubernetes заметить изменение шаблона пода. `latest` меняется под тем же именем и скрывает, что развёрнуто.

</details>

6. Что является источником истины после `helm rollback` во время инцидента и что нужно сделать дальше?

<details>
<summary>Ответ</summary>

По-прежнему репозиторий Git. Откат исправил кластер; теперь отмените или исправьте изменение в Git, чтобы следующий запуск конвейера или `helm upgrade` не вернул проблему (Урок 21.12).

</details>

## Итоги

- Развёртывание, близкое к продакшену, разделяет образ, конфигурацию (ConfigMap), секреты (Secret) и данные (PVC) и держит описание всего этого в Git.
- У проб разные задачи: готовность управляет трафиком, живость перезапускает зависшие процессы.
- Helm-чарт превращает манифесты в версионированные релизы с `helm upgrade`, `helm history` и `helm rollback`; он называет Secret, но никогда их не содержит.
- Конвейер CI проверяет код, чарт, правила и workflow и собирает образ с тегом коммита.
- Метрики, SLO с протестированным оповещением по скорости расходования и ранбук делают сервис управляемым; README делает его управляемым для других.

## Источники и что почитать

- [Kubernetes — Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) — PVC, PersistentVolume и режимы доступа.
- [Kubernetes — Resource management for pods and containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) — запросы и лимиты.
- [Helm — Chart template guide](https://helm.sh/docs/chart_template_guide/) и [Chart best practices](https://helm.sh/docs/chart_best_practices/) — написание шаблонов и значений; приём с перезапуском Deployment при изменении ConfigMap описан в [tips and tricks](https://helm.sh/docs/howto/charts_tips_and_tricks/).
- [Docker Official Image «postgres»](https://hub.docker.com/_/postgres) — переменные окружения вроде `POSTGRES_PASSWORD` и `PGDATA`.
- [GitHub Docs — Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry) и [docker/build-push-action](https://github.com/docker/build-push-action) — отправка образов из workflow.
- [kind — Quick start](https://kind.sigs.k8s.io/docs/user/quick-start/) — загрузка образов и параметры кластера.

---

[🏠 Начало](../START-HERE.md) · [← 21.12 Инциденты, дежурства и постмортемы](12-incidents-on-call-and-postmortems.md) · [📋 Этап 21](README.md) · [21.14 Контрольная точка →](14-checkpoint.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
