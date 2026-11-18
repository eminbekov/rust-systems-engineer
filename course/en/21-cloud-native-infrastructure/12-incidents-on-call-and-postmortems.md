<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 21.11 SRE basics: SLIs, SLOs and error budgets](11-sre-basics-slis-slos-and-error-budgets.md) · [📋 Phase 21](README.md) · [21.13 Project: run your service on Kubernetes →](13-project-run-your-service-on-kubernetes.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/21-cloud-native-infrastructure/12-incidents-on-call-and-postmortems.md) <!-- nav -->

# Lesson 21.12 — Incidents, on-call and postmortems

> **Phase 21 — Cloud-native infrastructure** · Lesson 12 of 14 · ⏱ about 105 minutes

## What you will learn

- What an incident is, and how severity levels decide how loud the response is.
- How on-call works: rotations, pages, runbooks, and what deserves to wake somebody up.
- The steps and roles of an incident response, and why you *mitigate first* and look for the cause later.
- How to practise a real incident in a kind cluster: a bad release, diagnosis with `kubectl`, and a rollback.
- How to write a blameless postmortem with a timeline and action items.

## Before you start

- You finished [Lesson 21.11 — SRE basics: SLIs, SLOs and error budgets](11-sre-basics-slis-slos-and-error-budgets.md). The burn-rate alert of that lesson, an alert on the SLO (Service Level Objective), is the "page" that starts an incident here.
- You know Deployments, rollouts and `kubectl rollout` from [Lesson 21.4](04-deploying-a-rust-service-to-kubernetes.md), and how to read logs and events from [Lesson 21.10](10-logs-and-traces-in-a-cluster.md).
- You have the image `course-p21-orders:0.1.0` from Lesson 21.9.

## New words

| Word | Meaning |
|---|---|
| **incident** | An unplanned event that hurts users or puts the service at risk and needs a coordinated response now. |
| **severity** (SEV1, SEV2, SEV3) | How bad an incident is. SEV is short for *severity*, and the number is the level: SEV1 is the worst. It decides who is called, how fast, and how often people get updates. |
| **on-call** | The duty of being reachable and ready to respond to pages during a shift. |
| **rotation** | The schedule that passes on-call duty from person to person, for example one week each. |
| **runbook** | A short document for one alert: what it means, how to check, how to mitigate, whom to call. |
| **mitigation** | An action that stops the damage quickly (roll back, switch traffic, add capacity), even before the cause is understood. |
| **IC** (Incident Commander) | The person who leads the response: decides, delegates and keeps the overview. They do not fix things themselves. |
| **postmortem** | A written analysis after an incident: what happened, why, what was learned, and what will change. |
| **blameless** | A postmortem rule: look for causes in systems and processes, never for a guilty person. |
| **action item** | A concrete task from a postmortem with an owner and a due date. |
| **MTTR** (Mean Time To Recovery) | The average time from the start of an incident to recovery. Fast mitigation keeps it low. |

## 1. What an incident is

Not every problem is an incident. A slow test in CI (Continuous Integration) is a problem; an incident is something that hurts users **now** (or is about to) and needs people to coordinate. Teams declare an incident quickly and cancel it if it turns out small: declaring costs little, being late costs a lot.

**Severity** sets how loud the response is. Every company writes its own table; a typical one:

| Severity | Example | Response |
|---|---|---|
| **SEV1** critical | Data lost, or most users cannot use the product | Page now, all needed people join, updates every 30 minutes, postmortem required |
| **SEV2** major | A feature is broken for a part of users; the error budget burns fast | Page the person on call, updates every hour, postmortem required |
| **SEV3** minor | Small or partial problem, a workaround exists | Ticket, fix in working hours, postmortem optional |

Severity can change during an incident. Raise it as soon as you learn it is worse; lower it when the damage is contained.

## 2. On-call

