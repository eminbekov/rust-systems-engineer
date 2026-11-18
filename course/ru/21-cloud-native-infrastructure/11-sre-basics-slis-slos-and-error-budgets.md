<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 21.10 Логи и трассировки в кластере](10-logs-and-traces-in-a-cluster.md) · [📋 Этап 21](README.md) · [21.12 Инциденты, дежурства и постмортемы →](12-incidents-on-call-and-postmortems.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) <!-- nav -->

# Урок 21.11 — Основы SRE: SLI, SLO и бюджет ошибок

> **Этап 21 — Облачная (cloud-native) инфраструктура** · Урок 11 из 14 · ⏱ около 120 минут

## Что вы узнаете

- Что такое SRE (Site Reliability Engineering) и почему «надёжность 100%» — неправильная цель.
- Как измерять надёжность со стороны пользователя с помощью SLI (Service Level Indicator) и как ставить для него цель — SLO (Service Level Objective).
- Что такое бюджет ошибок, как его посчитать и как команда с его помощью выбирает между новыми возможностями и работой над надёжностью.
- Что такое скорость расходования бюджета (burn rate) и как поднимать тревогу по ней, а не по каждой ошибке.
- Как записать SLO в виде правил записи и оповещения Prometheus, протестировать их через `promtool` и увидеть срабатывание оповещения в кластере kind.

## Перед началом

- Вы прошли [Урок 21.10 — Логи и трассировки в кластере](10-logs-and-traces-in-a-cluster.md).
- Вы знаете счётчик запросов `http_requests_total` и запрос доли ошибок из [Урока 21.9](09-metrics-with-prometheus-and-grafana.md); в этом уроке используются тот же сервис, образ `course-p21-orders:0.1.0`, и тот же чарт Prometheus.
- Вы прочитали [Урок 20.1 — Что такое надёжность](../20-reliability-and-hardening/01-what-reliability-means.md) и чек-лист готовности к продакшену из [Урока 20.9](../20-reliability-and-hardening/09-production-readiness-checklist.md). Этот урок превращает «сервис достаточно надёжен» в число, которое можно проверить.
- Docker Desktop запущен; `promtool` мы запускаем из официального образа `prom/prometheus`, поэтому устанавливать его не нужно.

## Новые слова

| Слово | Значение |
|---|---|
| **SRE** (Site Reliability Engineering, инженерия надёжности сайтов) | Подход к эксплуатации сервисов, начатый в Google: надёжность — инженерная задача с измеримыми целями, автоматизацией и общими правилами для разработчиков и эксплуатации. |
| **SLI** (Service Level Indicator, показатель уровня обслуживания) | Измеряемое число, которое показывает, насколько хорошо сервис работает для пользователей, обычно «хорошие события / все допустимые события», например доля успешных запросов. |
| **SLO** (Service Level Objective, цель уровня обслуживания) | Цель для SLI за временное окно: «99,5% запросов заказов успешны, измерено за 30 дней». |
| **SLA** (Service Level Agreement, соглашение об уровне обслуживания) | Обещание в договоре с клиентами с последствиями (возврат денег) при нарушении. Всегда мягче внутреннего SLO. |
| **бюджет ошибок** (error budget) | Объём сбоев, который допускает SLO: `1 − SLO`. Для 99,5% это 0,5% всех запросов в окне. |
| **скорость расходования** (burn rate) | Как быстро тратится бюджет ошибок. При значении 1 весь бюджет заканчивается ровно к концу окна; при 10 — в десять раз быстрее. |
| **правило записи** (recording rule) | Правило Prometheus, которое регулярно вычисляет запрос и сохраняет результат как новый временной ряд со своим именем. |
| **правило оповещения** (alerting rule) | Правило Prometheus с условием; пока условие истинно, Prometheus поднимает оповещение. |
| **Alertmanager** | Компонент Prometheus, который принимает оповещения и отправляет уведомления (пейджер, чат, почта). Мы держим его выключенным и смотрим оповещения в самом Prometheus. |
| **`promtool`** | Утилита командной строки Prometheus, которая проверяет файлы правил и запускает для них модульные тесты. |
| **вызов дежурного** (page) | Срочное уведомление, которое поднимает дежурного (Урок 21.12). |

## 1. Почему не 100%?

«Сервис никогда не должен падать» звучит правильно, но это плохая цель, по трём причинам.

