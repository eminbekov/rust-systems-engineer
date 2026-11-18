<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 21.13 Проект: ваш сервис в Kubernetes](13-project-run-your-service-on-kubernetes.md) · [📋 Этап 21](README.md) · [Этап 22 — Специализации →](../22-specializations/README.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/21-cloud-native-infrastructure/14-checkpoint.md) <!-- nav -->

# Урок 21.14 — Контрольная точка

> **Этап 21 — Облачная (cloud-native) инфраструктура** · Урок 14 из 14 · ⏱ около 90 минут

## Что вы узнаете

- Можете ли вы объяснить, как Kubernetes запускает, настраивает, обновляет и восстанавливает сервис.
- Можете ли вы упаковать и выпустить сервис и описать инфраструктуру как код.
- Можете ли вы наблюдать за сервисом через метрики, логи и трассировки, поставить SLO (Service Level Objective, цель уровня обслуживания) и поднимать тревогу по скорости расходования бюджета.
- Можете ли вы отреагировать на инцидент и написать полезный постмортем.
- Какой урок этого этапа перечитать для каждого пробела.

## Перед началом

- Вы прошли [Урок 21.13 — Проект: ваш сервис в Kubernetes](13-project-run-your-service-on-kubernetes.md) или хотя бы его вехи M1–M4.
- Проходите самопроверку **не подглядывая** в уроки. Запишите ответы, потом откройте скрытые ответы. Практические задания можно делать с открытыми уроками.
- Инструменты: Docker (для `promtool`) и для Задания 5 — кластер kind. Файлы кладите в `~/projects/rust/code/p21-checkpoint`.

## Новые слова

| Слово | Значение |
|---|---|
| **оповещение о медленном расходовании** (slow-burn alert) | Оповещение SLO для умеренной скорости расходования в длинных окнах (здесь 6 за 6 часов и 30 минут). Оно создаёт задачу, а не будит человека. |
| **задача** (ticket) | Задача в трекере команды, которую выполняют в рабочее время. Противоположность вызова дежурного. |

## Вопросы для самопроверки

1. Что означает «декларативный» в Kubernetes и какой компонент делает объявленное состояние правдой?

<details>
<summary>Ответ</summary>

Вы объявляете желаемое состояние в манифесте («3 реплики этого образа»); контроллеры снова и снова сравнивают его с фактическим и действуют по разнице (цикл согласования, Урок 21.1). Например, контроллер Deployment создаёт или удаляет поды.

</details>

2. Чем отличаются под, ReplicaSet и Deployment?

<details>
<summary>Ответ</summary>

Под запускает один или несколько контейнеров. ReplicaSet поддерживает фиксированное число одинаковых подов. Deployment управляет ReplicaSet, чтобы можно было постепенно обновлять шаблон пода и откатываться.

</details>

3. Как Service находит свои поды и чем это полезно?

<details>
<summary>Ответ</summary>

По селектору меток, например `app: orders`. Поды приходят и уходят с новыми именами и адресами; Service даёт им одно постоянное имя и адрес и распределяет соединения по готовым подам.

</details>

4. Ваш сервис слушает `127.0.0.1:8080` в контейнере. Под в состоянии `Running`, но через Service — «connection refused». Почему?

<details>
<summary>Ответ</summary>

`127.0.0.1` принимает соединения только изнутри контейнера. Service подключается к IP-адресу (IP — Internet Protocol, межсетевой протокол) пода, поэтому программа должна слушать `0.0.0.0` (Урок 21.4).

</details>

5. Какие настройки идут в ConfigMap, а какие в Secret? Шифруется ли Secret?

<details>
<summary>Ответ</summary>

Несекретная конфигурация (порты, лимиты, URL — Uniform Resource Locator, веб-адреса — без паролей) — в ConfigMap; пароли, токены и ключи — в Secret. По умолчанию Secret только закодирован в base64 (обратимая текстовая кодировка, а не шифрование): защищайте его правами доступа и держите его значения вне Git (Урок 21.5).

</details>

6. Что происходит, когда не проходит проба готовности, и что — когда не проходит проба живости?

<details>
<summary>Ответ</summary>

Готовность: под убирается из эндпоинтов Service и не получает трафик, но продолжает работать. Живость: Kubernetes убивает и перезапускает контейнер. Поэтому проба живости никогда не должна зависеть от базы данных.

</details>

7. Что Helm добавляет к простым манифестам? Назовите две команды, которые этим пользуются.

<details>
<summary>Ответ</summary>

