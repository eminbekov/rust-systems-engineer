<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 21.10 Logs and traces in a cluster](10-logs-and-traces-in-a-cluster.md) · [📋 Phase 21](README.md) · [21.12 Incidents, on-call and postmortems →](12-incidents-on-call-and-postmortems.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/21-cloud-native-infrastructure/11-sre-basics-slis-slos-and-error-budgets.md) <!-- nav -->

# Lesson 21.11 — SRE basics: SLIs, SLOs and error budgets

> **Phase 21 — Cloud-native infrastructure** · Lesson 11 of 14 · ⏱ about 120 minutes

## What you will learn

- What SRE (Site Reliability Engineering) is, and why "100% reliable" is the wrong goal.
- How to measure reliability from the user's side with an SLI (Service Level Indicator), and how to set a goal for it with an SLO (Service Level Objective).
- What an error budget is, how to compute it, and how a team uses it to decide between new features and reliability work.
- What a burn rate is, and how to alert on it instead of on every error.
- How to write the SLO as Prometheus recording and alerting rules, test them with `promtool`, and see the alert fire in a kind cluster.

## Before you start

- You finished [Lesson 21.10 — Logs and traces in a cluster](10-logs-and-traces-in-a-cluster.md).
- You know the request counter `http_requests_total` and the error-ratio query from [Lesson 21.9](09-metrics-with-prometheus-and-grafana.md); this lesson uses the same service, image `course-p21-orders:0.1.0`, and the same Prometheus chart.
- You read [Lesson 20.1 — What reliability means](../20-reliability-and-hardening/01-what-reliability-means.md) and the production-readiness checklist of [Lesson 20.9](../20-reliability-and-hardening/09-production-readiness-checklist.md). This lesson turns "the service is reliable enough" into a number you can check.
- Docker Desktop is running; we run `promtool` from the official `prom/prometheus` image, so you do not need to install it.

## New words

| Word | Meaning |
|---|---|
| **SRE** (Site Reliability Engineering) | A way of running services, started at Google, that treats reliability as an engineering problem with measured goals, automation and shared rules between developers and operators. |
| **SLI** (Service Level Indicator) | A measured number that says how well the service works for users, usually "good events / all valid events", for example the share of successful requests. |
| **SLO** (Service Level Objective) | A goal for an SLI over a time window: "99.5% of order requests succeed, measured over 30 days". |
| **SLA** (Service Level Agreement) | A promise in a contract with customers, with consequences (money back) if it is broken. Always looser than the internal SLO. |
| **error budget** | The amount of failure the SLO allows: `1 − SLO`. For 99.5% it is 0.5% of all requests in the window. |
| **burn rate** | How fast the error budget is being used. Burn rate 1 uses exactly the whole budget by the end of the window; burn rate 10 uses it ten times faster. |
| **recording rule** | A Prometheus rule that computes a query regularly and stores the result as a new time series with its own name. |
| **alerting rule** | A Prometheus rule with a condition; while the condition is true, Prometheus raises an alert. |
| **Alertmanager** | The Prometheus component that receives alerts and sends notifications (pager, chat, email). We keep it switched off and look at alerts in Prometheus. |
| **`promtool`** | The command-line tool of Prometheus that checks rule files and runs unit tests for them. |
| **page** | An urgent notification that wakes up the person on call (Lesson 21.12). |

## 1. Why not 100%?

"The service must never fail" sounds right, but it is a bad goal, for three reasons.

1. **It is impossible.** Networks drop packets, disks fail, deployments contain bugs. Lesson 20.1 showed that failure is normal; the question is only *how much*.
2. **Users cannot tell the difference.** A user on a phone network that works 99% of the time cannot see whether your service is 99.99% or 99.999% reliable. The last nines cost a lot and nobody notices them.
3. **It blocks all change.** Every deployment is a risk. A team that may never fail can never ship anything.

**SRE** (Site Reliability Engineering) turns this into numbers: pick a reliability level that keeps users happy, measure it, and use the difference to 100% as a *budget* for risk. That budget is spent on deployments, experiments and the occasional bad day.

## 2. SLI: measure what users feel

An **SLI** (Service Level Indicator) is a measurement from the user's point of view. The most useful form is a ratio:

```text
          good events
  SLI = ───────────────    (a number between 0 and 1, often shown as a percentage)
         valid events
```

| SLI kind | Good event | Valid event | From our metrics |
|---|---|---|---|
| availability | the request did not fail on our side (not 5xx) | every order request | `http_requests_total` by `status` |
| latency | the request finished in less than 100 ms | every order request | `http_request_duration_seconds_bucket{le="0.1"}` |

