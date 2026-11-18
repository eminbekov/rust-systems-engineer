<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.28 Cloud-native track: testing and running operators](28-cloud-native-track-testing-and-running-operators.md) · [📋 Phase 22](README.md) · [22.30 Checkpoint: your specialization →](30-checkpoint-your-specialization.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/29-cloud-native-track-project-and-next-steps.md) <!-- nav -->

# Lesson 22.29 — Cloud-native track: project and next steps

> **Phase 22 — Specializations** · Lesson 29 of 30 · ⏱ about 20 hours (10 sittings)

## What you will learn

- How to design an operator that manages several built-in objects (a ConfigMap, a Deployment and a Service) for one
  custom resource.
- How to report useful status from the objects you own, and how to roll out pods when only a ConfigMap changed.
- How to build, test and ship such an operator on your own, milestone by milestone.
- Where to go next on this track: documentation, books, open-source projects in Rust, and communities.

## Before you start

- You finished [Lesson 22.28 — Cloud-native track: testing and running operators](28-cloud-native-track-testing-and-running-operators.md).
  This project reuses everything from Lessons 22.26 to 22.28; keep `code/p22-greeter-operator` open as a reference.
- The kind cluster `course-22-operators` is running. The greeter operator is not needed any more: remove its
  Deployment with `kubectl delete -f k8s/operator.yaml` (in `code/p22-greeter-operator`) to save memory.
- **Not your track?** Skip to [Lesson 22.30 — Checkpoint: your specialization](30-checkpoint-your-specialization.md).

## New words

| Word | Meaning |
|---|---|
| **child object** | An object that an operator creates for a custom resource and owns through an owner reference, here a ConfigMap, a Deployment and a Service. |
| **label selector** | A rule like "all pods with `app.kubernetes.io/name: hello`". A Deployment uses it to find its pods, a Service to find where to send traffic. |
| **recommended labels** | The `app.kubernetes.io/...` label keys that the Kubernetes documentation suggests, such as `name` and `managed-by`. Tools understand them. |
| **rollout** | Replacing the pods of a Deployment step by step with new ones, after the pod template changed. |
| **pod template annotation** | A note in `spec.template.metadata.annotations` of a Deployment. Changing it changes the template, which starts a rollout. |
| **port-forward** | `kubectl port-forward` connects a port on your Mac to a Service or pod in the cluster, for testing without an ingress (the entry point for outside web traffic). |
| **phase** | A short word in a status that sums up the state, here `Pending` or `Ready`. |
| **CRD** (CustomResourceDefinition) | The object that adds a new type to the Kubernetes API (Lesson 22.26). |
| **RBAC** (Role-Based Access Control) | The Kubernetes permission system (Lesson 22.28). |
| **HTML** (HyperText Markup Language) | The text format of web pages, for example `<h1>Hello</h1>`. |

## 1. Designing the operator

Before code, decide **what should exist** for one custom resource. That decision is the whole operator; the rest is the
same machinery as in the greeter.

```text
StaticSite default/hello               (spec: html, replicas)
│
├── ConfigMap  hello-html              data: index.html = spec.html
├── Deployment hello                   replicas = spec.replicas, image nginx:1.30-alpine,
│                                      mounts hello-html at /usr/share/nginx/html
└── Service    hello                   port 80 → the pods of the Deployment

status: readyReplicas (from the Deployment), phase (Pending/Ready), url
```

Four design decisions are worth writing down in your README:

1. **Names are derived** from the StaticSite name (`hello`, `hello-html`). The operator never has to store which
   objects it created; it can always compute them again. This is what makes reconcile level-triggered.
2. **Labels connect things.** Every child gets `app.kubernetes.io/name: <site>` and
   `app.kubernetes.io/managed-by: site-operator`. The Deployment's selector and the Service's selector use them.
   `kubectl get deploy,svc,cm -l app.kubernetes.io/managed-by=site-operator` then shows everything the operator made.
3. **Rolling out on content changes.** Pods read files from the mounted ConfigMap. When only the ConfigMap changes,
   the Deployment does not change, so no new pods start (the kubelet, the agent on each node that runs pods, updates
   mounted files after a delay, but you cannot see when). The standard trick: put a hash of the HTML into a **pod template annotation**. New HTML → new
   hash → new pod template → a rollout.