1. **Это невозможно.** Сети теряют пакеты, диски ломаются, в развёртываниях бывают ошибки. Урок 20.1 показал, что сбои — это нормально; вопрос только в том, *сколько* их.
2. **Пользователи не видят разницы.** Пользователь мобильной сети, которая работает 99% времени, не заметит, надёжен ли ваш сервис на 99,99% или на 99,999%. Последние девятки стоят дорого, и их никто не замечает.
3. **Это останавливает любые изменения.** Каждое развёртывание — риск. Команда, которой нельзя ошибаться никогда, ничего не может выпустить.

**SRE** (Site Reliability Engineering) превращает это в числа: выберите уровень надёжности, при котором пользователи довольны, измеряйте его, а разницу до 100% используйте как *бюджет* на риск. Этот бюджет тратится на развёртывания, эксперименты и изредка на плохой день.

## 2. SLI: измеряйте то, что чувствуют пользователи

**SLI** (Service Level Indicator) — измерение с точки зрения пользователя. Самая полезная форма — отношение:

```text
          good events
  SLI = ───────────────    (a number between 0 and 1, often shown as a percentage)
         valid events
```

| Вид SLI | Хорошее событие | Допустимое событие | Из наших метрик |
|---|---|---|---|
| доступность (availability) | запрос не упал по нашей вине (не 5xx) | каждый запрос заказа | `http_requests_total` по `status` |
| задержка (latency) | запрос завершился быстрее чем за 100 мс | каждый запрос заказа | `http_request_duration_seconds_bucket{le="0.1"}` |

Решения, спрятанные в этой таблице:

- **404 — это хорошо.** Пользователь спросил заказ, которого нет, и получил правильный ответ. Статус 4xx означает «ошибся клиент», 5xx — «ошиблись мы». Против нас считаются только 5xx.
- **Учитывается только настоящий пользовательский трафик.** Проверки здоровья и `/metrics` — не пользователи; мы фильтруем по `path="/orders/{id}"`.
- **Измеряйте как можно ближе к пользователю.** Наш сервис измеряет себя сам, поэтому запрос, который до него не дошёл (сломанный Service или сеть), невидим. Многие команды измеряют ещё и на балансировщике нагрузки или внешней проверкой.

## 3. SLO и SLA

**SLO** (Service Level Objective) — цель для SLI за **окно**:

> 99,5% запросов заказов успешны (не 5xx), измерено за последние 30 дней.

Важна каждая часть. *99,5%* — цель. *Запросы заказов* говорят, какой SLI. *30 дней* — **скользящее окно** (rolling window): каждый день вы смотрите ровно на 30 дней назад, поэтому плохой день учитывается месяц, а потом исчезает.

**SLA** (Service Level Agreement) — другое: договор с клиентами («99% в месяц, иначе вернём 10% денег»). SLA — деловое обещание; SLO — внутренняя цель команды. SLO всегда **строже** SLA, чтобы команда замечала проблемы и исправляла их задолго до нарушения договора.

> 📏 **Соглашение:** начинайте с небольшого числа SLO (по одному SLO доступности и одному SLO задержки на каждый сервис для пользователей) и ставьте первую цель чуть *ниже* того, что сервис даёт сегодня. SLO, нарушенный с первого дня, никого ничему не учит. Ужесточайте его позже, когда поймёте, что нужно пользователям.

## 4. Бюджет ошибок

**Бюджет ошибок** — то, что SLO разрешает сломать: `1 − SLO`. Небольшая программа на Rust делает числа наглядными. Создайте её в `code/p21-slo` (`cargo new code/p21-slo`) с таким `src/main.rs`:

```rust
/// Share of events that may fail: 1 - SLO. For 99.9% that is 0.001.
fn error_budget(slo: f64) -> f64 {
    1.0 - slo
}

/// Minutes of full outage the budget allows in a window of `days` days.
fn allowed_downtime_minutes(slo: f64, days: f64) -> f64 {
    error_budget(slo) * days * 24.0 * 60.0
}

/// How fast the budget is used: 1.0 means "exactly used up at the end of the window".
fn burn_rate(error_ratio: f64, slo: f64) -> f64 {
    error_ratio / error_budget(slo)
}

/// Share of the budget that is still left after `failed` of `total` events.
fn budget_left(slo: f64, failed: u64, total: u64) -> f64 {
    if total == 0 {
        return 1.0;
    }
    let allowed = error_budget(slo) * total as f64;
    1.0 - failed as f64 / allowed
}

fn main() {
    println!("{:>8}  {:>12}  {:>14}", "SLO", "budget", "downtime/30d");
    for slo in [0.99, 0.995, 0.999, 0.9999] {
        println!(
            "{:>7.2}%  {:>11.2}%  {:>10.1} min",
            slo * 100.0,
            error_budget(slo) * 100.0,
            allowed_downtime_minutes(slo, 30.0)
        );
    }

    let slo = 0.995;
    println!();
    println!("SLO 99.5%, 1,000,000 requests, 3,000 failed:");
    println!(
        "  budget left: {:.0}%",
        budget_left(slo, 3_000, 1_000_000) * 100.0
    );
    println!("  burn rate at 5% errors: {:.1}", burn_rate(0.05, slo));
}

#[cfg(test)]
mod tests {
    use super::*;

    fn close(a: f64, b: f64) -> bool {
        (a - b).abs() < 1e-9
    }

    #[test]
    fn three_nines_allow_43_minutes_in_30_days() {
        assert!(close(allowed_downtime_minutes(0.999, 30.0), 43.2));
    }

    #[test]
    fn errors_at_the_budget_burn_at_rate_one() {
        assert!(close(burn_rate(0.005, 0.995), 1.0));
    }

    #[test]
    fn no_traffic_means_full_budget() {
        assert_eq!(budget_left(0.995, 0, 0), 1.0);
    }
}
```

```bash
cargo run
cargo test
```

```text
     SLO        budget    downtime/30d
  99.00%         1.00%       432.0 min
  99.50%         0.50%       216.0 min
  99.90%         0.10%        43.2 min
  99.99%         0.01%         4.3 min

SLO 99.5%, 1,000,000 requests, 3,000 failed:
  budget left: 40%
  burn rate at 5% errors: 10.0
```

```text
running 3 tests
test tests::no_traffic_means_full_budget ... ok
test tests::errors_at_the_budget_burn_at_rate_one ... ok
test tests::three_nines_allow_43_minutes_in_30_days ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**Что происходит:**

1. Таблица показывает знаменитые «девятки». Каждая лишняя девятка делит бюджет на десять: 99,9% («три девятки») допускают всего 43,2 минуты полного простоя за 30 дней, 99,99% — около 4 минут, меньше одного медленного развёртывания.
2. При 1 000 000 запросов SLO 99,5% допускает 5000 сбоев. Произошло 3000, значит, осталось 40% бюджета.
3. `close` сравнивает числа с плавающей точкой с небольшим допуском, потому что `1.0 - 0.995` в двоичной плавающей точке не равно точно `0.005` (Урок 02.2).

> 🧪 **Тестирование:** маленькие чистые функции вроде `burn_rate` легко тестировать, а в расчётах SLO легко ошибиться в 100 раз (проценты против долей). Тестируйте известные точки: три девятки дают 43,2 минуты; ошибки ровно в пределах бюджета дают скорость расходования 1.

Бюджет становится полезным через **политику бюджета ошибок** (error budget policy) — короткий документ, о котором команда договаривается заранее, например:

- Бюджет остался: новые возможности выпускаются как обычно; запланированная рискованная работа (миграции, эксперименты) допустима.
- Бюджет исчерпан: выпуск новых возможностей этого сервиса останавливается. Пока бюджет не восстановится, выходят только исправления и работа над надёжностью. Постмортем инцидента, который его потратил (Урок 21.12), получает высший приоритет.

Политика превращает бесконечный спор («достаточно ли стабильно?») в число, которое разработчики и эксплуатация приняли заранее.

## 5. Скорость расходования: тревога по тому, что важно

Наивное оповещение говорит: «вызови меня, когда доля ошибок выше 0,5%». Оно срабатывает на каждом коротком всплеске, в три часа ночи, по проблемам, которые проходят сами, и люди учатся его игнорировать.

Правильный вопрос: **как скоро при такой скорости мы потратим бюджет?** Эта скорость и есть **скорость расходования** (burn rate):

```text
                current error ratio        0.05
  burn rate = ─────────────────────── = ──────── = 10      (5% errors, SLO 99.5%)
               error budget (1 − SLO)     0.005