Decisions hidden in this table:

- **404 is good.** The user asked for an order that does not exist and got the correct answer. Status 4xx means "the client made a mistake", 5xx means "we failed". Only 5xx count against us.
- **Only real user traffic counts.** Health checks and `/metrics` are not users; we filter by `path="/orders/{id}"`.
- **Measure as close to the user as possible.** Our service measures itself, so a request that never reaches it (a broken Service or network) is invisible. Many teams measure at the load balancer or with an outside probe as well.

## 3. SLO and SLA

An **SLO** (Service Level Objective) is a goal for an SLI over a **window**:

> 99.5% of order requests succeed (not 5xx), measured over the last 30 days.

Each part matters. *99.5%* is the target. *Order requests* says which SLI. *30 days*, a **rolling window**, means every day you look back exactly 30 days, so a bad day counts for a month and then disappears.

An **SLA** (Service Level Agreement) is different: a contract with customers ("99% per month, or you get 10% of your money back"). The SLA is a business promise; the SLO is the team's internal goal. The SLO is always **stricter** than the SLA, so that the team notices problems and fixes them long before the contract is broken.

> 📏 **Convention:** Start with few SLOs (one availability and one latency SLO per user-facing service), and set the first target slightly *below* what the service does today. An SLO that is broken from day one teaches nobody anything. Tighten it later, when you know what users need.

## 4. The error budget

The **error budget** is what the SLO allows to fail: `1 − SLO`. A small Rust program makes the numbers concrete. Create it in `code/p21-slo` (`cargo new code/p21-slo`) with this `src/main.rs`:

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

**What happens:**

1. The table shows the famous "nines". Every extra nine divides the budget by ten: 99.9% ("three nines") allows only 43.2 minutes of full outage per 30 days, 99.99% about 4 minutes, less than one slow deployment.
2. With 1,000,000 requests, a 99.5% SLO allows 5,000 failures. 3,000 have happened, so 40% of the budget is left.
3. `close` compares floating-point numbers with a small tolerance, because `1.0 - 0.995` is not exactly `0.005` in binary floating point (Lesson 02.2).

> 🧪 **Testing:** Small pure functions such as `burn_rate` are easy to test, and SLO maths is easy to get wrong by a factor of 100 (percent versus fraction). Test the known points: three nines give 43.2 minutes; errors exactly at the budget give burn rate 1.

The budget becomes useful through an **error budget policy**, a short document the team agrees on in advance, for example:

- Budget left: ship features normally; planned risky work (migrations, experiments) is fine.
- Budget spent: stop feature releases for this service. Only fixes and reliability work ship until the budget recovers. The postmortem of the incident that spent it (Lesson 21.12) gets top priority.

The policy turns an endless argument ("is it stable enough?") into a number both developers and operators accepted beforehand.

## 5. Burn rate: alert on what matters

A naive alert says "page me when the error ratio is above 0.5%". It fires on every short spike, at 3 a.m., for problems that fix themselves, and people learn to ignore it.

The better question is: **at this speed, how soon will we spend the budget?** That speed is the **burn rate**:

```text
                current error ratio        0.05
  burn rate = ─────────────────────── = ──────── = 10      (5% errors, SLO 99.5%)
               error budget (1 − SLO)     0.005
```

Burn rate 1 spends exactly the 30-day budget in 30 days. Burn rate 10 spends it in 3 days. The SRE Workbook (see the sources) recommends combining a **long window** (is it real?) with a **short window** (is it still happening?):

| Alert | Burn rate | Long window | Short window | Budget spent when it fires | Action |
|---|---|---|---|---|---|
| fast burn | > 14.4 | 1 hour | 5 minutes | 2% in 1 hour | page the person on call |
| slow burn | > 6 | 6 hours | 30 minutes | 5% in 6 hours | ticket for working hours |

14.4 comes from the maths: in 1 hour of a 720-hour window, burn rate 14.4 spends `14.4 / 720 = 2%` of the budget. The short window makes the alert stop soon after the problem is fixed, instead of an hour later.

## 6. The SLO as Prometheus rules

Write the SLI as **recording rules**, so the ratio is computed once per evaluation and stored under a short name, and the alert as an **alerting rule**. `rules/slo-rules.yml`:

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

**What happens:**