4. **Status comes from the children.** The Deployment's `status.readyReplicas` tells how many pods serve traffic. The
   operator copies it into the StaticSite status, so a user only ever looks at one object.

> 📏 **Convention:** the Kubernetes API conventions say: users write `spec`, controllers write `status`, and status
> describes what *is*, not what you wish. Read them before you design a bigger API (link in the sources).

## The project

### What you will build

`code/p22-site-operator`: an operator for a custom resource `StaticSite`. Apply this file and you get a running web
site with two pods behind a Service:

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

### Contract

**The custom resource** — group `course.example.com`, version `v1`, kind `StaticSite`, namespaced, short name `site`:

| Field | Type | Rules |
|---|---|---|
| `spec.html` | string | required, 1 to 10,000 characters; becomes `index.html` |
| `spec.replicas` | integer | optional, default `1`, from `0` to `5` |
| `status.readyReplicas` | integer | copied from the Deployment |
| `status.phase` | string | `Ready` when `readyReplicas >= spec.replicas`, otherwise `Pending` |
| `status.url` | string | `http://<name>.<namespace>.svc.cluster.local` |

`kubectl get staticsites` shows the columns `REPLICAS`, `READY` and `PHASE`.

**The children** of a StaticSite `<name>`, all in its namespace, all with an owner reference to it and the labels
`app.kubernetes.io/name: <name>` and `app.kubernetes.io/managed-by: site-operator`:

| Kind | Name | Content |
|---|---|---|
| ConfigMap | `<name>-html` | key `index.html` = `spec.html` |
| Deployment | `<name>` | `spec.replicas` pods with image `nginx:1.30-alpine`, port 80, the ConfigMap mounted read-only at `/usr/share/nginx/html`, pod template annotation `course.example.com/html-hash` = a hash of `spec.html` |
| Service | `<name>` | type ClusterIP, port 80, selector = the labels above |

**The behaviour:** creating, changing or deleting a StaticSite is reflected in its children within seconds; deleting
or editing a child by hand is repaired; deleting the StaticSite deletes all children.

**The project layout and quality bar** (as in every project of the course): `src/lib.rs` (types and pure `desired_*`
functions, with unit tests), `src/main.rs` (reconcile and controller), `src/bin/crdgen.rs`, `tests/e2e.rs`
(`#[ignore]`d), `k8s/` (CRD, example, operator manifests), `Dockerfile`, `README.md`. `cargo fmt`, `cargo clippy
--all-targets` without warnings, `cargo test` green. One Git commit per milestone.

### Step-by-step plan

#### M1 — The custom resource and its CRD

**Goal:** `StaticSiteSpec` and `StaticSiteStatus` in `src/lib.rs`, `crdgen`, and the CRD in the cluster.

**Guidance:** copy `Cargo.toml` from the greeter (rename the package). A default value that is not `false`/`0` needs a
function: `#[serde(default = "one")]` with `fn one() -> i32 { 1 }`. Number limits use `#[schemars(range(min = 0, max = 5))]`.
Write two example manifests: `k8s/hello-site.yaml` (above) and `k8s/too-many.yaml` with `replicas: 9`.

<details>
<summary>Hint</summary>

Three printer columns, two of them integers: if the compiler warns `duplicated attribute` on the second
`printcolumn(..., type_ = "integer", ...)`, write that column in the older raw-JSON form, which kube-derive still accepts:
`printcolumn = r#"{"name":"Ready","type":"integer","jsonPath":".status.readyReplicas"}"#`. The generated CRD is the same.

</details>

**Done when:**

```bash
cargo run -q --bin crdgen > k8s/staticsite-crd.yaml
kubectl apply -f k8s/staticsite-crd.yaml
kubectl apply -f k8s/too-many.yaml
```

```text
customresourcedefinition.apiextensions.k8s.io/staticsites.course.example.com created
The StaticSite "too-many" is invalid: spec.replicas: Invalid value: 9: spec.replicas in body should be less than or equal to 5
```

#### M2 — Pure builders with unit tests

**Goal:** `desired_config_map`, `desired_deployment`, `desired_service` and `status_for(site, ready_replicas)` in
`src/lib.rs`, without any network call.