```

При скорости 1 30-дневный бюджет тратится ровно за 30 дней. При скорости 10 — за 3 дня. SRE Workbook (см. источники) советует сочетать **длинное окно** (это реально?) с **коротким окном** (это всё ещё происходит?):

| Оповещение | Скорость | Длинное окно | Короткое окно | Потрачено бюджета при срабатывании | Действие |
|---|---|---|---|---|---|
| быстрое расходование (fast burn) | > 14,4 | 1 час | 5 минут | 2% за 1 час | вызвать дежурного |
| медленное расходование (slow burn) | > 6 | 6 часов | 30 минут | 5% за 6 часов | задача на рабочее время |

14,4 следует из расчёта: за 1 час 720-часового окна скорость 14,4 тратит `14.4 / 720 = 2%` бюджета. Короткое окно заставляет оповещение прекратиться вскоре после исправления проблемы, а не через час.

## 6. SLO в виде правил Prometheus

Запишите SLI как **правила записи**, чтобы отношение вычислялось один раз за оценку и хранилось под коротким именем, а оповещение — как **правило оповещения**. `rules/slo-rules.yml`:

```yaml
groups:
  - name: orders-slo
    rules:
      # SLI: share of order requests that failed (5xx), over two windows.
      - record: slo:orders_errors:ratio_rate5m
        expr: |
          sum(rate(http_requests_total{path="/orders/{id}", status=~"5.."}[5m]))
            / sum(rate(http_requests_total{path="/orders/{id}"}[5m]))
      - record: slo:orders_errors:ratio_rate1h
        expr: |
          sum(rate(http_requests_total{path="/orders/{id}", status=~"5.."}[1h]))
            / sum(rate(http_requests_total{path="/orders/{id}"}[1h]))

  - name: orders-slo-alerts
    rules:
      # SLO 99.5% -> error budget 0.005. Burn rate 14.4 uses 2% of a 30-day budget in 1 hour.
      - alert: OrdersErrorBudgetFastBurn
        expr: |
          slo:orders_errors:ratio_rate1h > (14.4 * 0.005)
            and slo:orders_errors:ratio_rate5m > (14.4 * 0.005)
        for: 2m
        labels:
          severity: page
        annotations:
          summary: "Orders are burning the error budget 14x too fast"
          description: "{{ $value | humanizePercentage }} of order requests fail (SLO allows 0.5%)."
```

**Что происходит:**

1. **Группа** вычисляется один раз за *интервал оценки* (evaluation interval). По умолчанию в чарте это одна минута (сбор метрик, как мы задали, идёт каждые 15 секунд).
2. `record:` сохраняет результат `expr` как новый ряд. Имя следует соглашению Prometheus `level:metric:operations`: агрегировано (`slo`), что (`orders_errors`), как (`ratio_rate5m`).
3. Оповещение сравнивает долю ошибок с `14.4 × budget`. «Доля ошибок > 14,4 × 0,005» — то же самое, что «скорость расходования > 14,4». `and` требует выполнения условия в обоих окнах.
4. `for: 2m`: условие должно оставаться истинным 2 минуты, прежде чем оповещение *сработает* (firing); до этого оно *ожидает* (pending). Так отсеиваются единичные плохие оценки.
5. `labels` добавляются к оповещению (по `severity` Alertmanager решает, кого вызывать); `annotations` — текст для людей. `{{ $value | humanizePercentage }}` вставляет значение выражения в процентах.

**Проверьте и протестируйте правила**, прежде чем они куда-то попадут. `promtool check rules` находит синтаксические ошибки; `promtool test rules` запускает **модульные тесты** на придуманных входных рядах. `rules/slo-rules-test.yml`:

```yaml
rule_files:
  - slo-rules.yml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      # 90 good and 10 failed requests per minute: 10% errors.
      - series: 'http_requests_total{path="/orders/{id}", status="200"}'
        values: '0+90x60'
      - series: 'http_requests_total{path="/orders/{id}", status="500"}'
        values: '0+10x60'
    promql_expr_test:
      - expr: round(slo:orders_errors:ratio_rate5m, 0.001)
        eval_time: 10m
        exp_samples:
          - labels: '{}'
            value: 0.1
    alert_rule_test:
      - eval_time: 10m
        alertname: OrdersErrorBudgetFastBurn
        exp_alerts:
          - exp_labels:
              severity: page
            exp_annotations:
              summary: "Orders are burning the error budget 14x too fast"
              description: "10% of order requests fail (SLO allows 0.5%)."