Someone has to answer when the alert from Lesson 21.11 fires at 3 a.m. That person is **on call**. A team shares the duty in a **rotation**: for example one week each, a *primary* who receives the pages and a *secondary* as backup. At the end of a shift there is a short **handover**: what happened, what is still open.

What makes on-call bearable, and safe for the service:

- **Page only for symptoms that hurt users** and need a human now: SLO burn-rate alerts, not "CPU (Central Processing Unit) at 85%". A cause-based alert that users do not feel becomes a ticket.
- **Every page needs a runbook.** Half-asleep, nobody should have to invent the first steps.
- **Few pages.** The SRE (Site Reliability Engineering) Workbook suggests at most a couple of incidents per shift; if there are more, fix the alerts or the service. People who get many false pages stop reacting: *alert fatigue*.
- **Time to recover.** A night with pages is followed by rest; on-call is work and is paid or compensated.

A runbook for our alert, kept in Git next to the rules (`runbooks/orders-error-budget-fast-burn.md`):

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

## 3. The incident response

An incident has a life cycle:

```text
 detect ──► triage ──► mitigate ──► resolve ──► follow up
 (alert,    (declare,   (stop the     (cause      (postmortem,
  user       severity,   damage:       fixed,      action items)
  report)    roles)      rollback)     all clear)
```

**Mitigate first.** Users do not care *why* it broke; they care that it works again. The fastest safe mitigation is often to undo the last change: roll back the release, turn off a feature flag, move traffic to a healthy region. Understanding the cause comes after users are safe. A team that debugs a broken release for an hour, instead of rolling back in one minute, has a long MTTR (Mean Time To Recovery).

**Roles.** In a small incident, one person does everything. As soon as more people join, separate the roles:

| Role | Does | Does not |
|---|---|---|
| **IC** (Incident Commander) | Leads: keeps the overview, decides, assigns tasks, declares the end | Type commands |
| **Operations** | Investigates and changes systems, says each action aloud before doing it | Change things nobody asked for |
| **Communications** | Writes regular updates for users, support and management | Guess about causes |
| **Scribe** (often the IC) | Writes the timeline: times, observations, decisions | Edit the timeline later to look better |

Work happens in one place, for example one chat channel per incident, so the timeline writes itself. Updates follow a fixed form: *what is broken, what is the impact, what are we doing, when is the next update*.

> 💡 **Tip:** "Say it before you do it" sounds slow, but it prevents the worst incident mistake: two people changing the same system at once, each making the other's change invisible.

## 4. A practice incident

Practise in a calm moment what you will need in a stressful one. Create a cluster and deploy the healthy service (the `k8s/orders.yaml` of Lesson 21.9 without the `env` block, so nothing fails):

```bash
kind create cluster --name course-21-12
kind load docker-image course-p21-orders:0.1.0 --name course-21-12
kubectl apply -f k8s/orders.yaml
kubectl -n shop rollout status deployment/orders
kubectl -n shop annotate deployment/orders kubernetes.io/change-cause="release 0.1.0"
```

The annotation `kubernetes.io/change-cause` is a note for humans; `kubectl rollout history` shows it per revision.

A **smoke test** is a tiny test that only checks "does it basically work?". Ours sends 100 order requests from inside the cluster and counts failures. Keep it in a shell variable to reuse it:

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

(The first two lines are a standard notice of `kubectl run -i`; we leave them out below.) Now a colleague ships a "small config change". In real life it would be a bad environment variable or flag; we use `FAIL_EVERY=2`:

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

The rollout **succeeded**: the new pods answer `/healthz`, so the readiness probe is happy. A few minutes later the burn-rate alert of Lesson 21.11 would page you. Here, the smoke test plays the alert:

```bash
kubectl -n shop run smoke --rm -i --restart=Never --image=busybox:1.37 -- sh -c "$SMOKE"
```

```text
failed: 48 of 100
```

