<!-- nav:top -->
[🏠 Начало](../START-HERE.md) · [← 21.11 Основы SRE: SLI, SLO и бюджет ошибок](11-sre-basics-slis-slos-and-error-budgets.md) · [📋 Этап 21](README.md) · [21.13 Проект: ваш сервис в Kubernetes →](13-project-run-your-service-on-kubernetes.md) · [📖 Глоссарий](../GLOSSARY.md) · [🇬🇧 English](../../en/21-cloud-native-infrastructure/12-incidents-on-call-and-postmortems.md) <!-- nav -->

# Урок 21.12 — Инциденты, дежурства и постмортемы

> **Этап 21 — Облачная (cloud-native) инфраструктура** · Урок 12 из 14 · ⏱ около 105 минут

## Что вы узнаете

- Что такое инцидент и как уровни серьёзности определяют, насколько громкой будет реакция.
- Как устроены дежурства: ротации, вызовы, ранбуки и что заслуживает того, чтобы кого-то разбудить.
- Шаги и роли реагирования на инцидент и почему сначала *смягчают последствия*, а причину ищут потом.
- Как отработать настоящий инцидент в кластере kind: неудачный выпуск, диагностика через `kubectl` и откат.
- Как написать постмортем без поиска виноватых, с хронологией и задачами по итогам.

## Перед началом

- Вы прошли [Урок 21.11 — Основы SRE: SLI, SLO и бюджет ошибок](11-sre-basics-slis-slos-and-error-budgets.md). Оповещение о скорости расходования бюджета из того урока, оповещение по SLO (Service Level Objective, цель уровня обслуживания), — это «вызов», с которого здесь начинается инцидент.
- Вы знаете Deployment, выкатки и `kubectl rollout` из [Урока 21.4](04-deploying-a-rust-service-to-kubernetes.md) и умеете читать логи и события по [Уроку 21.10](10-logs-and-traces-in-a-cluster.md).
- У вас есть образ `course-p21-orders:0.1.0` из Урока 21.9.

## Новые слова

| Слово | Значение |
|---|---|
| **инцидент** (incident) | Незапланированное событие, которое вредит пользователям или ставит сервис под угрозу и требует согласованной реакции прямо сейчас. |
| **серьёзность** (severity: SEV1, SEV2, SEV3) | Насколько плох инцидент. SEV — сокращение от *severity*, а число — уровень: SEV1 самый серьёзный. Определяет, кого вызывают, как быстро и как часто люди получают новости. |
| **дежурство** (on-call) | Обязанность быть на связи и готовым отвечать на вызовы в течение смены. |
| **ротация** (rotation) | Расписание, которое передаёт дежурство от человека к человеку, например по неделе каждому. |
| **ранбук** (runbook) | Короткий документ для одного оповещения: что оно значит, как проверить, как смягчить последствия, кого звать. |
| **смягчение последствий** (mitigation) | Действие, которое быстро останавливает ущерб (откат, переключение трафика, добавление мощности), даже до того, как понятна причина. |
| **IC** (Incident Commander, руководитель инцидента) | Человек, который ведёт реагирование: принимает решения, раздаёт задачи и держит общую картину. Сам ничего не чинит. |
| **постмортем** (postmortem) | Письменный разбор после инцидента: что произошло, почему, чему научились и что изменится. |
| **без поиска виноватых** (blameless) | Правило постмортема: искать причины в системах и процессах, никогда — виноватого человека. |
| **задача по итогам** (action item) | Конкретная задача из постмортема с ответственным и сроком. |
| **MTTR** (Mean Time To Recovery, среднее время восстановления) | Среднее время от начала инцидента до восстановления. Быстрое смягчение последствий держит его низким. |

## 1. Что такое инцидент

Не каждая проблема — инцидент. Медленный тест в CI (Continuous Integration, непрерывная интеграция) — проблема; инцидент — то, что вредит пользователям **сейчас** (или вот-вот начнёт) и требует, чтобы люди действовали согласованно. Команды объявляют инцидент быстро и отменяют, если он оказался мелким: объявить стоит мало, опоздать — дорого.

**Серьёзность** определяет громкость реакции. Каждая компания пишет свою таблицу; типичная выглядит так:

| Серьёзность | Пример | Реакция |
|---|---|---|
| **SEV1** критический | Данные потеряны, или большинство пользователей не может пользоваться продуктом | Вызов немедленно, подключаются все нужные люди, новости каждые 30 минут, постмортем обязателен |
| **SEV2** серьёзный | Возможность сломана для части пользователей; бюджет ошибок быстро тратится | Вызов дежурного, новости каждый час, постмортем обязателен |
| **SEV3** незначительный | Небольшая или частичная проблема, есть обходной путь | Задача, исправление в рабочее время, постмортем по желанию |

Серьёзность может меняться во время инцидента. Повышайте её, как только узнаёте, что всё хуже; понижайте, когда ущерб ограничен.

## 2. Дежурство

Кто-то должен ответить, когда оповещение из Урока 21.11 срабатывает в три часа ночи. Этот человек — **дежурный**. Команда делит обязанность в **ротации**: например, по неделе каждому, *основной* дежурный получает вызовы, *запасной* подстраховывает. В конце смены — короткая **передача** (handover): что произошло, что ещё открыто.

Что делает дежурство терпимым, а сервис — безопасным:

- **Вызывайте только по симптомам, которые вредят пользователям** и требуют человека сейчас: оповещения о скорости расходования SLO, а не «CPU (Central Processing Unit, центральный процессор) на 85%». Оповещение о причине, которую пользователи не чувствуют, превращается в задачу.
- **У каждого вызова должен быть ранбук.** Спросонья никто не должен придумывать первые шаги.
- **Мало вызовов.** SRE (Site Reliability Engineering) Workbook советует не больше пары инцидентов за смену; если их больше, чините оповещения или сервис. Люди, получающие много ложных вызовов, перестают реагировать: *усталость от оповещений* (alert fatigue).
- **Время на восстановление.** После ночи с вызовами — отдых; дежурство — это работа, она оплачивается или компенсируется.

Ранбук для нашего оповещения, лежит в Git рядом с правилами (`runbooks/orders-error-budget-fast-burn.md`):

```text
# OrdersErrorBudgetFastBurn

Meaning: more than 7.2% of order requests fail (5xx) for the last hour and the
last 5 minutes. At this speed the 30-day error budget is gone in about 2 days.

1. Check: Grafana dashboard "Orders" - which status codes, since when, which pods?
2. What changed? kubectl -n shop rollout history deployment/orders
   A release in the last hour is the most likely cause -> roll back first:
   kubectl -n shop rollout undo deployment/orders
3. Not a release? Look at the logs and traces of failing requests (Jaeger, error=true).
4. Users still affected after 15 minutes? Declare SEV2 and call the secondary.
Owner: orders team. Escalation: database team for errors from the database.
```

## 3. Реагирование на инцидент

У инцидента есть жизненный цикл:

```text
 detect ──► triage ──► mitigate ──► resolve ──► follow up
 (alert,    (declare,   (stop the     (cause      (postmortem,
  user       severity,   damage:       fixed,      action items)
  report)    roles)      rollback)     all clear)
```

**Сначала смягчите последствия.** Пользователям неважно, *почему* сломалось; им важно, чтобы снова заработало. Самое быстрое безопасное действие часто — отменить последнее изменение: откатить выпуск, выключить флаг возможности (feature flag), увести трафик в здоровый регион. Понимание причины — после того, как пользователи в безопасности. Команда, которая час отлаживает сломанный выпуск вместо отката за минуту, получает высокий MTTR (Mean Time To Recovery).

**Роли.** В маленьком инциденте всё делает один человек. Как только подключаются другие, разделите роли:

| Роль | Делает | Не делает |
|---|---|---|
| **IC** (Incident Commander) | Руководит: держит общую картину, решает, раздаёт задачи, объявляет окончание | Не набирает команды |
| **Эксплуатация** (Operations) | Исследует и меняет системы, вслух говорит о каждом действии перед тем, как его сделать | Не меняет то, о чём никто не просил |
| **Коммуникации** (Communications) | Пишет регулярные новости для пользователей, поддержки и руководства | Не гадает о причинах |
| **Секретарь** (Scribe, часто сам IC) | Ведёт хронологию: время, наблюдения, решения | Не правит хронологию потом, чтобы выглядеть лучше |

Работа идёт в одном месте, например в отдельном чат-канале на каждый инцидент, и хронология пишется сама. Новости следуют одной форме: *что сломано, какое влияние, что мы делаем, когда следующая новость*.