Шаблоны со значениями, чтобы один чарт обслуживал многие окружения, и **релизы** с историей ревизий. `helm upgrade --install` создаёт или обновляет релиз; `helm rollback <release> <revision>` возвращает к старой ревизии (`helm history` их перечисляет). Kustomize вместо этого накладывает на простой YAML патчи из оверлеев, без шаблонов (Урок 21.6).

</details>

8. Чем отличаются непрерывная интеграция, непрерывная доставка и непрерывное развёртывание (вместе — CI/CD)?

<details>
<summary>Ответ</summary>

Интеграция: каждое изменение автоматически собирается и тестируется. Доставка: каждое прошедшее проверки изменение превращается в кандидата на выпуск (например, образ в реестре), который *может* уйти в продакшен по одному решению. Развёртывание: оно уходит в продакшен автоматически, без шага человека (Урок 21.7).

</details>

9. Зачем OpenTofu файл состояния и почему его нельзя хранить в Git?

<details>
<summary>Ответ</summary>

Состояние связывает блоки ресурсов с настоящими объектами; без него OpenTofu не узнал бы, что удалять, когда блок убран. Оно содержит секретные значения и должно блокироваться во время применения, поэтому его место — в защищённом бэкенде, а не в Git (Урок 21.8).

</details>

10. Кто-то вручную изменил число реплик Deployment. Что сделает `tofu plan` (или следующий релиз Helm) и какое правило за этим стоит?

<details>
<summary>Ответ</summary>

Он покажет дрейф, а следующее применение вернёт значение. Правило: управляемую инфраструктуру меняют только через её источник истины — файлы в Git; у каждого объекта ровно один владелец.

</details>

11. Почему Prometheus собирает метрики с каждого пода, а не через Service?

<details>
<summary>Ответ</summary>

Через Service каждый сбор попадает на случайный под, поэтому счётчик прыгал бы между значениями разных подов, а `rate()` давал бы неверный результат. Каждый под — отдельная цель; запросы складывают их через `sum`.

</details>

12. Напишите запрос PromQL (Prometheus Query Language, язык запросов Prometheus) для доли ошибок 5xx маршрута заказов за 5 минут.

<details>
<summary>Ответ</summary>

```text
sum(rate(http_requests_total{path="/orders/{id}", status=~"5.."}[5m]))
  / sum(rate(http_requests_total{path="/orders/{id}"}[5m]))
```

</details>

13. Почему у меток метрик должно быть мало значений? Приведите одну плохую и одну хорошую метку.

<details>
<summary>Ответ</summary>

Каждое новое значение создаёт новые временные ряды, которые стоят памяти в сервисе и в Prometheus. Плохо: `user_id`, сырой путь `/orders/7`. Хорошо: шаблон маршрута `/orders/{id}`, `status`, `method`.

</details>

14. Откуда читает `kubectl logs` и зачем нужен сборщик логов?

<details>
<summary>Ответ</summary>

Из файлов логов, которые среда выполнения контейнеров пишет на узле, а kubelet отдаёт. Они ротируются и удаляются вместе с подом или узлом, поэтому агент узла (DaemonSet) должен копировать их в хранилище логов (Урок 21.10).

</details>

15. SLO — 99,9% за 30 дней. Каков бюджет ошибок в минутах полного простоя и какова скорость расходования при 0,5% ошибок?

<details>
<summary>Ответ</summary>

`0.001 × 30 × 24 × 60 = 43.2` минуты. Скорость `0.005 / 0.001 = 5`: бюджета хватит на 6 дней.

</details>

16. Зачем оповещению о быстром расходовании и 1-часовое, и 5-минутное окно?

<details>
<summary>Ответ</summary>

Длинное окно доказывает, что проблема реальна и велика; короткое — что она всё ещё происходит, поэтому оповещение прекращается вскоре после исправления.

</details>

17. Во время инцидента поды зелёные, но половина запросов падает. Что вы спросите первым делом и что сделаете?

<details>
<summary>Ответ</summary>

«Что изменилось?» (`kubectl rollout history`, недавние выпуски). Если вероятная причина — недавнее изменение, сначала смягчите последствия откатом, затем подтвердите дымовым тестом, потом исправьте источник истины и разбирайтесь (Урок 21.12).

</details>

18. Назовите три части постмортема без поиска виноватых.

<details>
<summary>Ответ</summary>

Любые три из: краткое описание и влияние; хронология с временем; способствующие факторы (найденные несколькими «почему?», без обвинения людей); что прошло хорошо; задачи по итогам с ответственными и сроками.

