<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 21.13 Project: run your service on Kubernetes](13-project-run-your-service-on-kubernetes.md) · [📋 Phase 21](README.md) · [Phase 22 — Specializations →](../22-specializations/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/21-cloud-native-infrastructure/14-checkpoint.md) <!-- nav -->

# Lesson 21.14 — Checkpoint

> **Phase 21 — Cloud-native infrastructure** · Lesson 14 of 14 · ⏱ about 90 minutes

## What you will learn

- Whether you can explain how Kubernetes runs, configures, updates and heals a service.
- Whether you can package and release a service, and describe infrastructure as code.
- Whether you can observe a service with metrics, logs and traces, set an SLO (Service Level Objective), and alert on its burn rate.
- Whether you can respond to an incident and write a useful postmortem.
- Which lesson of this phase to read again for each gap.

## Before you start

- You finished [Lesson 21.13 — Project: run your service on Kubernetes](13-project-run-your-service-on-kubernetes.md), or at least its milestones M1–M4.
- Do the self-test **without looking** at the lessons. Write your answers down, then open the hidden answers. The practical tasks may be done with the lessons open.
- Tools: Docker (for `promtool`) and, for Task 5, a kind cluster. Your files go in `~/projects/rust/code/p21-checkpoint`.

## New words

| Word | Meaning |
|---|---|
| **slow-burn alert** | An SLO alert for a moderate burn rate over long windows (here 6 over 6 hours and 30 minutes). It creates a ticket instead of waking someone up. |
| **ticket** | A task in the team's tracker, handled in working hours. The opposite of a page. |

## Self-test quiz

1. What does "declarative" mean in Kubernetes, and which component makes the declared state true?

<details>
<summary>Answer</summary>

You declare the desired state in a manifest ("3 replicas of this image"); controllers compare it with the actual state again and again and act on the difference (the reconciliation loop, Lesson 21.1). The Deployment controller, for example, creates or removes pods.

</details>

2. What is the difference between a pod, a ReplicaSet and a Deployment?

<details>
<summary>Answer</summary>

A pod runs one or more containers. A ReplicaSet keeps a fixed number of identical pods running. A Deployment manages ReplicaSets so that you can update the pod template step by step and roll back.

</details>

3. How does a Service find its pods, and why is that useful?

<details>
<summary>Answer</summary>

By a label selector, for example `app: orders`. Pods come and go with new names and addresses; the Service gives them one stable name and address and spreads the connections over the ready ones.

</details>

4. Your service listens on `127.0.0.1:8080` in the container. The pod is `Running`, but the Service gives "connection refused". Why?

<details>
<summary>Answer</summary>

`127.0.0.1` accepts connections only from inside the container. The Service connects to the pod's IP (Internet Protocol) address, so the program must listen on `0.0.0.0` (Lesson 21.4).

</details>

5. Which settings go into a ConfigMap and which into a Secret? Is a Secret encrypted?

<details>
<summary>Answer</summary>

Non-secret configuration (ports, limits, URLs, Uniform Resource Locators, without passwords) goes into a ConfigMap; passwords, tokens and keys into a Secret. By default a Secret is only base64-encoded (a reversible text encoding, not encryption): protect it with access rights, and keep its values out of Git (Lesson 21.5).

</details>

6. What happens when a readiness probe fails, and what happens when a liveness probe fails?

<details>
<summary>Answer</summary>

Readiness: the pod is removed from the Service's endpoints and gets no traffic, but keeps running. Liveness: Kubernetes kills and restarts the container. That is why liveness must never depend on the database.

</details>

7. What does Helm add on top of plain manifests? Name two commands that use it.

<details>
<summary>Answer</summary>

Templates with values, so one chart serves many environments, and **releases** with a history of revisions. `helm upgrade --install` creates or updates a release; `helm rollback <release> <revision>` returns to an older one (`helm history` lists them). Kustomize instead patches plain YAML with overlays, without templates (Lesson 21.6).

</details>

8. What is the difference between continuous integration, continuous delivery and continuous deployment (together: CI/CD)?

<details>
<summary>Answer</summary>