> 💡 **Совет:** «скажи, прежде чем сделать» звучит медленно, но защищает от худшей ошибки во время инцидента: двое одновременно меняют одну систему, и каждый делает изменение другого невидимым.

## 4. Учебный инцидент

Отрабатывайте в спокойное время то, что понадобится в напряжённое. Создайте кластер и разверните здоровый сервис (`k8s/orders.yaml` из Урока 21.9 без блока `env`, чтобы ничего не падало):

```bash
kind create cluster --name course-21-12
kind load docker-image course-p21-orders:0.1.0 --name course-21-12
kubectl apply -f k8s/orders.yaml
kubectl -n shop rollout status deployment/orders
kubectl -n shop annotate deployment/orders kubernetes.io/change-cause="release 0.1.0"
```

Аннотация `kubernetes.io/change-cause` — заметка для людей; `kubectl rollout history` показывает её для каждой ревизии.

**Дымовой тест** (smoke test) — крошечный тест, который проверяет только «работает ли в принципе?». Наш отправляет 100 запросов заказов изнутри кластера и считает сбои. Сохраните его в переменной оболочки, чтобы использовать повторно:

```bash
SMOKE='fail=0; for i in $(seq 1 100); do wget -q -O /dev/null http://orders/orders/$i 2>/dev/null || fail=$((fail+1)); done; echo "failed: $fail of 100"'
kubectl -n shop run smoke --rm -i --restart=Never --image=busybox:1.37 -- sh -c "$SMOKE"
```

```text
All commands and output from this session will be recorded in container logs, including credentials and sensitive information passed through the command prompt.
If you don't see a command prompt, try pressing enter.
failed: 0 of 100
pod "smoke" deleted from shop namespace
```

(Первые две строки — стандартное уведомление `kubectl run -i`; ниже мы их опускаем.) Теперь коллега выпускает «небольшое изменение конфигурации». В жизни это была бы неверная переменная окружения или флаг; мы используем `FAIL_EVERY=2`:

```bash
kubectl -n shop set env deployment/orders FAIL_EVERY=2
kubectl -n shop annotate deployment/orders kubernetes.io/change-cause="config: FAIL_EVERY=2"
kubectl -n shop rollout status deployment/orders
```

```text
deployment.apps/orders env updated
deployment.apps/orders annotated
Waiting for deployment "orders" rollout to finish: 1 out of 2 new replicas have been updated...
...
deployment "orders" successfully rolled out
```

Выкатка **прошла успешно**: новые поды отвечают на `/healthz`, поэтому проба готовности довольна. Через несколько минут вас вызвало бы оповещение о скорости расходования из Урока 21.11. Здесь роль оповещения играет дымовой тест:

```bash
kubectl -n shop run smoke --rm -i --restart=Never --image=busybox:1.37 -- sh -c "$SMOKE"
```

```text
failed: 48 of 100
```

**Сортировка** (triage). Падает половина запросов: пользователям явно плохо. Объявите инцидент; один сервис, часть запросов: SEV2. Теперь вопросы в порядке ранбука. Сначала: здоровы ли поды?

```bash
kubectl -n shop get pods
```

```text
NAME                      READY   STATUS    RESTARTS   AGE
orders-5766c88465-6tjrj   1/1     Running   0          5s
orders-5766c88465-thsl9   1/1     Running   0          6s
```

(Старые поды, которые ещё были в состоянии `Terminating`, не показаны.) Всё выглядит зелёным. Это классика: **пробы проверяют, что процесс жив, а не что он работает для пользователей.** Поэтому вызов приходит от SLO, а не от статуса подов. Второй вопрос, самый полезный в любом инциденте: **что изменилось?**

```bash
kubectl -n shop rollout history deployment/orders
```

```text
deployment.apps/orders 
REVISION  CHANGE-CAUSE
1         release 0.1.0
2         config: FAIL_EVERY=2
```

Изменение за минуты до ошибок — главный подозреваемый. (Для подтверждения `kubectl -n shop port-forward deployment/orders 3000:3000` и `curl -s localhost:3000/metrics | grep '^http_requests_total'` покажут, что в новых подах `status="500"` растёт так же быстро, как `status="200"`.) **Смягчите последствия**, отменив изменение, не дожидаясь полного анализа:

```bash
kubectl -n shop rollout undo deployment/orders
kubectl -n shop rollout status deployment/orders
kubectl -n shop run smoke --rm -i --restart=Never --image=busybox:1.37 -- sh -c "$SMOKE"
```

```text
Warning: resource deployments/orders was previously managed with 'kubectl apply'. Rolling back will not update the kubectl.kubernetes.io/last-applied-configuration annotation, which may cause unexpected behavior on future 'kubectl apply' operations. Consider using 'kubectl apply' with your previous configuration file instead.
deployment.apps/orders rolled back
...
deployment "orders" successfully rolled out
failed: 2 of 100
```

Два сбоя сразу после выкатки: несколько запросов ещё попали в старые плохие поды, пока те завершались. Запустите дымовой тест ещё раз через несколько секунд; он печатает `failed: 0 of 100`. Пользователи снова в безопасности. Ещё раз посмотрите на историю:

```bash
kubectl -n shop rollout history deployment/orders
```

```text
deployment.apps/orders 
REVISION  CHANGE-CAUSE
2         config: FAIL_EVERY=2
3         release 0.1.0
```

**Что происходит:**

1. `rollout undo` создаёт **новую** ревизию (3) с шаблоном предыдущей (1). Ревизия 1 пропадает из списка, потому что теперь она стала ревизией 3.
2. Предупреждение важно. Откат изменил кластер, но **не источник истины**: ваши файлы в Git (или значения Helm, или конфигурацию OpenTofu из Урока 21.8). Если их никто не исправит, следующий `kubectl apply` или запуск конвейера вернёт плохую конфигурацию. Это дрейф, созданный намеренно. Исправление источника — часть *устранения* инцидента.
3. Для пользователей инцидент закончился, но не для команды: дальше — постмортем.

> 🔒 **Безопасность:** во время инцидента хочется действовать в обход правил: выдать всем права администратора, вставить секреты в чат, «на минутку» выключить проверки безопасности. Записывайте каждое такое экстренное изменение в хронологию и отменяйте его потом; злоумышленники любят моменты неразберихи.

## 5. Постмортем без поиска виноватых

**Постмортем** превращает плохой день в долгосрочное улучшение. Его пишут в течение нескольких дней участники событий, а читает вся команда. Главное правило: он **без поиска виноватых** (blameless). «Коллега ввёл неверное значение» — не причина; люди ошибаются, и вопрос в том, почему система позволила одной ошибке навредить пользователям. Когда людей обвиняют, они скрывают информацию, и следующий инцидент получается хуже.

Постмортем нашего учебного инцидента (время в UTC, Coordinated Universal Time, всемирное координированное время, из нашего запуска):

```text
# Postmortem: half of order requests failed after a config change

Severity: SEV2        Status: action items in progress
Duration: 09:02-09:03 UTC (about 1 minute of impact in the practice cluster)
Authors: the on-call engineer and the author of the change

## Summary
A configuration change (FAIL_EVERY=2) made every second order request fail with
500. The release passed its readiness probes. A smoke test detected the errors;
a rollback restored service.

## Impact
48 of 100 test requests failed. No data was lost.

## Timeline (UTC)
09:02  Config change rolled out; rollout reports success.
09:03  Smoke test: 48 of 100 requests fail. Incident declared, SEV2.
09:03  `rollout history` shows the change minutes before the errors.
09:03  Rollback with `kubectl rollout undo`; smoke test: 0 of 100 fail. Mitigated.
09:04  Noticed: the bad value is still in Git. Fix merged later the same day. Resolved.

## Contributing factors
- Probes only check /healthz, which does not exercise the order path.
- The change was applied directly to the cluster, without review or a smoke test.
- No automatic check stopped the rollout when errors rose.

## What went well
- Rollout history with change-cause made the suspect obvious in one minute.
- Rollback first, analysis second: users were safe within a minute.

## Action items
| Action                                               | Owner       | Due        |
| Run the smoke test in the deploy pipeline            | orders team | 2026-11-25 |
| Config changes only through Git and review           | orders team | 2026-11-25 |
| Automatic rollback when the smoke test fails         | platform    | 2026-12-09 |
```