```

**Что происходит:**

1. `input_series` придумывает данные, по одному значению на `interval`. `'0+90x60'` означает «начать с 0 и 60 раз прибавить 90»: счётчик, который растёт на 90 в минуту в течение часа.
2. `promql_expr_test` проверяет запрос в заданный момент (`eval_time: 10m`). `round(..., 0.001)` округляет до трёх знаков; почему — см. «Частые ошибки».
3. `alert_rule_test` говорит, какие оповещения в этот момент должны **сработать**, с какими метками и текстами. К 10-й минуте условие истинно дольше, чем `for: 2m`.

Запустите обе проверки с `promtool` из официального образа Prometheus (из папки, в которой лежит `rules/`):

```bash
docker run --rm -v "$PWD/rules":/work -w /work --entrypoint promtool prom/prometheus:v3.16.0 check rules slo-rules.yml
docker run --rm -v "$PWD/rules":/work -w /work --entrypoint promtool prom/prometheus:v3.16.0 test rules slo-rules-test.yml
```

```text
Checking slo-rules.yml
  SUCCESS: 3 rules found

Processing test file: slo-rules-test.yml
  SUCCESS
```

## 7. SLO в кластере

Создайте `course-21-11`, загрузите `course-p21-orders:0.1.0` и примените `k8s/orders.yaml` из Урока 21.9, но с `FAIL_EVERY`, равным `"10"`: плохой день с 10% ошибок. Затем передайте правила Prometheus. Чарт читает дополнительные файлы правил из значения `serverFiles`; `k8s/prometheus-values.yaml` — это файл из Урока 21.9 плюс те же группы в `serverFiles.alerting_rules.yml`:

```yaml
# Only the Prometheus server; the extra components are switched off.
alertmanager:
  enabled: false
prometheus-pushgateway:
  enabled: false
kube-state-metrics:
  enabled: false
prometheus-node-exporter:
  enabled: false
server:
  global:
    scrape_interval: 15s
  persistentVolume:
    enabled: false
serverFiles:
  alerting_rules.yml:
    groups:
      - name: orders-slo
        rules:
          # SLI: share of order requests that failed (5xx), over two windows.
          - record: slo:orders_errors:ratio_rate5m
            expr: |
              sum(rate(http_requests_total{path="/orders/{id}", status=~"5.."}[5m]))
                / sum(rate(http_requests_total{path="/orders/{id}"}[5m]))
          - record: slo:orders_errors:ratio_rate1h
            expr: |
              sum(rate(http_requests_total{path="/orders/{id}", status=~"5.."}[1h]))
                / sum(rate(http_requests_total{path="/orders/{id}"}[1h]))

      - name: orders-slo-alerts
        rules:
          # SLO 99.5% -> error budget 0.005. Burn rate 14.4 uses 2% of a 30-day budget in 1 hour.
          - alert: OrdersErrorBudgetFastBurn
            expr: |
              slo:orders_errors:ratio_rate1h > (14.4 * 0.005)
                and slo:orders_errors:ratio_rate5m > (14.4 * 0.005)
            for: 2m
            labels:
              severity: page
            annotations:
              summary: "Orders are burning the error budget 14x too fast"
              description: "{{ $value | humanizePercentage }} of order requests fail (SLO allows 0.5%)."
```

```bash
kind create cluster --name course-21-11
kind load docker-image course-p21-orders:0.1.0 --name course-21-11
kubectl apply -f k8s/orders.yaml
helm install prometheus prometheus-community/prometheus --version 29.47.2 \
  --namespace monitoring --create-namespace -f k8s/prometheus-values.yaml
kubectl -n monitoring rollout status deployment/prometheus-server
kubectl -n shop run load --image=busybox:1.37 --restart=Never -- \
  sh -c 'while true; do for i in $(seq 1 100); do wget -q -O /dev/null http://orders/orders/$i 2>/dev/null; sleep 0.1; done; done'
kubectl -n monitoring port-forward service/prometheus-server 9090:80
```

Во втором терминале проверьте, что правила загружены и в порядке:

```bash
curl -s localhost:9090/api/v1/rules | jq -c '.data.groups[] | {name, rules: [.rules[] | {name, health}]}'
```

```text
{"name":"orders-slo","rules":[{"name":"slo:orders_errors:ratio_rate5m","health":"ok"},{"name":"slo:orders_errors:ratio_rate1h","health":"ok"}]}
{"name":"orders-slo-alerts","rules":[{"name":"OrdersErrorBudgetFastBurn","health":"ok"}]}
```

Подождите несколько минут, затем запросите SLI, скорость расходования и бюджет. Команды `curl` сокращены, как в Уроке 21.9 (`curl -s localhost:9090/api/v1/query --data-urlencode 'query=...' | jq -c '.data.result[] | [.metric, .value[1]]'`):

```text
slo:orders_errors:ratio_rate5m
[{"__name__":"slo:orders_errors:ratio_rate5m"},"0.09749641565756222"]