**Guidance:** build the objects with `k8s-openapi` structs (`Deployment`, `DeploymentSpec`, `PodTemplateSpec`,
`PodSpec`, `Container`, `Volume`, `ConfigMapVolumeSource`, `Service`, `ServiceSpec`, `ServicePort`) and
`..Default::default()` for everything you do not set. Write one helper that builds the `ObjectMeta` shared by all
children (name, namespace, labels, owner reference). For the hash, `std::hash::DefaultHasher` is enough: format
`finish()` as hexadecimal with `format!("{:016x}", …)`.

<details>
<summary>Hint</summary>

Read the field types on docs.rs before you write a struct: in `k8s-openapi` 0.29, `Deployment.spec` is a plain
`DeploymentSpec` (not an `Option`), and so are `DeploymentSpec.selector` and `DeploymentSpec.template`, while
`Service.spec` is still an `Option<ServiceSpec>`. The compiler tells you with `error[E0308]: mismatched types` if
you wrap one in `Some(...)` by mistake. Good tests: replicas copied; the volume points at `<name>-html`; two different
HTML texts give two different annotations; the Service selector equals the labels; `status_for` gives `Pending` and
`Ready` at the right numbers; children carry an owner reference of kind `StaticSite`.

</details>

**Done when:** `cargo test --lib` ends with `test result: ok. 6 passed` (or more), and `cargo clippy --all-targets`
prints no warnings.

#### M3 — Reconcile the three children

**Goal:** an operator that creates and updates all children with server-side apply.

**Guidance:** reconcile applies the ConfigMap, then the Deployment, then the Service, each with
`PatchParams::apply("site-operator").force()`. The controller watches StaticSites and `.owns(...)` all three child
types. Run it with `cargo run` from your Mac first.

<details>
<summary>Hint</summary>

Apply the ConfigMap **before** the Deployment: pods that start before their ConfigMap exists wait in
`ContainerCreating` until it appears. It still works in the other order, only slower and with confusing events.

</details>

**Done when** (ages and IP addresses differ; IP is the Internet Protocol):

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

and, in a second terminal while the port-forward runs, `curl -s localhost:8080` prints
`<h1>Hello from a StaticSite</h1>`.

#### M4 — Status

**Goal:** the StaticSite reports `readyReplicas`, `phase` and `url`.

**Guidance:** the result of the Deployment's apply call *is* the stored Deployment, including its `status`; take
`ready_replicas` from it (it is `None` while no pod is ready). Write the status with `patch_status` and a merge patch,
but only when it differs from `site.status`; derive `PartialEq` on the status type for that.

<details>
<summary>Hint</summary>

The status changes when pods become ready, and that is a change of the **Deployment**. Because the controller
`.owns()` Deployments, every such change reconciles the StaticSite again, and the status catches up without any timer.

</details>

**Done when:**

```bash
kubectl wait --for=jsonpath='{.status.phase}'=Ready staticsite/hello --timeout=180s
kubectl get staticsites
```

```text
staticsite.course.example.com/hello condition met
NAME    REPLICAS   READY   PHASE
hello   2          2       Ready
```

#### M5 — Changes and self-healing

**Goal:** new HTML reaches the browser, a deleted Deployment comes back, scaling works.

**Guidance:** this milestone is mostly testing what M2–M4 built. If the HTML does not change, check the hash
annotation in `kubectl get deployment hello -o yaml`.

<details>
<summary>Hint</summary>

`kubectl rollout status deployment/hello` waits until a rollout is finished; run it right after you patch the HTML.

</details>

**Done when:**

```bash
kubectl patch staticsite hello --type merge -p '{"spec":{"html":"<h1>Version 2</h1>\n"}}'
kubectl rollout status deployment/hello --timeout=120s
```

ends with `deployment "hello" successfully rolled out`, and `curl -s localhost:8080` (through a new port-forward)
prints `<h1>Version 2</h1>`. Then:

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

#### M6 — Ship it

**Goal:** the operator runs in the cluster in namespace `site-system`, with an e2e test and a README.

**Guidance:** an ignored e2e test creates a StaticSite `e2e-site`, waits (with `await_condition` and a timeout) until
its phase is `Ready`, deletes it and waits until the Deployment `e2e-site` is gone. Then a `Dockerfile` like the
greeter's, an image `course-site-operator:0.1.0` loaded with `kind load docker-image`, and `k8s/operator.yaml` with a
Namespace, a ServiceAccount, a ClusterRole, a ClusterRoleBinding and a Deployment.