**Способствующие факторы** (contributing factors) получаются из нескольких «почему?». Почему пользователи видели ошибки? Потому что конфигурация заставила сервис падать. Почему конфигурация попала в продакшен? Потому что прошла без проверки. Почему её ничто не остановило? Потому что пробы не видят путь заказа. Каждое «почему» указывает на систему, которую можно улучшить, — для этого и нужны **задачи по итогам**. Задача без ответственного и срока — пожелание, а не план.

> 📏 **Соглашение:** постмортемы широко распространяют и хранят там, где их может найти каждый. Их ценность в том, что на них учатся другие, в том числе другие команды.

## Частые ошибки

> ⚠️ **Частая ошибка:** отлаживать до смягчения последствий. Причина интересна, но каждая минута отладки — минута сбоев у пользователей. Если вероятная причина — недавнее изменение, сначала откатитесь.

> ⚠️ **Частая ошибка:** доверять зелёным подам. `Running` и `1/1 READY` означают, что проба проходит. Они ничего не говорят о том, получают ли пользователи правильные ответы. Измеряйте то, что получают пользователи (SLO, дымовой тест).

> ⚠️ **Частая ошибка:** забыть об источнике истины после отката. Кластер исправлен, Git — нет; следующее развёртывание повторит инцидент. Отмените изменение в Git в рамках устранения.

> ⚠️ **Частая ошибка:** постмортем, который называет виновника, или задачи вроде «быть внимательнее». Ни то, ни другое не меняет систему. Пишите действия, которые делают ошибку невозможной или безвредной.

## Попробуйте сами

1. Повторите учебный инцидент, но на этот раз проговаривайте роли вслух с другом (или в одиночку): IC говорит «объявляем SEV2, проверь, что изменилось», Эксплуатация говорит «откатываю ревизию 2», прежде чем это набрать.
2. Напишите новость, которую вы отправили бы в поддержку после неудачного дымового теста, в четырёх предложениях: что сломано, влияние, что мы делаем, когда следующая новость.
3. После отката выполните `kubectl apply -f k8s/orders.yaml` и `kubectl -n shop rollout history deployment/orders`. В файле нет `FAIL_EVERY`, поэтому ничего не ломается; подумайте, что было бы, если бы плохое значение было в файле.
4. Когда закончите: `kind delete cluster --name course-21-12`.

## Упражнения

### Упражнение 1 — Выберите серьёзность (легко)

По таблице из раздела 1 выберите серьёзность для каждого случая и скажите, кого вызывают: (а) поиск в магазине медленный для 3% пользователей, обходной путь — просматривать категории; (б) ошибка удалила историю заказов всех клиентов одной страны; (в) в 02:00 срабатывает оповещение о скорости расходования бюджета ошибок сервиса заказов, падает около 10% запросов заказов.

<details>
<summary>Подсказка</summary>

Задайте два вопроса: потеряны ли данные? Сколько пользователей затронуто и могут ли они обойти проблему?

</details>

<details>
<summary>Ответ</summary>

(а) SEV3: пользователей мало, обходной путь есть; задача на рабочее время. (б) SEV1: потеря данных, сколько бы ни было пользователей; вызов немедленно и подключение нужных людей, включая команду баз данных. (в) SEV2: падает часть запросов, и бюджет быстро тратится; вызов уходит дежурному, который может поднять его до SEV1, если станет хуже.

Почему это работает: серьёзность измеряет вред пользователям и данным, а не то, насколько интересна или сложна техническая проблема.

</details>

### Упражнение 2 — Серьёзность в коде, с тестами (средне)

Напишите небольшую программу `code/p21-severity` с перечислением `Severity { Sev1, Sev2, Sev3 }` и функцией `severity(affected_users: f64, data_lost: bool) -> Severity`: потеря данных или не меньше 50% пользователей — SEV1, не меньше 5% — SEV2, всё остальное — SEV3. **Напишите тесты** для потери данных, для 50% и для обеих сторон границы 5%. `main` печатает четыре случая.

<details>
<summary>Подсказка</summary>

`#[derive(Debug, PartialEq)]` позволяет печатать перечисление через `{:?}` и сравнивать его в `assert_eq!`. Проверяйте границы значениями ровно на пределе и чуть ниже.

</details>

<details>
<summary>Ответ</summary>