**Triage.** Half of the requests fail: users are clearly hurt. Declare an incident; one service, a part of the requests: SEV2. Now the questions in the order of the runbook. First, are the pods healthy?

```bash
kubectl -n shop get pods
```

```text
NAME                      READY   STATUS    RESTARTS   AGE
orders-5766c88465-6tjrj   1/1     Running   0          5s
orders-5766c88465-thsl9   1/1     Running   0          6s
```

(Old pods that were still `Terminating` are left out.) Everything looks green. This is a classic: **probes check that the process lives, not that it works for users.** That is why the page comes from an SLO and not from pod status. Second question, the most useful one in every incident: **what changed?**

```bash
kubectl -n shop rollout history deployment/orders
```

```text
deployment.apps/orders 
REVISION  CHANGE-CAUSE
1         release 0.1.0
2         config: FAIL_EVERY=2
```

A change minutes before the errors: the prime suspect. (To confirm, `kubectl -n shop port-forward deployment/orders 3000:3000` and `curl -s localhost:3000/metrics | grep '^http_requests_total'` show `status="500"` growing as fast as `status="200"` in the new pods.) **Mitigate** by undoing the change, without waiting for the full analysis:

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

Two failures right after the rollout: a few requests still reached the old, bad pods while they were shutting down. Run the smoke test once more a few seconds later; it prints `failed: 0 of 100`. Users are safe again. Look at the history once more:

```bash
kubectl -n shop rollout history deployment/orders
```

```text
deployment.apps/orders 
REVISION  CHANGE-CAUSE
2         config: FAIL_EVERY=2
3         release 0.1.0
```

**What happens:**

1. `rollout undo` creates a **new** revision (3) with the template of the previous one (1). Revision 1 disappears from the list because it is now revision 3.
2. The warning matters. The rollback changed the cluster, but **not the source of truth**: your files in Git (or the Helm values, or the OpenTofu configuration of Lesson 21.8). If nobody fixes them, the next `kubectl apply` or pipeline run brings the bad config back. That is drift, created on purpose. Fixing the source is part of *resolving* the incident.
3. The incident is over for users, but not for the team: now comes the postmortem.

> 🔒 **Security:** During an incident people are tempted to take shortcuts: give everybody admin rights, paste secrets into the chat, turn off security checks "for a minute". Write every such emergency change into the timeline and undo it afterwards; attackers like confused moments.

## 5. The blameless postmortem

A **postmortem** turns a bad day into lasting improvement. It is written within a few days, by the people involved, and read by the whole team. Its most important rule: it is **blameless**. "A colleague typed the wrong value" is not a cause; people make mistakes, and the question is why the system allowed one mistake to hurt users. When people are blamed, they hide information, and the next incident is worse.

A postmortem for our practice incident (times in UTC, Coordinated Universal Time, from our run):

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

The **contributing factors** use "why?" several times: why did users see errors? Because the config made the service fail. Why did the config reach production? Because it skipped review. Why did nothing stop it? Because the probes cannot see the order path. Each "why" points to a system you can improve, which is what **action items** are for. An action item without an owner and a date is a wish, not a plan.

> 📏 **Convention:** Postmortems are shared widely and stored where everybody can find them. Their value comes from others learning from them, including other teams.

## Common mistakes

> ⚠️ **Common mistake:** Debugging before mitigating. The cause is interesting, but every minute of debugging is a minute of users failing. If a recent change is the likely cause, roll back first.

> ⚠️ **Common mistake:** Trusting green pods. `Running` and `1/1 READY` mean that the probe passes. They say nothing about whether users get correct answers. Measure what users get (SLO, smoke test).

> ⚠️ **Common mistake:** Forgetting the source of truth after a rollback. The cluster is fixed, Git is not; the next deploy repeats the incident. Revert the change in Git as part of the resolution.

> ⚠️ **Common mistake:** A postmortem that names a culprit, or one with action items like "be more careful". Neither changes the system. Write actions that make the mistake impossible or harmless.