<details>
<summary>Hint</summary>

List the API calls, as in Exercise 3 of Lesson 22.28: StaticSites are only watched (`list`, `watch`), their status is
patched (`staticsites/status`: `patch`), and each child kind is watched and applied (`list`, `watch`, `create`,
`patch`). Deployments are in the API group `apps`; ConfigMaps and Services in the core group `""`. One rule can list
several resources: `resources: ["configmaps", "services"]`. There is no finalizer in this project, so StaticSites need
no `patch`.

</details>

**Done when:** with the operator running only in the cluster (stop your `cargo run`):

- `kubectl get pods -n site-system` shows one `Running` pod;
- `cargo test --test e2e -- --ignored` reports `1 passed`;
- `kubectl logs deployment/site-operator -n site-system | grep -ci forbidden` prints `0`;
- after `kubectl delete staticsite hello`, `kubectl get deploy,svc,cm -l app.kubernetes.io/managed-by=site-operator`
  prints `No resources found in default namespace.`;
- `README.md` explains what the operator does, how to install it, the contract table, and how to run both kinds of tests.

### Stretch goals

- **Conditions.** Replace `phase` with a list of standard conditions (`type: Ready`, `status: "True"`, `reason`,
  `message`, `lastTransitionTime`), as the API conventions describe.
- **Events.** Publish Kubernetes Events (`kube::runtime::events::Recorder`) such as "Rolled out new HTML", so
  `kubectl describe staticsite hello` tells the story.
- **A safer web server.** Run nginx as non-root with a read-only root file system (it then needs writable `emptyDir`
  volumes for its cache and its PID file, the file with its process ID) and add a readiness probe on port 80.
- **Metrics.** Count reconciles and errors and expose them on `/metrics` for Prometheus (Lesson 21.9).
- **Validation with CEL** (Common Expression Language, the rule language of the API server). Add a rule that `html` must contain `<h1>`, using `#[x_kube(validation = ...)]`, and see the
  API server reject a page without a heading.

## Next steps on the cloud-native track

You can now build, test and run a real operator. Here is how to keep going.

**Read** (in this order):