```rust
#[derive(Debug, PartialEq)]
enum Severity {
    Sev1, // critical: page now, all hands
    Sev2, // major: page the person on call
    Sev3, // minor: fix in working hours
}

/// Chooses a severity from the share of affected users (0.0 to 1.0) and data loss.
fn severity(affected_users: f64, data_lost: bool) -> Severity {
    if data_lost || affected_users >= 0.5 {
        Severity::Sev1
    } else if affected_users >= 0.05 {
        Severity::Sev2
    } else {
        Severity::Sev3
    }
}

fn main() {
    let cases = [(0.5, false), (0.10, false), (0.01, false), (0.01, true)];
    for (affected, lost) in cases {
        println!(
            "{:>4.0}% of users, data lost: {:<5} -> {:?}",
            affected * 100.0,
            lost,
            severity(affected, lost)
        );
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn data_loss_is_always_sev1() {
        assert_eq!(severity(0.0, true), Severity::Sev1);
    }

    #[test]
    fn half_of_the_users_is_sev1() {
        assert_eq!(severity(0.5, false), Severity::Sev1);
    }

    #[test]
    fn borders_between_sev2_and_sev3() {
        assert_eq!(severity(0.05, false), Severity::Sev2);
        assert_eq!(severity(0.0499, false), Severity::Sev3);
    }
}
```

```text
  50% of users, data lost: false -> Sev1
  10% of users, data lost: false -> Sev2
   1% of users, data lost: false -> Sev3
   1% of users, data lost: true  -> Sev1
```

```text
running 3 tests
test tests::data_loss_is_always_sev1 ... ok
test tests::borders_between_sev2_and_sev3 ... ok
test tests::half_of_the_users_is_sev1 ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Почему это работает: `if` сначала проверяет самое серьёзное условие, поэтому потеря данных побеждает даже при малом числе пользователей. Тесты границ ловят самую частую ошибку таких правил — `>` вместо `>=`. Правило команды, записанное кодом и тестами, нельзя «вспомнить по-другому» в три часа ночи.

</details>

### Упражнение 3 — Застрявшая выкатка (сложно)

В учебном кластере выпустите образ, которого не существует: `kubectl -n shop set image deployment/orders orders=course-p21-orders:9.9.9`, затем `kubectl -n shop rollout status deployment/orders --timeout=45s`. Выясните, что не так, только с помощью `kubectl`, запустите дымовой тест и объясните, почему пользователи **не** пострадали. Затем смягчите последствия.

<details>
<summary>Подсказка</summary>

`kubectl get pods` показывает состояние нового пода; `kubectl describe pod <name>` показывает в конце его события. Подумайте, как постепенное обновление (rolling update) заменяет поды.

</details>

<details>
<summary>Ответ</summary>

```text
deployment.apps/orders image updated
Waiting for deployment "orders" rollout to finish: 1 out of 2 new replicas have been updated...
error: timed out waiting for the condition
```

```text
NAME                      READY   STATUS             RESTARTS   AGE
orders-7878c5fd55-k69q8   0/1     ImagePullBackOff   0          45s
orders-f6597ddff-6zxg2    1/1     Running            0          50s
orders-f6597ddff-t52jf    1/1     Running            0          51s
```

События нового пода (сокращённо):

```text
  Normal   Pulling    4s (x3 over 45s)   kubelet            spec.containers{orders}: Pulling image "course-p21-orders:9.9.9"
  Warning  Failed     3s (x3 over 44s)   kubelet            spec.containers{orders}: Failed to pull image "course-p21-orders:9.9.9": failed to pull and unpack image "docker.io/library/course-p21-orders:9.9.9": failed to resolve reference "docker.io/library/course-p21-orders:9.9.9": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
  Warning  Failed     3s (x3 over 44s)   kubelet            spec.containers{orders}: Error: ErrImagePull