## Try it yourself

1. Repeat the practice incident, but this time play the roles aloud with a friend (or alone): the IC says "declare SEV2, check what changed", Operations says "I will roll back revision 2" before typing it.
2. Write the status update you would send to support after the smoke test fails, in four sentences: what is broken, impact, what we do, next update.
3. After the rollback, run `kubectl apply -f k8s/orders.yaml` and `kubectl -n shop rollout history deployment/orders`. The file has no `FAIL_EVERY`, so nothing breaks; think about what would have happened if the bad value were in the file.
4. When you are done: `kind delete cluster --name course-21-12`.

## Exercises

### Exercise 1 — Choose the severity (easy)

Using the table from Section 1, choose a severity for each case and say who is called: (a) the shop's search is slow for 3% of users, a workaround is to browse categories; (b) a bug deleted the order history of all customers of one country; (c) the error-budget burn-rate alert of the orders service fires at 02:00, about 10% of order requests fail.

<details>
<summary>Hint</summary>

Ask two questions: is data lost? How many users are affected, and can they work around it?

</details>

<details>
<summary>Answer</summary>

(a) SEV3: few users, a workaround exists; a ticket for working hours. (b) SEV1: data loss, whatever the number of users; page now and bring in the needed people, including the database team. (c) SEV2: a part of the requests fails and the budget burns fast; the page goes to the person on call, who may raise it to SEV1 if it grows.

Why it works: severity measures the harm to users and data, not how interesting or how hard the technical problem is.

</details>

### Exercise 2 — Severity as code, with tests (medium)

Write a small program `code/p21-severity` with an enum `Severity { Sev1, Sev2, Sev3 }` and a function `severity(affected_users: f64, data_lost: bool) -> Severity`: data loss or at least 50% of users is SEV1, at least 5% is SEV2, everything else SEV3. **Write tests** for data loss, for 50%, and for both sides of the 5% border. `main` prints four cases.

<details>
<summary>Hint</summary>

`#[derive(Debug, PartialEq)]` lets you print the enum with `{:?}` and compare it in `assert_eq!`. Test borders with values exactly on and just below the limit.

</details>

<details>
<summary>Answer</summary>

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

Why it works: the `if` checks the most severe condition first, so data loss wins even with few users. The border tests catch the most common bug in such rules, `>` instead of `>=`. A team rule written as code and tests cannot be "remembered differently" at 3 a.m.

</details>

### Exercise 3 — A stuck rollout (hard)

In the practice cluster, release an image that does not exist: `kubectl -n shop set image deployment/orders orders=course-p21-orders:9.9.9`, then `kubectl -n shop rollout status deployment/orders --timeout=45s`. Find out what is wrong using only `kubectl`, run the smoke test, and explain why users are **not** affected. Then mitigate.

<details>
<summary>Hint</summary>

`kubectl get pods` shows the status of the new pod; `kubectl describe pod <name>` shows its events at the end. Think about how a rolling update replaces pods.

</details>

<details>
<summary>Answer</summary>

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

The events of the new pod (shortened):

```text
  Normal   Pulling    4s (x3 over 45s)   kubelet            spec.containers{orders}: Pulling image "course-p21-orders:9.9.9"
  Warning  Failed     3s (x3 over 44s)   kubelet            spec.containers{orders}: Failed to pull image "course-p21-orders:9.9.9": failed to pull and unpack image "docker.io/library/course-p21-orders:9.9.9": failed to resolve reference "docker.io/library/course-p21-orders:9.9.9": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
  Warning  Failed     3s (x3 over 44s)   kubelet            spec.containers{orders}: Error: ErrImagePull
```

The smoke test prints `failed: 0 of 100`. Mitigation: `kubectl -n shop rollout undo deployment/orders` prints `deployment.apps/orders rolled back` (plus the same warning as above), and the broken pod is removed.