1. A **group** is evaluated once per *evaluation interval*. The chart's default is one minute (scraping happens every 15 seconds, as we set).
2. `record:` stores the result of `expr` as a new series. The name follows the Prometheus convention `level:metric:operations`: aggregated (`slo`), what (`orders_errors`), how (`ratio_rate5m`).
3. The alert compares the error ratio with `14.4 × budget`. "Error ratio > 14.4 × 0.005" is the same as "burn rate > 14.4". `and` requires both windows.
4. `for: 2m`: the condition must stay true for 2 minutes before the alert *fires*; until then it is *pending*. This filters out single bad evaluations.
5. `labels` are added to the alert (Alertmanager uses `severity` to decide who is paged); `annotations` are the human text. `{{ $value | humanizePercentage }}` inserts the value of the expression as a percentage.

**Check and test the rules** before they go anywhere. `promtool check rules` finds syntax errors; `promtool test rules` runs **unit tests** with invented input series. `rules/slo-rules-test.yml`:

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

**What happens:**

1. `input_series` invents data, one value per `interval`. `'0+90x60'` means "start at 0, add 90, 60 times": a counter that grows by 90 per minute for an hour.
2. `promql_expr_test` checks a query at a moment (`eval_time: 10m`). `round(..., 0.001)` rounds to three decimals; see Common mistakes for why.
3. `alert_rule_test` says which alerts must be **firing** at that moment, with which labels and texts. At 10 minutes the condition has been true for longer than `for: 2m`.

Run both checks with the `promtool` that is inside the official Prometheus image (from the folder that contains `rules/`):

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

## 7. The SLO in the cluster

Create `course-21-11`, load `course-p21-orders:0.1.0` and apply the `k8s/orders.yaml` from Lesson 21.9, but with `FAIL_EVERY` set to `"10"`: a bad day with 10% errors. Then give the rules to Prometheus. The chart reads extra rule files from the value `serverFiles`; `k8s/prometheus-values.yaml` is the file from Lesson 21.9 plus the same groups under `serverFiles.alerting_rules.yml`:

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

In a second terminal, check that the rules are loaded and healthy:

```bash
curl -s localhost:9090/api/v1/rules | jq -c '.data.groups[] | {name, rules: [.rules[] | {name, health}]}'
```

```text
{"name":"orders-slo","rules":[{"name":"slo:orders_errors:ratio_rate5m","health":"ok"},{"name":"slo:orders_errors:ratio_rate1h","health":"ok"}]}
{"name":"orders-slo-alerts","rules":[{"name":"OrdersErrorBudgetFastBurn","health":"ok"}]}
```

Wait a few minutes, then ask for the SLI, the burn rate and the budget. Shortened `curl` commands, as in Lesson 21.9 (`curl -s localhost:9090/api/v1/query --data-urlencode 'query=...' | jq -c '.data.result[] | [.metric, .value[1]]'`):

```text
slo:orders_errors:ratio_rate5m
[{"__name__":"slo:orders_errors:ratio_rate5m"},"0.09749641565756222"]

slo:orders_errors:ratio_rate5m / 0.005
[{},"19.499283131512442"]

1 - (sum(increase(http_requests_total{path="/orders/{id}", status=~"5.."}[30d]))
     / sum(increase(http_requests_total{path="/orders/{id}"}[30d]))) / 0.005
[{},"-18.53401567218997"]
```

**What happens:**

1. About 9.7% of order requests fail: `FAIL_EVERY=10` in each pod, measured with `rate`, which estimates a little.
2. The burn rate is about 19.5: at this speed a 30-day budget lasts `30 / 19.5 ≈ 1.5` days.
3. "Budget left" is `1 − (error ratio / budget)` over the window. Our cluster is only minutes old, so the 30-day window contains only these minutes, and the result is about −18.5: the failures so far are about 19.5 times the whole budget. A negative number means "spent, and more".

After `for: 2m` the alert fires. The alerts API (Application Programming Interface) or the **Alerts** page of the UI (User Interface) shows it:

```bash
curl -s localhost:9090/api/v1/alerts \
  | jq -c '.data.alerts[] | {name: .labels.alertname, severity: .labels.severity, state, description: .annotations.description}'
```

```text
{"name":"OrdersErrorBudgetFastBurn","severity":"page","state":"firing","description":"9.692% of order requests fail (SLO allows 0.5%)."}
```

With Alertmanager switched on (it is part of the same chart), this alert would now be routed by its `severity: page` label to the pager of the person on call. What that person does next is the topic of Lesson 21.12.