```

Дымовой тест печатает `failed: 0 of 100`. Смягчение: `kubectl -n shop rollout undo deployment/orders` печатает `deployment.apps/orders rolled back` (и то же предупреждение, что выше), и сломанный под удаляется.

Почему это работает: образа нет на узле (мы никогда не загружали `9.9.9`), поэтому kubelet пробует Docker Hub и получает отказ: `ErrImagePull`, затем `ImagePullBackOff` (ожидание между попытками растёт). Постепенное обновление сначала запускает новый под и удаляет старый, только когда новый **готов** (для 2 реплик: не больше 1 лишнего пода и ни одного недоступного). Новый под так и не становится готовым, поэтому оба старых продолжают работать. Выкатка застряла, но сервис не сломан: максимум SEV3, но исправить нужно, потому что следующий настоящий выпуск заблокирован.

</details>

## Вопросы для самопроверки

1. Почему инцидент нужно объявлять рано, даже если он может оказаться мелким?

<details>
<summary>Ответ</summary>

Объявить стоит мало, и объявление можно отменить; позднее объявление означает потерянное время без координации, ролей и коммуникации, пока пользователи страдают.

</details>

2. Что должно вызывать дежурного: «CPU на 85%» или «бюджет ошибок тратится в 14 раз быстрее»? Почему?

<details>
<summary>Ответ</summary>

Оповещение о скорости расходования: это симптом, который чувствуют пользователи, и он требует действий сейчас. Высокая загрузка CPU (Central Processing Unit) может быть безвредной; её место на дашборде или в задаче.

</details>

3. Что на практике означает «сначала смягчите последствия»?

<details>
<summary>Ответ</summary>

Остановить ущерб самым быстрым безопасным действием, часто отменой последнего изменения (откат, выключение флага), до того как точно понятна причина.

</details>

4. Что делает руководитель инцидента и чего он *не* делает?

<details>
<summary>Ответ</summary>

IC руководит: держит общую картину, принимает решения, раздаёт задачи, организует коммуникацию и объявляет окончание. IC не набирает команды, чтобы кто-то всегда видел всю картину.

</details>

5. Почему после `kubectl rollout undo` инцидент ещё не устранён?

<details>
<summary>Ответ</summary>

Кластер исправлен, но источник истины (Git, значения Helm, файлы IaC) всё ещё содержит плохое изменение. Следующее развёртывание вернёт его, поэтому изменение нужно отменить и там.

</details>

6. Что делает постмортем «без поиска виноватых» и почему это важно?

<details>
<summary>Ответ</summary>

Он ищет причины в системах и процессах, а не в людях. Когда людей не обвиняют, они рассказывают, что на самом деле произошло, а задачи чинят систему так, чтобы та же ошибка больше не навредила пользователям.

</details>

## Итоги

- Инцидент вредит пользователям сейчас и требует координации; серьёзность (SEV1–SEV3) задаёт громкость реакции. Объявляйте рано.
- Дежурство делится в ротации; вызывайте только по симптомам, которые чувствуют пользователи, например по расходованию SLO, и давайте каждому вызову ранбук.
- Жизненный цикл: обнаружение → сортировка → смягчение → устранение → работа по итогам. Сначала смягчайте последствия, часто откатом.
- Разделяйте роли, как только подключаются другие люди: руководитель инцидента, эксплуатация, коммуникации, секретарь.
- Зелёные поды не означают довольных пользователей; первым делом спрашивайте «что изменилось?» и после отката исправляйте источник истины.
- Постмортем без поиска виноватых содержит хронологию, способствующие факторы и задачи с ответственными и сроками.

## Источники и что почитать

- [Site Reliability Engineering — Managing Incidents](https://sre.google/sre-book/managing-incidents/) и [Being On-Call](https://sre.google/sre-book/being-on-call/) — роли в инциденте и здоровая практика дежурств.
- [The Site Reliability Workbook — Incident Response](https://sre.google/workbook/incident-response/) и [On-Call](https://sre.google/workbook/on-call/) — практические примеры того и другого.
- [Site Reliability Engineering — Postmortem Culture](https://sre.google/sre-book/postmortem-culture/) и [глава Workbook о постмортемах](https://sre.google/workbook/postmortem-culture/) — постмортемы без поиска виноватых, с хорошими и плохими примерами.
- [Документация PagerDuty Incident Response](https://response.pagerduty.com/) — открыто опубликованный процесс реагирования с описанием ролей и чек-листами.
- [Kubernetes — Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) и [Debug running pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/) — постепенные обновления, `rollout undo` и как исследовать поды и события.

---

[🏠 Начало](../START-HERE.md) · [← 21.11 Основы SRE: SLI, SLO и бюджет ошибок](11-sre-basics-slis-slos-and-error-budgets.md) · [📋 Этап 21](README.md) · [21.13 Проект: ваш сервис в Kubernetes →](13-project-run-your-service-on-kubernetes.md) · [📖 Глоссарий](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