Why it works: the image is not in the node (we never loaded `9.9.9`), so the kubelet tries Docker Hub and fails: `ErrImagePull`, then `ImagePullBackOff` (waiting longer between tries). A rolling update starts a new pod first and removes an old one only when the new one is **ready** (for 2 replicas: at most 1 extra pod, no pod unavailable). The new pod never becomes ready, so both old pods keep serving. The rollout is stuck, not broken: a SEV3 at most, but fix it, because the next real release is blocked.

</details>

## Quiz

1. Why should you declare an incident early, even if it might turn out small?

<details>
<summary>Answer</summary>

Declaring costs little and can be cancelled; declaring late means lost time without coordination, roles or communication while users are hurt.

</details>

2. What should page the person on call: "CPU at 85%" or "the error budget burns 14 times too fast"? Why?

<details>
<summary>Answer</summary>

The burn-rate alert: it is a symptom that users feel and needs action now. High CPU (Central Processing Unit) usage may be harmless; it belongs on a dashboard or in a ticket.

</details>

3. What does "mitigate first" mean in practice?

<details>
<summary>Answer</summary>

Stop the damage with the fastest safe action, often undoing the last change (rollback, feature flag off), before understanding the exact cause.

</details>

4. What does the Incident Commander do, and what do they *not* do?

<details>
<summary>Answer</summary>

The IC leads: keeps the overview, decides, assigns tasks, organises communication and declares the end. The IC does not type commands, so that someone always sees the whole picture.

</details>

5. After `kubectl rollout undo`, why is the incident not yet resolved?

<details>
<summary>Answer</summary>

The cluster is fixed, but the source of truth (Git, Helm values, IaC files) still contains the bad change. The next deploy would bring it back, so the change must also be reverted there.

</details>

6. What makes a postmortem "blameless", and why does it matter?

<details>
<summary>Answer</summary>

It looks for causes in systems and processes, not in people. When people are not blamed, they share what really happened, and the actions fix the system so the same mistake cannot hurt users again.

</details>

## Summary

- An incident hurts users now and needs coordination; severity (SEV1–SEV3) sets how loud the response is. Declare early.
- On-call is shared in a rotation; page only for user-facing symptoms such as SLO burn, and give every page a runbook.
- The life cycle is detect → triage → mitigate → resolve → follow up. Mitigate first, often with a rollback.
- Separate roles as soon as more people join: Incident Commander, Operations, Communications, Scribe.
- Green pods do not mean happy users; ask "what changed?" first, and fix the source of truth after a rollback.
- A blameless postmortem has a timeline, contributing factors and action items with owners and dates.

## Sources and further reading

- [Site Reliability Engineering — Managing Incidents](https://sre.google/sre-book/managing-incidents/) and [Being On-Call](https://sre.google/sre-book/being-on-call/) — the incident roles and a healthy on-call practice.
- [The Site Reliability Workbook — Incident Response](https://sre.google/workbook/incident-response/) and [On-Call](https://sre.google/workbook/on-call/) — practical examples of both.
- [Site Reliability Engineering — Postmortem Culture](https://sre.google/sre-book/postmortem-culture/) and [the Workbook chapter on postmortems](https://sre.google/workbook/postmortem-culture/) — blameless postmortems, with good and bad examples.
- [PagerDuty Incident Response documentation](https://response.pagerduty.com/) — an openly published incident process with role descriptions and checklists.
- [Kubernetes — Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) and [Debug running pods](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/) — rolling updates, `rollout undo` and how to inspect pods and events.

---

[🏠 Start](../START-HERE.md) · [← 21.11 SRE basics: SLIs, SLOs and error budgets](11-sre-basics-slis-slos-and-error-budgets.md) · [📋 Phase 21](README.md) · [21.13 Project: run your service on Kubernetes →](13-project-run-your-service-on-kubernetes.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