> ⚠️ **Common mistake:** The rules live twice here, in `rules/slo-rules.yml` (for `promtool`) and in the values file (for the chart). If they drift apart, you test one thing and run another. In a real project, generate one from the other in CI (Continuous Integration, Lesson 21.7), or use the Prometheus Operator, where rules are Kubernetes objects (`PrometheusRule`) that you can keep in one file.

## Common mistakes

> ⚠️ **Common mistake:** Comparing a computed ratio exactly in a `promtool` test. Without `round`, the test fails although the maths is right:

```text
  FAILED:
    expr: "slo:orders_errors:ratio_rate5m", time: 10m,
        exp: {__name__="slo:orders_errors:ratio_rate5m"} 1E-01
        got: {__name__="slo:orders_errors:ratio_rate5m"} 9.999999999999999E-02
```

Floating-point division rarely gives exactly `0.1`. Round in the test, as above, or test the alert instead of the raw number.

> ⚠️ **Common mistake:** Counting 4xx as failures. Then a client that asks for many missing orders, or a scanner trying random URLs (Uniform Resource Locators, web addresses), spends *your* error budget, and the team gets paged for other people's mistakes.

> ⚠️ **Common mistake:** An SLO nobody looks at. An SLO is only useful if a dashboard shows the budget, an alert watches the burn rate, and the team agreed on a policy for an empty budget.

> ⚠️ **Common mistake:** Dividing by zero traffic. At night with no requests, `0 / 0` gives "no data" (`NaN`), and the alert neither fires nor resolves. That is acceptable for a burn-rate alert; for the "budget left" number, decide what no traffic means (our Rust function says: the full budget is left).

## Try it yourself