</details>

## Практические задания

### Задание 1 — Бюджет и скорость расходования на бумаге

У сервиса SLO доступности 99,5% за 30 дней. За последние 30 дней он обработал 8 000 000 запросов, и 30 000 из них упали с 5xx. Прямо сейчас доля ошибок за последний час — 4%.

1. Сколько сбоев допускает бюджет и сколько от него осталось?
2. Какова текущая скорость расходования? Сработало бы оповещение о быстром (14,4) или медленном (6) расходовании, если короткие окна согласны?
3. Что сейчас говорит типичная политика бюджета ошибок?

<details>
<summary>Подсказка</summary>

Допустимо = `(1 − SLO) × total`; скорость = доля ошибок / `(1 − SLO)`. Первый ответ можно проверить функцией `budget_left` из `code/p21-slo` (Урок 21.11).

</details>

<details>
<summary>Ответ</summary>

1. Допустимо: `0.005 × 8,000,000 = 40,000`. Потрачено: `30,000 / 40,000 = 75%`; осталось 25%.
2. Скорость `0.04 / 0.005 = 8`. Выше 6, ниже 14,4: срабатывает оповещение о медленном расходовании (задача), оповещение о быстром не вызывает дежурного. При такой скорости оставшиеся 25% 30-дневного бюджета закончатся примерно за `0.25 × 30 / 8 ≈ 0.9` дня.
3. Бюджет ещё есть: новые возможности можно выпускать, но команде стоит уже сейчас разобраться с причиной 4%; если бюджет закончится, политика остановит выпуск новых возможностей до его восстановления.

</details>

### Задание 2 — Ещё два вопроса на PromQL

По метрикам сервиса заказов (Урок 21.9) напишите запросы: (а) 99-й перцентиль задержки заказов за 5 минут; (б) частоту запросов по подам за 1 минуту, только для пространства имён `shop`.

<details>
<summary>Подсказка</summary>

(а) — это Запрос 4 из Урока 21.9 с другим квантилем и окном. (б) есть в «Попробуйте сами» того же урока.

</details>

<details>
<summary>Ответ</summary>

```text
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket{path="/orders/{id}"}[5m])))
```

```text
sum by (pod) (rate(http_requests_total{namespace="shop"}[1m]))
```

Почему это работает: `histogram_quantile` нужны корзины, просуммированные по подам, но сгруппированные по `le`; частота по подам после суммирования оставляет только метку `pod`.

</details>

### Задание 3 — Оповещение о медленном расходовании с тестами

В `code/p21-checkpoint/slow-burn.yml` напишите два правила записи (доля ошибок маршрута заказов за 30 минут и за 6 часов) и оповещение `OrdersErrorBudgetSlowBurn` для SLO 99,5%: скорость расходования выше 6 в обоих окнах, `for: 15m`, метка `severity: ticket`. Затем **напишите тест `promtool`** с двумя случаями: 4% ошибок в течение 8 часов — срабатывает на 7-м часу; 2% ошибок — не срабатывает никогда.

<details>
<summary>Подсказка</summary>

Возьмите за образец `rules/slo-rules.yml` и `rules/slo-rules-test.yml` из Урока 21.11. Для 8 часов данных по одному значению в минуту нужно `x480`. Файл теста может содержать несколько записей в `tests:`, у каждой свои `input_series`.

</details>

<details>
<summary>Ответ</summary>

`slow-burn.yml`:

```yaml
groups:
  - name: orders-slo-slow
    rules:
      - record: slo:orders_errors:ratio_rate30m
        expr: |
          sum(rate(http_requests_total{path="/orders/{id}", status=~"5.."}[30m]))
            / sum(rate(http_requests_total{path="/orders/{id}"}[30m]))
      - record: slo:orders_errors:ratio_rate6h
        expr: |
          sum(rate(http_requests_total{path="/orders/{id}", status=~"5.."}[6h]))
            / sum(rate(http_requests_total{path="/orders/{id}"}[6h]))
      - alert: OrdersErrorBudgetSlowBurn
        expr: |
          slo:orders_errors:ratio_rate6h > (6 * 0.005)
            and slo:orders_errors:ratio_rate30m > (6 * 0.005)
        for: 15m
        labels:
          severity: ticket
```

`slow-burn-test.yml`:

```yaml
rule_files:
  - slow-burn.yml

evaluation_interval: 1m

tests:
  # 4% errors for 8 hours: burn rate 8, above 6.
  - interval: 1m
    input_series:
      - series: 'http_requests_total{path="/orders/{id}", status="200"}'
        values: '0+96x480'
      - series: 'http_requests_total{path="/orders/{id}", status="500"}'
        values: '0+4x480'
    alert_rule_test:
      - eval_time: 7h
        alertname: OrdersErrorBudgetSlowBurn
        exp_alerts:
          - exp_labels:
              severity: ticket
  # 2% errors: burn rate 4, below 6, never a ticket.
  - interval: 1m
    input_series:
      - series: 'http_requests_total{path="/orders/{id}", status="200"}'
        values: '0+98x480'
      - series: 'http_requests_total{path="/orders/{id}", status="500"}'
        values: '0+2x480'
    alert_rule_test:
      - eval_time: 7h
        alertname: OrdersErrorBudgetSlowBurn
        exp_alerts: []
```

```bash
docker run --rm -v "$PWD":/work -w /work --entrypoint promtool prom/prometheus:v3.16.0 check rules slow-burn.yml
docker run --rm -v "$PWD":/work -w /work --entrypoint promtool prom/prometheus:v3.16.0 test rules slow-burn-test.yml
```

```text
Checking slow-burn.yml
  SUCCESS: 3 rules found

Processing test file: slow-burn-test.yml
  SUCCESS
```

Почему это работает: 4% — это в 8 раз больше бюджета 0,5%, поэтому оба окна задолго до 7 часов выше `6 × 0.005 = 0.03`, и `for: 15m` уже прошёл. 2% всё время остаются ниже 0,03.

</details>

### Задание 4 — Прочитайте инцидент

Только что прошёл выпуск. Вы видите вот это:

```text
NAME                      READY   STATUS             RESTARTS   AGE
orders-7878c5fd55-k69q8   0/1     ImagePullBackOff   0          45s
orders-f6597ddff-6zxg2    1/1     Running            0          50s
orders-f6597ddff-t52jf    1/1     Running            0          51s
```

Пострадали ли пользователи? Какая наиболее вероятная причина, какая команда это докажет и что вы сделаете?

<details>
<summary>Подсказка</summary>

Посчитайте готовые поды старого ReplicaSet. Затем посмотрите на события пода, который не готов.

</details>

<details>
<summary>Ответ</summary>

Пользователи не пострадали: оба старых пода готовы и обслуживают трафик; постепенное обновление не удаляет старый под, пока новый не готов. Новый образ не удаётся скачать — скорее всего, неверный тег или образа нет. `kubectl -n shop describe pod orders-7878c5fd55-k69q8` показывает это в событиях (`Failed to pull image ... repository does not exist`, `ErrImagePull`). Откатитесь командой `kubectl -n shop rollout undo deployment/orders` (или исправьте тег в Git и выпустите снова), затем исправьте конвейер так, чтобы выпуск мог использовать только образ, который был собран и отправлен (Урок 21.12, Упражнение 3).

</details>

### Задание 5 — Инфраструктура как код по памяти

Создайте `course-21-14` через kind и в `code/p21-checkpoint/infra` напишите конфигурацию OpenTofu (провайдер Kubernetes), которая создаёт пространство имён `checkpoint` и ConfigMap `settings` с одним ключом `mode = "practice"`. Выполните `tofu init`, `tofu plan` и `tofu apply`. Затем вручную измените ConfigMap через `kubectl -n checkpoint edit configmap settings` и покажите дрейф через `tofu plan`. Удалите всё и удалите кластер.

<details>
<summary>Подсказка</summary>

Скопируйте `versions.tf` из Урока 21.8 и замените контекст на `kind-course-21-14`. Нужны два ресурса: `kubernetes_namespace_v1` и `kubernetes_config_map_v1`, второй ссылается на первый.

</details>

<details>
<summary>Ответ</summary>

Ресурсы (`main.tf`):

```hcl
resource "kubernetes_namespace_v1" "checkpoint" {
  metadata {
    name = "checkpoint"
  }
}

resource "kubernetes_config_map_v1" "settings" {
  metadata {
    name      = "settings"
    namespace = kubernetes_namespace_v1.checkpoint.metadata[0].name
  }

  data = {
    mode = "practice"
  }
}
```