slo:orders_errors:ratio_rate5m / 0.005
[{},"19.499283131512442"]

1 - (sum(increase(http_requests_total{path="/orders/{id}", status=~"5.."}[30d]))
     / sum(increase(http_requests_total{path="/orders/{id}"}[30d]))) / 0.005
[{},"-18.53401567218997"]
```

**Что происходит:**

1. Около 9,7% запросов заказов падают: `FAIL_EVERY=10` в каждом поде, измеренное через `rate`, который немного оценивает.
2. Скорость расходования около 19,5: при такой скорости 30-дневного бюджета хватит на `30 / 19.5 ≈ 1.5` дня.
3. «Остаток бюджета» — это `1 − (error ratio / budget)` за окно. Нашему кластеру всего несколько минут, поэтому в 30-дневное окно попали только эти минуты, и результат около −18,5: сбоев уже примерно в 19,5 раза больше всего бюджета. Отрицательное число означает «потрачено, и ещё сверх того».

После `for: 2m` оповещение срабатывает. Его показывает API (Application Programming Interface) оповещений или страница **Alerts** в UI (User Interface):

```bash
curl -s localhost:9090/api/v1/alerts \
  | jq -c '.data.alerts[] | {name: .labels.alertname, severity: .labels.severity, state, description: .annotations.description}'
```

```text
{"name":"OrdersErrorBudgetFastBurn","severity":"page","state":"firing","description":"9.692% of order requests fail (SLO allows 0.5%)."}
```

Если включить Alertmanager (он входит в тот же чарт), это оповещение сейчас ушло бы по метке `severity: page` на пейджер дежурного. Что дежурный делает дальше — тема Урока 21.12.

> ⚠️ **Частая ошибка:** здесь правила живут дважды — в `rules/slo-rules.yml` (для `promtool`) и в файле значений (для чарта). Если они разойдутся, вы будете тестировать одно, а запускать другое. В настоящем проекте генерируйте один файл из другого в CI (Continuous Integration, Урок 21.7) или используйте Prometheus Operator, где правила — объекты Kubernetes (`PrometheusRule`), которые можно держать в одном файле.

## Частые ошибки

> ⚠️ **Частая ошибка:** точное сравнение вычисленного отношения в тесте `promtool`. Без `round` тест падает, хотя математика верна:

```text
  FAILED:
    expr: "slo:orders_errors:ratio_rate5m", time: 10m,
        exp: {__name__="slo:orders_errors:ratio_rate5m"} 1E-01
        got: {__name__="slo:orders_errors:ratio_rate5m"} 9.999999999999999E-02