Integration: every change is built and tested automatically. Delivery: every change that passes is turned into a release candidate (for example an image in a registry) that *can* go to production with one decision. Deployment: it goes to production automatically, without a human step (Lesson 21.7).

</details>

9. Why does OpenTofu need a state file, and why must it not be in Git?

<details>
<summary>Answer</summary>

The state maps resource blocks to real objects; without it OpenTofu could not know what to destroy when a block is removed. It contains secret values and must be locked during an apply, so it belongs in a protected backend, not in Git (Lesson 21.8).

</details>

10. Somebody scaled a Deployment by hand. What will `tofu plan` (or the next Helm release) do, and what is the rule behind it?

<details>
<summary>Answer</summary>

It shows the drift and the next apply puts the value back. Rule: managed infrastructure is changed only through its source of truth, the files in Git; every object has exactly one owner.

</details>

11. Why does Prometheus scrape every pod instead of the Service?

<details>
<summary>Answer</summary>

Through the Service every scrape lands on a random pod, so a counter would jump between the values of different pods and `rate()` would be wrong. Each pod is its own target; queries add them up with `sum`.

</details>

12. Write a PromQL (Prometheus Query Language) query for the 5xx error ratio of the order route over 5 minutes.

<details>
<summary>Answer</summary>

```text
sum(rate(http_requests_total{path="/orders/{id}", status=~"5.."}[5m]))
  / sum(rate(http_requests_total{path="/orders/{id}"}[5m]))
```

</details>

13. Why must metric labels have few values? Give one bad and one good label.

<details>
<summary>Answer</summary>

Every distinct value creates new time series that cost memory in the service and in Prometheus. Bad: `user_id`, the raw path `/orders/7`. Good: the route pattern `/orders/{id}`, `status`, `method`.

</details>

14. Where does `kubectl logs` read from, and why do you need a log collector?

<details>
<summary>Answer</summary>

From the log files that the container runtime writes on the node, served by the kubelet. They are rotated and deleted with the pod or the node, so a node agent (a DaemonSet) must copy them to a log store (Lesson 21.10).

</details>

15. An SLO is 99.9% over 30 days. How big is the error budget in minutes of full outage, and what is the burn rate at 0.5% errors?

<details>
<summary>Answer</summary>

`0.001 × 30 × 24 × 60 = 43.2` minutes. Burn rate `0.005 / 0.001 = 5`: the budget would last 6 days.

</details>

16. Why does the fast-burn alert use both a 1-hour and a 5-minute window?

<details>
<summary>Answer</summary>

The long window proves that the problem is real and large; the short window proves it is still happening, so the alert stops soon after the fix.

</details>

17. During an incident, the pods are green but half of the requests fail. What do you ask first, and what do you do?

<details>
<summary>Answer</summary>

"What changed?" (`kubectl rollout history`, recent releases). If a recent change is the likely cause, mitigate first with a rollback, then confirm with a smoke test, then fix the source of truth and investigate (Lesson 21.12).

</details>

18. Name three parts of a blameless postmortem.

<details>
<summary>Answer</summary>

Any three of: summary and impact; a timeline with times; contributing factors (found by asking "why?" several times, without blaming people); what went well; action items with owners and due dates.

</details>

## Practical tasks

### Task 1 — Budget and burn rate on paper

A service has an availability SLO of 99.5% over 30 days. In the last 30 days it served 8,000,000 requests, and 30,000 of them failed with 5xx. Right now, the error ratio over the last hour is 4%.

1. How many failures does the budget allow, and how much of it is left?
2. What is the current burn rate? Would the fast-burn alert (14.4) or the slow-burn alert (6) fire, if the short windows agree?
3. What does a typical error budget policy say now?

<details>
<summary>Hint</summary>

Allowed = `(1 − SLO) × total`; burn rate = error ratio / `(1 − SLO)`. You can check the first answer with `budget_left` from `code/p21-slo` (Lesson 21.11).

</details>

<details>
<summary>Answer</summary>