1. The [kube.rs documentation](https://kube.rs/) to the end: the controller guide also covers optimization
   (watching less, caching less), availability (several replicas with leader election) and observability.
2. The Kubernetes [API conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md):
   how spec, status and conditions should look in any API you design.
3. The CNCF [Operator White Paper](https://tag-app-delivery.cncf.io/whitepapers/operator/): what makes an operator good,
   including security and best practices.
4. Books: *Programming Kubernetes* by Michael Hausenblas and Stefan Schimanski (how controllers and the API machinery
   work inside) and *Kubernetes Patterns* by Bilgin Ibryam and Roland Huß (the operator and controller patterns among
   many others). Their examples are not in Rust, but the ideas carry over unchanged.

**Read code** (Lesson 24.3 shows how to read a large codebase):

| Project | What it is | Why read it |
|---|---|---|
| [kube-rs/kube](https://github.com/kube-rs/kube) | the crates you used | `kube-runtime/src/controller` and `finalizer.rs` are short and well commented |
| [kube-rs/controller-rs](https://github.com/kube-rs/controller-rs) | the reference operator | metrics, tracing, events, tests and Helm chart in one small repo |
| [stackabletech](https://github.com/stackabletech) | many production operators in Rust (for Kafka, Spark, ZooKeeper, …) | real-world operators built on kube-rs |
| [linkerd/linkerd2-proxy](https://github.com/linkerd/linkerd2-proxy) | the data-plane proxy of the Linkerd service mesh, in Rust | high-performance networking inside clusters |
| [kubewarden/policy-server](https://github.com/kubewarden/policy-server) | a Kubernetes policy engine in Rust that runs policies as WebAssembly | admission control, another way to extend Kubernetes |
| [youki-dev/youki](https://github.com/youki-dev/youki) | a container runtime in Rust | what happens below the pod: namespaces and cgroups from Phase 13 |
| [project-akri/akri](https://github.com/project-akri/akri) | discovers devices at the edge and exposes them to the cluster | a CNCF project with Rust controllers and agents |

**Contribute** (Lessons 24.4 and 24.5 teach the process): start with the issues labelled
[good first issue in kube-rs](https://github.com/kube-rs/kube/issues?q=is%3Aissue+is%3Aopen+label%3A%22good+first+issue%22),
documentation fixes in kube.rs, or examples. Read the project's
[CONTRIBUTING.md](https://github.com/kube-rs/kube/blob/main/CONTRIBUTING.md) first; it also explains the project's
rules for AI-generated contributions.

**Talk to people:** the `#kube` channel on the [Tokio Discord](https://discord.gg/tokio) (linked from kube.rs), the
[Kubernetes Slack](https://slack.k8s.io/) and the [CNCF Slack](https://slack.cncf.io/). Kubernetes SIGs (special
interest groups) such as SIG API Machinery hold open video meetings.

**Build next:** an operator for something you run yourself, for example a backup schedule for the PostgreSQL from
Phase 16 (a CronJob, the object that runs a job on a schedule, plus a status with the last backup time), or one of the capstones in Phase 23 deployed with its
own operator.

## Quiz

1. Why does the operator derive child names from the StaticSite name instead of storing the names it generated?

<details>
<summary>Answer</summary>

Because reconcile must be able to rebuild the full desired state from the StaticSite alone, after a restart and
without memory of earlier runs. Derived names are always known; stored ones could be lost or out of date.

</details>

2. You change only `spec.html`. Without the hash annotation, what happens to the running pods, and why?

<details>
<summary>Answer</summary>

The ConfigMap changes, but the Deployment's pod template does not, so no rollout starts. The pods keep running and see
the new file only when the kubelet refreshes the mounted ConfigMap, after an unpredictable delay. The annotation makes
the template change, which starts a rollout at once.

</details>

3. How does the StaticSite status learn that a pod became ready, without a timer?

<details>
<summary>Answer</summary>

A pod becoming ready changes the Deployment's status. The controller `.owns()` Deployments, so that change triggers a
reconcile of the owning StaticSite, which reads `readyReplicas` from the Deployment and updates its own status.

</details>

4. Which RBAC verbs does the site operator need on Deployments, and why each?

<details>
<summary>Answer</summary>

`list` and `watch` for the `.owns()` watch, `patch` for server-side apply, and `create` because applying a Deployment
that does not exist yet creates it. It never deletes (the garbage collector does) and never does a single `get`.

</details>

## Summary

- Design first: write down what should exist for one custom resource, with derived names and shared labels. The
  operator is that design plus the reconcile machinery you already know.
- One operator can own several child kinds; `.owns()` each of them so that changes and deletions are repaired at once.
- Status comes from the children; write it only when it changed.
- A hash of the content in a pod template annotation turns a ConfigMap change into a rollout.
- Ship with unit tests, an ignored e2e test, a least-privilege ClusterRole and a README.
- Next: the kube.rs guide, the API conventions, the Operator White Paper, reading kube-rs and other Rust projects, and
  your first contribution.

## Sources and further reading

- [kube.rs — Optimization](https://kube.rs/controllers/optimization/) and [kube.rs — Availability](https://kube.rs/controllers/availability/) — the next chapters of the controller guide.
- [Kubernetes — API conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) — spec, status and conditions.
- [CNCF — Operator White Paper](https://tag-app-delivery.cncf.io/whitepapers/operator/) — what a good operator does.
- [Kubernetes — Extending Kubernetes](https://kubernetes.io/docs/concepts/extend-kubernetes/) — every way to extend a cluster, operators among them.
- [Kubernetes — Use Port Forwarding to Access Applications in a Cluster](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/) — `kubectl port-forward` in detail.
- [docs.rs — k8s-openapi Deployment](https://docs.rs/k8s-openapi/0.29.0/k8s_openapi/api/apps/v1/struct.Deployment.html) — the field types you build in M2.
- [Docker Hub — nginx](https://hub.docker.com/_/nginx) — the official image the sites run on.

---

[🏠 Start](../START-HERE.md) · [← 22.28 Cloud-native track: testing and running operators](28-cloud-native-track-testing-and-running-operators.md) · [📋 Phase 22](README.md) · [22.30 Checkpoint: your specialization →](30-checkpoint-your-specialization.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