```

Деление чисел с плавающей точкой редко даёт ровно `0.1`. Округляйте в тесте, как выше, или проверяйте оповещение, а не сырое число.

> ⚠️ **Частая ошибка:** считать 4xx сбоями. Тогда клиент, который спрашивает много несуществующих заказов, или сканер, перебирающий случайные URL (Uniform Resource Locator, веб-адреса), тратит *ваш* бюджет ошибок, и команду поднимают из-за чужих ошибок.

> ⚠️ **Частая ошибка:** SLO, на который никто не смотрит. SLO полезен, только если дашборд показывает бюджет, оповещение следит за скоростью расходования, а команда договорилась о политике на случай пустого бюджета.

> ⚠️ **Частая ошибка:** деление на нулевой трафик. Ночью без запросов `0 / 0` даёт «нет данных» (`NaN`), и оповещение не срабатывает и не снимается. Для оповещения по скорости расходования это приемлемо; для числа «остаток бюджета» решите, что означает отсутствие трафика (наша функция на Rust говорит: бюджет остался целиком).

## Попробуйте сами

1. Измените `FAIL_EVERY` на `"200"` (0,5% ошибок): `kubectl -n shop set env deployment/orders FAIL_EVERY=200`. Примерно через 3 минуты оповещение исчезнет из списка: 5-минутное окно заполнится здоровым трафиком, а скорость 1 — ровно в бюджете, вызывать некого. (1-часовое окно ещё помнит плохие минуты; именно поэтому оповещению нужны *оба* окна.)
2. В UI Prometheus откройте **Alerts** (<http://localhost:9090/alerts>) и посмотрите, как оповещение переходит из *inactive* в *pending*, а затем в *firing*, после того как вы вернёте `FAIL_EVERY` к `"10"`.
3. Запустите `cargo test` в `code/p21-slo`, заменив тело `close` на `a == b`. Какие тесты упадут и почему? (Два: `three_nines_allow_43_minutes_in_30_days` и `errors_at_the_budget_burn_at_rate_one`. Их результаты отличаются от 43,2 и 1,0 только в последних знаках, а `==` этого не прощает.) Верните как было.
4. Когда закончите: `kind delete cluster --name course-21-11`.

## Упражнения

### Упражнение 1 — Бюджет в запросах (легко)

Сервис получил 2 400 000 запросов за последние 30 дней, и у него SLO доступности 99,9%. 1800 запросов упали с 5xx. Сколько сбоев допускает бюджет, сколько от него осталось и что говорит политика бюджета ошибок из раздела 4?

<details>
<summary>Подсказка</summary>

Допустимые сбои = `(1 − SLO) × total`. Остаток = `1 − failed / allowed`. Проверить можно функцией `budget_left` из `p21-slo`.

</details>

<details>
<summary>Ответ</summary>

Допустимо: `0.001 × 2,400,000 = 2,400` неудачных запросов. Потрачено: `1,800 / 2,400 = 75%`, значит, осталось 25%. `budget_left(0.999, 1_800, 2_400_000)` возвращает `0.25000000000000067`, что при печати через `{:.2}` даёт `0.25`.

Почему это работает: бюджет — это количество допустимых плохих событий в окне. При остатке 25% команда ещё выпускает изменения как обычно, но это хороший момент посмотреть, что потратило остальные 75%, пока одно плохое развёртывание не опустошило остаток.

</details>

### Упражнение 2 — Тест, что здоровый сервис никого не будит (средне)

**Напишите тест `promtool`** `rules/healthy-test.yml` для тех же правил: при 99 хороших и 1 неудачном запросе в минуту (1% ошибок, скорость расходования 2) оповещение `OrdersErrorBudgetFastBurn` **не** должно срабатывать на 30-й минуте. Запустите его, затем замените неудачные запросы на 10 в минуту и покажите, что теперь тест падает.

<details>
<summary>Подсказка</summary>

Пустой список `exp_alerts: []` означает «в момент `eval_time` не срабатывает ни одно оповещение с этим именем».

</details>

<details>
<summary>Ответ</summary>

```yaml
rule_files:
  - slo-rules.yml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      # 99 good and 1 failed request per minute: 1% errors, burn rate 2.
      - series: 'http_requests_total{path="/orders/{id}", status="200"}'
        values: '0+99x60'
      - series: 'http_requests_total{path="/orders/{id}", status="500"}'
        values: '0+1x60'
    alert_rule_test:
      - eval_time: 30m
        alertname: OrdersErrorBudgetFastBurn
        exp_alerts: []
```

```text
Processing test file: healthy-test.yml
  SUCCESS
```

С `'0+10x60'` для неудачных запросов:

```text
Processing test file: healthy-test.yml
  FAILED:
    alertname: OrdersErrorBudgetFastBurn, time: 30m, 
        exp:[], 
        got:[
            0:
              Labels:{alertname="OrdersErrorBudgetFastBurn", severity="page"}
              Annotations:{description="9.174% of order requests fail (SLO allows 0.5%).", summary="Orders are burning the error budget 14x too fast"}
            ]
```

Почему это работает: при 1% ошибок отношение (0,01) ниже `14.4 × 0.005 = 0.072`, поэтому оповещение должно молчать; при 10 неудачных на 99 хороших оно около 9,2%, выше порога. Пара таких тестов — «будит, когда должно» и «молчит, когда должно» — защищает оповещение от будущих правок.

</details>

### Упражнение 3 — SLO задержки (сложно)

Добавьте второй SLI: долю запросов заказов, выполненных не дольше чем за 100 мс, за 5 минут, в виде правила записи `slo:orders_latency_good:ratio_rate5m`. Используйте гистограмму из Урока 21.9. Какое у него значение в кластере и почему? Затем запишите SLO одним предложением.

<details>
<summary>Подсказка</summary>

Корзина `le="0.1"` считает запросы, занявшие не больше 0,1 с; `_count` считает все. Разделите их скорости.

</details>

<details>
<summary>Ответ</summary>

Добавьте в группу `orders-slo` (в оба файла):

```yaml
      - record: slo:orders_latency_good:ratio_rate5m
        expr: |
          sum(rate(http_request_duration_seconds_bucket{path="/orders/{id}", le="0.1"}[5m]))
            / sum(rate(http_request_duration_seconds_count{path="/orders/{id}"}[5m]))