1. Allowed: `0.005 × 8,000,000 = 40,000`. Spent: `30,000 / 40,000 = 75%`; 25% is left.
2. Burn rate `0.04 / 0.005 = 8`. Above 6, below 14.4: the slow-burn alert fires (a ticket), the fast-burn alert does not page. At this speed the remaining 25% of a 30-day budget is gone in about `0.25 × 30 / 8 ≈ 0.9` days.
3. Budget still left: features may ship, but the team should look at the cause of the 4% now; if the budget runs out, the policy stops feature releases until it recovers.

</details>

### Task 2 — Two more PromQL questions

Using the metrics of the order service (Lesson 21.9), write queries for: (a) the 99th percentile of order latency over 5 minutes; (b) the request rate per pod over 1 minute, only for the namespace `shop`.

<details>
<summary>Hint</summary>

(a) is Query 4 of Lesson 21.9 with a different quantile and window. (b) is in "Try it yourself" of the same lesson.

</details>

<details>
<summary>Answer</summary>

```text
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket{path="/orders/{id}"}[5m])))
```

```text
sum by (pod) (rate(http_requests_total{namespace="shop"}[1m]))
```

Why it works: `histogram_quantile` needs the buckets summed over pods but grouped by `le`; the per-pod rate keeps only the `pod` label after summing.

</details>

### Task 3 — A slow-burn alert with tests

In `code/p21-checkpoint/slow-burn.yml`, write two recording rules (error ratio of the order route over 30 minutes and 6 hours) and an alert `OrdersErrorBudgetSlowBurn` for an SLO of 99.5%: burn rate above 6 in both windows, `for: 15m`, label `severity: ticket`. Then **write a `promtool` test** with two cases: 4% errors for 8 hours fires at 7 hours; 2% errors never fires.

<details>
<summary>Hint</summary>

Follow `rules/slo-rules.yml` and `rules/slo-rules-test.yml` of Lesson 21.11. For 8 hours of data at one value per minute you need `x480`. A test file can contain several entries under `tests:`, each with its own `input_series`.

</details>

<details>
<summary>Answer</summary>

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

Why it works: 4% is 8 times the 0.5% budget, so both windows are above `6 × 0.005 = 0.03` long before 7 hours, and `for: 15m` has passed. 2% stays below 0.03 all the time.

</details>

### Task 4 — Read an incident

A release has just happened. You see this:

```text
NAME                      READY   STATUS             RESTARTS   AGE
orders-7878c5fd55-k69q8   0/1     ImagePullBackOff   0          45s
orders-f6597ddff-6zxg2    1/1     Running            0          50s
orders-f6597ddff-t52jf    1/1     Running            0          51s
```

Are users affected? What is the most likely cause, which command proves it, and what do you do?

<details>
<summary>Hint</summary>

Count the ready pods of the old ReplicaSet. Then look at the events of the pod that is not ready.

</details>

<details>
<summary>Answer</summary>

Users are not affected: both old pods are ready and serve traffic; the rolling update does not remove an old pod before a new one is ready. The new image cannot be pulled, most likely a wrong tag or a missing image. `kubectl -n shop describe pod orders-7878c5fd55-k69q8` shows it in the events (`Failed to pull image ... repository does not exist`, `ErrImagePull`). Roll back with `kubectl -n shop rollout undo deployment/orders` (or fix the tag in Git and release again), then fix the pipeline so that a release can only use an image that was built and pushed (Lesson 21.12, Exercise 3).

</details>

### Task 5 — Infrastructure as code from memory

Create `course-21-14` with kind, and in `code/p21-checkpoint/infra` write an OpenTofu configuration (Kubernetes provider) that creates a namespace `checkpoint` and a ConfigMap `settings` with one key `mode = "practice"`. Run `tofu init`, `tofu plan` and `tofu apply`. Then change the ConfigMap by hand with `kubectl -n checkpoint edit configmap settings` and show the drift with `tofu plan`. Destroy everything and delete the cluster.

<details>
<summary>Hint</summary>

Copy `versions.tf` from Lesson 21.8 and change the context to `kind-course-21-14`. You need two resources: `kubernetes_namespace_v1` and `kubernetes_config_map_v1`, the second referring to the first.

</details>

<details>
<summary>Answer</summary>

The resources (`main.tf`):

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