1. Change `FAIL_EVERY` to `"200"` (0.5% errors) with `kubectl -n shop set env deployment/orders FAIL_EVERY=200`. After about 3 minutes the alert disappears from the alerts list: the 5-minute window is filled with healthy traffic, and burn rate 1 is exactly on budget, nothing to page about. (The 1-hour window still remembers the bad minutes; this is why the alert needs *both* windows.)
2. In the Prometheus UI, open **Alerts** (<http://localhost:9090/alerts>) and watch the alert go from *inactive* to *pending* to *firing* after you set `FAIL_EVERY` back to `"10"`.
3. Run `cargo test` in `code/p21-slo` after changing the body of `close` to `a == b`. Which tests fail, and why? (Two of them: `three_nines_allow_43_minutes_in_30_days` and `errors_at_the_budget_burn_at_rate_one`. Their results differ from 43.2 and 1.0 only in the last digits, and `==` does not forgive that.) Change it back.
4. When you are done: `kind delete cluster --name course-21-11`.

## Exercises

### Exercise 1 — Budget in requests (easy)

A service received 2,400,000 requests in the last 30 days and has an availability SLO of 99.9%. 1,800 requests failed with 5xx. How many failures does the budget allow, how much of it is left, and what does the error budget policy from Section 4 say?

<details>
<summary>Hint</summary>

Allowed failures = `(1 − SLO) × total`. Left = `1 − failed / allowed`. You can check it with `budget_left` from `p21-slo`.

</details>

<details>
<summary>Answer</summary>

Allowed: `0.001 × 2,400,000 = 2,400` failed requests. Spent: `1,800 / 2,400 = 75%`, so 25% is left. `budget_left(0.999, 1_800, 2_400_000)` returns `0.25000000000000067`, printed with `{:.2}` as `0.25`.

Why it works: the budget is a count of allowed bad events in the window. With 25% left the team still ships normally, but it is a good moment to look at what spent the other 75% before a single bad deployment empties the rest.

</details>

### Exercise 2 — Test that a healthy service does not page (medium)

**Write a `promtool` test** `rules/healthy-test.yml` for the same rules: with 99 good and 1 failed request per minute (1% errors, burn rate 2), the alert `OrdersErrorBudgetFastBurn` must **not** fire at 30 minutes. Run it, then change the failed requests to 10 per minute and show that your test now fails.

<details>
<summary>Hint</summary>

An empty list `exp_alerts: []` means "no alert with this name is firing at `eval_time`".

</details>

<details>
<summary>Answer</summary>

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

With `'0+10x60'` for the failed requests:

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

Why it works: at 1% errors the ratio (0.01) is below `14.4 × 0.005 = 0.072`, so the alert must stay quiet; at 10 failed per 99 good it is about 9.2%, above the threshold. A pair of tests like this — "pages when it should" and "stays quiet when it should" — protects the alert from future edits.

</details>

### Exercise 3 — A latency SLO (hard)

Add a second SLI: the share of order requests that finished in at most 100 ms, over 5 minutes, as a recording rule `slo:orders_latency_good:ratio_rate5m`. Use the histogram from Lesson 21.9. What value does it have in the cluster, and why? Then write the SLO in one sentence.

<details>
<summary>Hint</summary>

The bucket `le="0.1"` counts requests that took at most 0.1 s; `_count` counts all of them. Divide their rates.

</details>

<details>
<summary>Answer</summary>

Add to the group `orders-slo` (in both files):

```yaml
      - record: slo:orders_latency_good:ratio_rate5m
        expr: |
          sum(rate(http_request_duration_seconds_bucket{path="/orders/{id}", le="0.1"}[5m]))
            / sum(rate(http_request_duration_seconds_count{path="/orders/{id}"}[5m]))
```

In the Prometheus UI the same expression gives:

```text
[{},"1"]
```

All order requests take about 31 ms (Lesson 21.9), far below 100 ms, so the share of good requests is 1, that is 100%.

The SLO: "99% of order requests finish in at most 100 ms, measured over 30 days." Why it works: the bucket limit 0.1 exists in our histogram, so the ratio is exact, not an estimate as with `histogram_quantile`. Choose latency thresholds that are bucket limits.

</details>

## Quiz

1. What is the difference between an SLI, an SLO and an SLA?

<details>
<summary>Answer</summary>

The SLI is the measurement (share of good requests). The SLO is the team's goal for it over a window (99.5% in 30 days). The SLA is a contract with customers, with consequences, and is looser than the SLO.

</details>

2. Why is a 404 answer usually a *good* event for an availability SLI?

<details>
<summary>Answer</summary>

The service worked correctly: it told the client that the thing does not exist. 4xx are client errors; only 5xx mean that the service failed.

</details>

3. How much full outage does a 99.9% SLO allow in 30 days?

<details>
<summary>Answer</summary>

`0.001 × 30 × 24 × 60 = 43.2` minutes.

</details>

4. The error ratio is 2% and the SLO is 99%. What is the burn rate, and how long until a 30-day budget is gone?

<details>
<summary>Answer</summary>

`0.02 / 0.01 = 2`. The budget lasts `30 / 2 = 15` days.

</details>

5. Why does the fast-burn alert use two windows (1 hour and 5 minutes)?

<details>
<summary>Answer</summary>

The long window makes sure the problem is real and big enough to matter; the short window makes sure it is still happening, so the alert stops soon after the fix.

</details>

6. What does `for: 2m` do in an alerting rule?

<details>
<summary>Answer</summary>

The condition must stay true for 2 minutes before the alert fires; until then it is pending. Single bad evaluations do not page anybody.

</details>

## Summary

- SRE sets reliability goals in numbers. 100% is the wrong goal: impossible, invisible to users, and it stops all change.
- An SLI is "good events / valid events" from the user's view; 5xx are bad, 4xx are usually good; health checks are not users.
- An SLO is a target for an SLI over a rolling window; an SLA is a looser promise in a contract.
- The error budget is `1 − SLO`; an agreed policy says what happens when it is spent.
- Alert on the burn rate with a long and a short window (14.4 over 1 hour and 5 minutes pages), not on single errors.
- Write SLIs as recording rules and the alert as an alerting rule, check them with `promtool check rules`, and test them with `promtool test rules`.

## Sources and further reading

- [Site Reliability Engineering — Embracing Risk](https://sre.google/sre-book/embracing-risk/) and [Service Level Objectives](https://sre.google/sre-book/service-level-objectives/) — the original chapters on error budgets, SLIs and SLOs.
- [The Site Reliability Workbook — Implementing SLOs](https://sre.google/workbook/implementing-slos/) and [Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/) — practical SLO design and the multiwindow burn-rate alerts used here.
- [The Site Reliability Workbook — Example error budget policy](https://sre.google/workbook/error-budget-policy/) — a complete policy document to adapt.
- [Prometheus — Recording rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/), [Alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/) and [Rule naming](https://prometheus.io/docs/practices/rules/) — rule syntax and the `level:metric:operations` convention.
- [Prometheus — Unit testing for rules](https://prometheus.io/docs/prometheus/latest/configuration/unit_testing_rules/) — the full `promtool test rules` file format.

---

[🏠 Start](../START-HERE.md) · [← 21.10 Logs and traces in a cluster](10-logs-and-traces-in-a-cluster.md) · [📋 Phase 21](README.md) · [21.12 Incidents, on-call and postmortems →](12-incidents-on-call-and-postmortems.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