```

В UI Prometheus то же выражение даёт:

```text
[{},"1"]
```

Все запросы заказов занимают около 31 мс (Урок 21.9), намного меньше 100 мс, поэтому доля хороших запросов равна 1, то есть 100%.

SLO: «99% запросов заказов выполняются не дольше чем за 100 мс, измерено за 30 дней». Почему это работает: граница 0,1 есть в нашей гистограмме, поэтому отношение точное, а не оценка, как у `histogram_quantile`. Выбирайте пороги задержки, совпадающие с границами корзин.

</details>

## Вопросы для самопроверки

1. Чем отличаются SLI, SLO и SLA?

<details>
<summary>Ответ</summary>

SLI — измерение (доля хороших запросов). SLO — цель команды для него за окно (99,5% за 30 дней). SLA — договор с клиентами с последствиями; он мягче SLO.

</details>

2. Почему ответ 404 обычно — *хорошее* событие для SLI доступности?

<details>
<summary>Ответ</summary>

Сервис сработал правильно: сообщил клиенту, что такой вещи нет. 4xx — ошибки клиента; только 5xx означают, что сломался сервис.

</details>

3. Сколько полного простоя допускает SLO 99,9% за 30 дней?

<details>
<summary>Ответ</summary>

`0.001 × 30 × 24 × 60 = 43.2` минуты.

</details>

4. Доля ошибок — 2%, SLO — 99%. Какова скорость расходования и через сколько закончится 30-дневный бюджет?

<details>
<summary>Ответ</summary>

`0.02 / 0.01 = 2`. Бюджета хватит на `30 / 2 = 15` дней.

</details>

5. Зачем оповещению о быстром расходовании два окна (1 час и 5 минут)?

<details>
<summary>Ответ</summary>

Длинное окно проверяет, что проблема реальна и достаточно велика; короткое — что она всё ещё происходит, поэтому оповещение прекращается вскоре после исправления.

</details>

6. Что делает `for: 2m` в правиле оповещения?

<details>
<summary>Ответ</summary>

Условие должно оставаться истинным 2 минуты, прежде чем оповещение сработает; до этого оно в состоянии pending. Единичные плохие оценки никого не будят.

</details>

## Итоги

- SRE задаёт цели надёжности в числах. 100% — неправильная цель: недостижимая, незаметная для пользователей и останавливающая любые изменения.
- SLI — «хорошие события / допустимые события» с точки зрения пользователя; 5xx — плохие, 4xx обычно хорошие; проверки здоровья — не пользователи.
- SLO — цель для SLI за скользящее окно; SLA — более мягкое обещание в договоре.
- Бюджет ошибок равен `1 − SLO`; согласованная политика говорит, что происходит, когда он потрачен.
- Поднимайте тревогу по скорости расходования с длинным и коротким окном (14,4 за 1 час и 5 минут — вызов дежурного), а не по единичным ошибкам.
- Записывайте SLI правилами записи, а оповещение — правилом оповещения, проверяйте их `promtool check rules` и тестируйте `promtool test rules`.

## Источники и что почитать

- [Site Reliability Engineering — Embracing Risk](https://sre.google/sre-book/embracing-risk/) и [Service Level Objectives](https://sre.google/sre-book/service-level-objectives/) — исходные главы о бюджете ошибок, SLI и SLO.
- [The Site Reliability Workbook — Implementing SLOs](https://sre.google/workbook/implementing-slos/) и [Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) — практическое проектирование SLO и оповещения по скорости расходования с несколькими окнами, использованные здесь.
- [The Site Reliability Workbook — Example error budget policy](https://sre.google/workbook/error-budget-policy/) — полный пример политики, который можно адаптировать.
- [Prometheus — Recording rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/), [Alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/) и [Rule naming](https://prometheus.io/docs/practices/rules/) — синтаксис правил и соглашение `level:metric:operations`.
- [Prometheus — Unit testing for rules](https://prometheus.io/docs/prometheus/latest/configuration/unit_testing_rules/) — полный формат файлов `promtool test rules`.

---

[🏠 Начало](../START-HERE.md) · [← 21.10 Логи и трассировки в кластере](10-logs-and-traces-in-a-cluster.md) · [📋 Этап 21](README.md) · [21.12 Инциденты, дежурства и постмортемы →](12-incidents-on-call-and-postmortems.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