`tofu plan` before the first apply ends with `Plan: 2 to add, 0 to change, 0 to destroy.` After a hand edit (for example to `hand-edited`; `kubectl -n checkpoint patch configmap settings --type merge -p '{"data":{"mode":"hand-edited"}}'` does the same without an editor) it shows `~ update in-place` for `kubernetes_config_map_v1.settings` with the line `~ "mode" = "hand-edited" -> "practice"` and `Plan: 0 to add, 1 to change, 0 to destroy.` `tofu destroy` ends with `Destroy complete! Resources: 2 destroyed.`

Why it works: OpenTofu refreshes the real objects before each plan and compares them with the files, exactly as in Section 4 of Lesson 21.8.

</details>

## If you got something wrong

| Topic | Lesson to read again |
|---|---|
| Desired state, reconciliation, what the platform expects | [21.1 What cloud native means](01-what-cloud-native-means.md) |
| Pods, Deployments, ReplicaSets, Services, labels | [21.2 Kubernetes concepts](02-kubernetes-concepts.md) |
| kind, kubectl, contexts | [21.3 A local cluster with kind and kubectl](03-a-local-cluster-with-kind.md) |
| Images in the cluster, rolling updates, `0.0.0.0` | [21.4 Deploying a Rust service to Kubernetes](04-deploying-a-rust-service-to-kubernetes.md) |
| ConfigMaps, Secrets, readiness and liveness probes | [21.5 ConfigMaps, Secrets and health probes](05-configmaps-secrets-and-probes.md) |
| Charts, values, releases, overlays | [21.6 Packaging with Helm and Kustomize](06-helm-and-kustomize.md) |
| CI, CD, pipelines that build and release | [21.7 CI/CD: from commit to production](07-ci-cd-from-commit-to-production.md) |
| OpenTofu, state, drift, `tofu test` | [21.8 Infrastructure as code with OpenTofu](08-infrastructure-as-code-with-opentofu.md) |
| Prometheus, PromQL, histograms, cardinality, Grafana | [21.9 Metrics with Prometheus and Grafana](09-metrics-with-prometheus-and-grafana.md) |
| `kubectl logs`, log collection, traces in a cluster | [21.10 Logs and traces in a cluster](10-logs-and-traces-in-a-cluster.md) |
| SLI (Service Level Indicator), SLO, error budget, burn-rate alerts, `promtool` | [21.11 SRE basics: SLIs, SLOs and error budgets](11-sre-basics-slis-slos-and-error-budgets.md) |
| Severity, on-call, mitigation, postmortems | [21.12 Incidents, on-call and postmortems](12-incidents-on-call-and-postmortems.md) |
| Putting it all together | [21.13 Project: run your service on Kubernetes](13-project-run-your-service-on-kubernetes.md) |

## What's next

You can now run a Rust service the way a platform team runs it: packaged, released, observed, measured against a goal, and repaired calmly when it breaks. The next phase, [Phase 22 — Specializations](../22-specializations/README.md), lets you go deep into one direction. If this phase was your favourite, choose the **cloud-native track**: there you write a Kubernetes *operator* in Rust with the `kube` crate — your own controller with its own reconciliation loop, the same idea that runs Deployments. Afterwards, [Phase 23 — Capstone projects](../23-capstone-projects/README.md) brings everything together in one large project of your own.

Before you go on, delete every cluster you still have: `kind get clusters`, then `kind delete cluster --name <name>` for each one.

## Sources and further reading

- [Kubernetes documentation — Concepts](https://kubernetes.io/docs/concepts/) — the official reference for every object used in this phase.
- [OpenTofu documentation](https://opentofu.org/docs/) — the language, the command-line tool and state.
- [Prometheus documentation](https://prometheus.io/docs/introduction/overview/) — metrics, PromQL, rules and unit tests.
- [The Site Reliability Workbook](https://sre.google/workbook/table-of-contents/) — SLOs, alerting, on-call and postmortems in practice, from the people who named SRE (Site Reliability Engineering).

---

[🏠 Start](../START-HERE.md) · [← 21.13 Project: run your service on Kubernetes](13-project-run-your-service-on-kubernetes.md) · [📋 Phase 21](README.md) · [Phase 22 — Specializations →](../22-specializations/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