`tofu plan` до первого применения заканчивается строкой `Plan: 2 to add, 0 to change, 0 to destroy.` После ручной правки (например, на `hand-edited`; `kubectl -n checkpoint patch configmap settings --type merge -p '{"data":{"mode":"hand-edited"}}'` делает то же без редактора) он показывает `~ update in-place` для `kubernetes_config_map_v1.settings` со строкой `~ "mode" = "hand-edited" -> "practice"` и `Plan: 0 to add, 1 to change, 0 to destroy.` `tofu destroy` заканчивается строкой `Destroy complete! Resources: 2 destroyed.`

Почему это работает: перед каждым планом OpenTofu обновляет сведения о настоящих объектах и сравнивает их с файлами, точно как в разделе 4 Урока 21.8.

</details>

## Если что-то не получилось

| Тема | Какой урок перечитать |
|---|---|
| Желаемое состояние, согласование, чего ждёт платформа | [21.1 Что значит cloud native](01-what-cloud-native-means.md) |
| Поды, Deployment, ReplicaSet, Service, метки | [21.2 Понятия Kubernetes](02-kubernetes-concepts.md) |
| kind, kubectl, контексты | [21.3 Локальный кластер с kind и kubectl](03-a-local-cluster-with-kind.md) |
| Образы в кластере, постепенные обновления, `0.0.0.0` | [21.4 Развёртывание сервиса на Rust в Kubernetes](04-deploying-a-rust-service-to-kubernetes.md) |
| ConfigMap, Secret, пробы готовности и живости | [21.5 ConfigMap, Secret и пробы работоспособности](05-configmaps-secrets-and-probes.md) |
| Чарты, значения, релизы, оверлеи | [21.6 Упаковка с Helm и Kustomize](06-helm-and-kustomize.md) |
| CI, CD, конвейеры сборки и выпуска | [21.7 CI/CD: от коммита до продакшена](07-ci-cd-from-commit-to-production.md) |
| OpenTofu, состояние, дрейф, `tofu test` | [21.8 Инфраструктура как код с OpenTofu](08-infrastructure-as-code-with-opentofu.md) |
| Prometheus, PromQL, гистограммы, кардинальность, Grafana | [21.9 Метрики с Prometheus и Grafana](09-metrics-with-prometheus-and-grafana.md) |
| `kubectl logs`, сбор логов, трассировки в кластере | [21.10 Логи и трассировки в кластере](10-logs-and-traces-in-a-cluster.md) |
| SLI (Service Level Indicator, показатель уровня обслуживания), SLO, бюджет ошибок, оповещения по скорости расходования, `promtool` | [21.11 Основы SRE: SLI, SLO и бюджет ошибок](11-sre-basics-slis-slos-and-error-budgets.md) |
| Серьёзность, дежурства, смягчение последствий, постмортемы | [21.12 Инциденты, дежурства и постмортемы](12-incidents-on-call-and-postmortems.md) |
| Всё вместе | [21.13 Проект: ваш сервис в Kubernetes](13-project-run-your-service-on-kubernetes.md) |

## Что дальше

Теперь вы умеете запускать сервис на Rust так, как это делает команда платформы: упакованным, выпущенным, под наблюдением, измеренным относительно цели и спокойно починенным, когда он ломается. Следующий этап, [Этап 22 — Специализации](../22-specializations/README.md), позволяет углубиться в одно направление. Если этот этап понравился вам больше всего, выберите **направление cloud-native**: там вы напишете на Rust *оператор* Kubernetes с крейтом `kube` — собственный контроллер с собственным циклом согласования, той же идеей, на которой работают Deployment. После этого [Этап 23 — Итоговые проекты](../23-capstone-projects/README.md) соберёт всё вместе в одном большом проекте.

Прежде чем идти дальше, удалите все оставшиеся кластеры: `kind get clusters`, затем `kind delete cluster --name <name>` для каждого.

## Источники и что почитать

- [Документация Kubernetes — Concepts](https://kubernetes.io/docs/concepts/) — официальный справочник по всем объектам этого этапа.
- [Документация OpenTofu](https://opentofu.org/docs/) — язык, командная утилита и состояние.
- [Документация Prometheus](https://prometheus.io/docs/introduction/overview/) — метрики, PromQL, правила и модульные тесты.
- [The Site Reliability Workbook](https://sre.google/workbook/table-of-contents/) — SLO, оповещения, дежурства и постмортемы на практике, от авторов самого понятия SRE (Site Reliability Engineering, инженерия надёжности сайтов).

---

[🏠 Начало](../START-HERE.md) · [← 21.13 Проект: ваш сервис в Kubernetes](13-project-run-your-service-on-kubernetes.md) · [📋 Этап 21](README.md) · [Этап 22 — Специализации →](../22-specializations/README.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
