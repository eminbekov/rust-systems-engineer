<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 21.12 Incidents, on-call and postmortems](12-incidents-on-call-and-postmortems.md) · [📋 Phase 21](README.md) · [21.14 Checkpoint →](14-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/21-cloud-native-infrastructure/13-project-run-your-service-on-kubernetes.md) <!-- nav -->

# Lesson 21.13 — Project: run your service on Kubernetes

> **Phase 21 — Cloud-native infrastructure** · Lesson 13 of 14 · ⏱ about 12 hours (6 sittings)

## What you will learn

- How to bring everything from this phase together for one real service: your URL (Uniform Resource Locator) shortener from Phase 16.
- How to split configuration into a ConfigMap and a Secret, keep data in a PersistentVolumeClaim, and probe the service correctly.
- How to package the service as a Helm chart, release new versions and roll back.
- How to build the image and check the chart and the rules in a CI (Continuous Integration) pipeline.
- How to give the service Prometheus metrics, one SLO (Service Level Objective) with a burn-rate alert, a runbook and a README that another engineer can follow.

## Before you start

- You finished [Lesson 21.12 — Incidents, on-call and postmortems](12-incidents-on-call-and-postmortems.md) and all lessons of this phase before it.
- You built the URL shortener in [Lesson 16.15 — Project: a URL shortener service](../16-backend-services/15-project-url-shortener-service.md): `code/p16-shortener`, with PostgreSQL, optional Redis, `/healthz`, `/readyz`, traces, a Dockerfile and integration tests. If you skipped it, use `code/p21-orders` from Lesson 21.9 instead and leave out the database parts; everything else applies unchanged.
- Docker Desktop is running, and `kind`, `kubectl`, `helm`, `jq` and `actionlint` work (Lessons 21.3, 21.6 and 09.12).

## New words

| Word | Meaning |
|---|---|
| **PVC** (PersistentVolumeClaim) | A request for storage in Kubernetes: "give me 1 GiB that survives pod restarts". The cluster finds or creates a **PersistentVolume** for it. kind creates one on the node's disk. |
| **StorageClass** | Describes *how* volumes are created. kind has one called `standard`, which is the default. |
| **resource requests / limits** | How much CPU (Central Processing Unit) and memory a container is guaranteed (requests, used by the scheduler) and may use at most (limits). |
| **release** (Helm) | One installation of a chart in a cluster, with a name and a history of revisions (Lesson 21.6). |
| **source of truth** | The one place where the desired state lives (here: the Git repository). The cluster is always made to match it, never the other way round. |
| **GHCR** (GitHub Container Registry) | GitHub's image registry at `ghcr.io`. A workflow can push to it with its built-in token. |

## The project

### What you will build

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

You take the shortener from Lesson 16.15 and make it run on Kubernetes the way a team would run it in production: configuration and secrets outside the image, data on a volume, two replicas behind a Service, correct probes, a Helm chart with versioned releases, a CI pipeline, metrics, an SLO with a tested alert, a runbook, and a README.

### Contract

**Folders.** Copy the project, so that the Phase 16 version stays as it is: `cp -R code/p16-shortener code/p21-shortener`. Everything of this project lives in `code/p21-shortener`, except the workflow, which GitHub reads only from the repository root:

```text
~/projects/rust/
├── .github/workflows/p21-shortener.yml   ← CI for this project (M5)
└── code/p21-shortener/
    ├── Cargo.toml, src/, migrations/, tests/, Dockerfile   ← from Lesson 16.15
    ├── README.md                    ← how to deploy, release, roll back (M6)
    ├── k8s/
    │   ├── namespace.yaml
    │   └── postgres.yaml            ← PVC, Deployment, Service (M2)
    ├── chart/shortener/             ← Helm chart of the service (M4)
    └── deploy/
        ├── prometheus-values.yaml   ← chart values with the SLO rules (M3)
        ├── rules/slo-rules.yml
        ├── rules/slo-rules-test.yml
        └── runbooks/shortener-slo-fast-burn.md
```

**Names.** Cluster `course-21-13`, namespace `shortener`, image `course-p21-shortener:<version from Cargo.toml>`, Deployment and Service `shortener` (Service port 80 → container port 8080), Helm release `shortener`.

**Configuration.** Non-secret settings in the ConfigMap `shortener-config`: `APP_HOST=0.0.0.0` (the default `127.0.0.1` cannot be reached from the Service, Lesson 21.4), `APP_PORT=8080`, `PUBLIC_BASE_URL`, `RATE_LIMIT_PER_MINUTE`. Secrets only in the Secret `shortener-secrets`: `DATABASE_URL` and `ADMIN_TOKEN`. No secret value is in Git, in the chart's values, in logs or in metric labels.

**Pods.** 2 replicas; readiness probe `GET /readyz`, liveness probe `GET /healthz`; CPU and memory requests and limits; the container runs as non-root (the distroless `nonroot` user from Lesson 16.13). A pod deletion or a node restart loses no data.

**Metrics.** `GET /metrics` in the Prometheus format with `http_requests_total{method,path,status}` and the histogram `http_request_duration_seconds{method,path}` as in Lesson 21.9 (route patterns as labels, never raw paths or codes). `/metrics` is not rate-limited. The pod template has the three `prometheus.io/*` annotations.

**SLO.** "99.5% of redirects (`GET /{code}`) do not fail with 5xx, over 30 days." Recording rules for the error ratio over 5 minutes and 1 hour, and a fast-burn alert (burn rate above 14.4 in both windows, `for: 2m`, `severity: page`), with a `promtool` test that it fires at 10% errors and stays quiet at 1%.

**CI.** One workflow; jobs: Rust gate (`cargo fmt --check`, `cargo clippy --all-targets -- -D warnings`, `cargo test --all-targets`), chart and rules (`helm lint`, `promtool test rules`), image build (push to GHCR only on `main`). `actionlint` reports nothing.

**Quality.** All Phase 16 tests still pass, `cargo clippy` is clean, `cargo fmt` applied. One Git commit per milestone (`M1: …`).

### Step-by-step plan

Keep one terminal for long-running commands (port-forwards) and one for everything else. Create the cluster once: `kind create cluster --name course-21-13`.

#### M1 — An image that fits the cluster

**Goal:** the shortener's image runs as a non-root container, listens on all interfaces when told so, and exposes `/metrics`.

**Guidance:** add the metrics middleware and `/metrics` route from Lesson 21.9 (`metrics` + `metrics-exporter-prometheus` with buckets for the duration histogram). Put `/metrics` next to `/healthz` and `/readyz` in the list of routes that the rate limiter skips. Set the version in `Cargo.toml` to `0.1.0` if it is not, build `course-p21-shortener:0.1.0` with your Dockerfile from Lesson 16.13, and run it once with Docker Compose from Lesson 16.15 (change the image name there) to check it.

<details>
<summary>Hint</summary>

A test like Exercise 2 of Lesson 21.9 proves that a redirect is counted with `path="/{code}"`. Run it before you build the image: the `MatchedPath` of the redirect route is the pattern you wrote in the router.

</details>

**Done when:**

```bash
cargo test
docker build -t course-p21-shortener:0.1.0 .
docker image ls --filter reference=course-p21-shortener --format '{{.Repository}}:{{.Tag}}'
curl -s localhost:8080/metrics | grep -c '^http_requests_total'
```

Expected: all tests pass; the image list shows `course-p21-shortener:0.1.0`; the last command (with the Compose stack running and after one redirect) prints a number greater than 0. Commit: `M1: metrics endpoint and image 0.1.0`.

#### M2 — Plain manifests: database, config, secret, probes

**Goal:** the service and PostgreSQL run in the cluster from plain YAML (YAML Ain't Markup Language) files, and data survives pod deletion.

**Guidance:** first `k8s/namespace.yaml` and `k8s/postgres.yaml`: a PVC `postgres-data` (1Gi, `ReadWriteOnce`), a Deployment `postgres` with the official `postgres` image, its password from a Secret `postgres`, the volume mounted, a readiness probe that runs `pg_isready`, and a Service `postgres` on port 5432. Create the two Secrets with `kubectl create secret generic` (never from a file in Git). Then write the service's Deployment, Service and ConfigMap by hand — you will turn them into a chart in M4, but doing it once by hand shows you what the chart must produce. Load the image with `kind load docker-image`. Migrations run when the service starts (Lesson 16.15), so the first ready pod creates the table.

<details>
<summary>Hint: the PostgreSQL Deployment</summary>

Use `strategy: type: Recreate` for a single database pod, so that the old pod releases the volume before the new one starts. With the `postgres` image, set `PGDATA` to a sub-folder of the mount, for example mount the volume at `/var/lib/postgresql/data` and set `PGDATA=/var/lib/postgresql/data/pgdata`. The password comes from `valueFrom.secretKeyRef` (Lesson 21.5). The service's `DATABASE_URL` then is `postgres://postgres:<password>@postgres.shortener.svc:5432/postgres`.

</details>

<details>
<summary>Hint: resources</summary>

Start small and measure later: for example requests `cpu: 50m, memory: 64Mi` and limits `cpu: 500m, memory: 256Mi` for the service. `kubectl top` needs a metrics server that kind does not install; for now, watch for `OOMKilled` in `kubectl describe pod`.

</details>

**Done when:**

```bash
kubectl -n shortener get pvc
kubectl -n shortener get pods
kubectl -n shortener port-forward service/shortener 8080:80
```

Expected output of the first two commands (names and ages will differ):

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

With the port-forward running, create a link as in Lesson 16.15 (`POST /api/links` with your admin token and `{"url":"https://www.rust-lang.org/learn","code":"learn"}`), then delete **all** pods with `kubectl -n shortener delete pod --all`, wait until they are ready again, restart the port-forward, and run:

```bash
curl -s -o /dev/null -w '%{http_code} %{redirect_url}\n' localhost:8080/learn
```

```text
302 https://www.rust-lang.org/learn
```

The link survived because it lives on the volume. Commit: `M2: plain manifests with PostgreSQL on a volume`.

#### M3 — Prometheus, the SLO and its test

**Goal:** Prometheus scrapes both service pods, and the SLO rules are loaded and tested.

**Guidance:** install the Prometheus chart as in Lesson 21.9, with `deploy/prometheus-values.yaml` containing the SLO groups under `serverFiles.alerting_rules.yml` (Lesson 21.11). Adapt the rules: the SLI (Service Level Indicator) is about `path="/{code}"`; name the series `slo:shortener_redirect_errors:ratio_rate5m` and `..._ratio_rate1h`, the alert `ShortenerRedirectsFastBurn`. Keep a copy of the groups in `deploy/rules/slo-rules.yml` with two tests in `slo-rules-test.yml`: one that fires at 10% errors and one that stays quiet at 1%.

<details>
<summary>Hint</summary>

Copy the two test cases from Lesson 21.11 (Section 6 and Exercise 2) into one file: a test file may contain several entries under `tests:`. Change the series to `http_requests_total{path="/{code}", status="302"}` for good redirects; a redirect's success status is 302, not 200.

</details>

**Done when:**

```bash
curl -s 'localhost:9090/api/v1/targets?state=active' \
  | jq -r '.data.activeTargets[] | select(.labels.namespace == "shortener") | [.labels.pod, .health] | @tsv'
docker run --rm -v "$PWD/deploy/rules":/work -w /work --entrypoint promtool prom/prometheus:v3.16.0 test rules slo-rules-test.yml
```

Expected (with a port-forward to Prometheus on 9090; pod names will differ):

```text
shortener-5d5598c9c7-b9vzq	up
shortener-5d5598c9c7-flttz	up
Processing test file: slo-rules-test.yml
  SUCCESS
```

Commit: `M3: metrics in Prometheus and a tested SLO`.

#### M4 — A Helm chart

**Goal:** the service's Deployment, Service and ConfigMap come from a chart; releases and rollbacks work.

**Guidance:** delete your hand-written Deployment, Service and ConfigMap of the service (keep PostgreSQL and the Secrets: data and secrets live longer than releases). Run `helm create shortener` in `chart/` and adapt it (Lesson 21.6): image repository and tag, `replicaCount: 2`, container port 8080, probes on `/readyz` and `/healthz`, resources, the Prometheus annotations in `podAnnotations`, a `templates/configmap.yaml` built from a `config:` map in `values.yaml`, and `envFrom` that reads the ConfigMap and the existing Secret. Remove what you do not use (ingress, HTTPRoute, the HPA (HorizontalPodAutoscaler, which adds pods under load), the test pod) so that the chart stays readable.

<details>
<summary>Hint: ConfigMap and Secret in the chart</summary>

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

In `templates/deployment.yaml`, above `ports:` of the container:

```yaml
          envFrom:
            - configMapRef:
                name: {{ include "shortener.fullname" . }}-config
            - secretRef:
                name: {{ .Values.existingSecret }}
```

and in `values.yaml` a `config:` map (all values quoted strings) and `existingSecret: shortener-secrets`. The chart never contains the secret values; it only knows the Secret's **name**.

</details>

**Done when:**

```bash
helm lint chart/shortener
helm upgrade --install shortener chart/shortener --namespace shortener --wait
helm list -n shortener
```

Expected (dates will differ; the `APP VERSION` column shows `appVersion` from `Chart.yaml`, here still the default `1.16.0` of `helm create` — set it to your version):

```text
==> Linting chart/shortener
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

```text
NAME     	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART          	APP VERSION
shortener	shortener	1       	2026-11-18 13:58:04.134089 +0500 +05	deployed	shortener-0.1.0	1.16.0     
```

The M2 redirect check still prints `302 https://www.rust-lang.org/learn`. Commit: `M4: Helm chart`.

#### M5 — The CI pipeline

**Goal:** every push and pull request runs the Rust gate, lints the chart, tests the rules and builds the image; `main` also pushes the image to GHCR.

**Guidance:** write `.github/workflows/p21-shortener.yml` in the **repository root** (Lesson 09.12 and 09.13 explain why); the workflow of Lesson 21.7 is your model. Run only for changes in the project: `paths: ["code/p21-shortener/**", ".github/workflows/p21-shortener.yml"]`. Set `defaults: run: working-directory: code/p21-shortener`, and give `Swatinem/rust-cache@v2` `workspaces: code/p21-shortener`. The image job `needs` the other two, has `permissions: packages: write`, logs in to `ghcr.io` with `${{ secrets.GITHUB_TOKEN }}` and pushes only on `main` (pull requests and other branches only build), and uses `docker/build-push-action@v7` with `context: code/p21-shortener`. Your integration tests start containers (Lesson 16.14); GitHub's Ubuntu runners have Docker, so they work there too.

<details>
<summary>Hint: the image job</summary>

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

For a pull request, `github.ref` is `refs/pull/<number>/merge`, so the image is built but not pushed. Image names on GHCR must be lowercase; if your GitHub name has capital letters, write the owner in lowercase by hand. Tag with the commit (`github.sha`), so that every image can be traced back to its code.

</details>

**Done when:** from the repository root, `actionlint` prints nothing and exits with 0:

```bash
actionlint; echo "exit=$?"
```

```text
exit=0
```

If the repository is on GitHub, the run of your next push is green, and on `main` the image appears under **Packages**. If it is not on GitHub, the workflow cannot run (Lesson 09.12); `actionlint` is your check. Commit: `M5: CI pipeline`.

#### M6 — A release drill, a runbook and the README

**Goal:** you can release a new version, check it, roll back, and another engineer can do the same from your README.

**Guidance:** change something visible (for example a log line at start-up), set the version to `0.2.0` in `Cargo.toml` (and, as good practice, `appVersion` in `Chart.yaml`), build and load `course-p21-shortener:0.2.0`, and release it with `helm upgrade ... --set image.tag=0.2.0`. Roll back with `helm rollback shortener 1`. Write the runbook for `ShortenerRedirectsFastBurn` (Lesson 21.12 shows the shape). The README must answer: what the service does; how to create the cluster, the Secrets and PostgreSQL; how to install and upgrade the chart; how to roll back; where the SLO, the alert and the runbook are; and at least three design decisions, for example "PostgreSQL is not part of the chart, because data must outlive releases" or "the chart only names the Secret; secret values never enter Git".

<details>
<summary>Hint</summary>

`helm history` shows every revision. Remember that a ConfigMap change alone does not restart pods: add the `checksum/config` annotation of Lesson 21.6 to the pod template of your chart, so that every configuration change rolls the Deployment.

</details>

**Done when:**

```bash
helm history shortener -n shortener
```

Expected after one upgrade and one rollback (dates will differ):

```text
REVISION	UPDATED                 	STATUS    	CHART          	APP VERSION	DESCRIPTION     
1       	Wed Nov 18 13:58:04 2026	superseded	shortener-0.1.0	1.16.0     	Install complete
2       	Wed Nov 18 13:58:28 2026	superseded	shortener-0.1.0	1.16.0     	Upgrade complete
3       	Wed Nov 18 13:58:29 2026	deployed  	shortener-0.1.0	1.16.0     	Rollback to 1   
```

`kubectl -n shortener get deployment shortener -o jsonpath='{.spec.template.spec.containers[0].image}'` prints `course-p21-shortener:0.1.0` again, and the M2 redirect check still works. Commit: `M6: release drill, runbook and README`. When you are done, `kind delete cluster --name course-21-13`.

> 🔒 **Security:** Check before every commit that `git diff --cached` contains no token, no password and no `DATABASE_URL` with a password. If a secret ever reaches Git, removing it from the next commit is not enough: it stays in the history. Change (rotate) the secret, as [Lesson 10.8 — Secrets management](../10-security-fundamentals/08-secrets-management.md) describes.

> 🧪 **Testing:** The project has five levels of checks now: unit and integration tests (`cargo test`), the chart (`helm lint`), the rules (`promtool test rules`), the workflow (`actionlint`), and a smoke test after each release (the M2 redirect check, or the smoke loop from Lesson 21.12). Put the last one into the README as the first thing to do after `helm upgrade`.

### Stretch goals

1. **Infrastructure as code:** create the namespace, the Secrets (from variables marked `sensitive = true`) and the PostgreSQL objects with OpenTofu and the Kubernetes provider (Lesson 21.8), and install the chart with the Helm provider. Keep the state out of Git.
2. **Latency SLO:** add "99% of redirects finish in at most 100 ms" as a second SLO (Lesson 21.11, Exercise 3), with a test.
3. **Grafana dashboard as code:** provision a dashboard with request rate, error ratio, p95 latency and the remaining error budget from a ConfigMap (Lesson 21.9).
4. **Traces and logs in the cluster:** deploy the collector and Jaeger from Lesson 21.10 and set `OTEL_EXPORTER_OTLP_ENDPOINT` in the ConfigMap; switch the logs to JSON (JavaScript Object Notation).
5. **An end-to-end job in CI:** add a job that creates a kind cluster on the runner (the official `helm/kind-action`), installs PostgreSQL and the chart, and runs the smoke test.
6. **Redis:** run Redis in the cluster as well (official image, no volume needed for a cache) and check that `x-cache: hit` appears on the second redirect.

## Quiz

1. Why is PostgreSQL not part of the service's Helm chart in this project?

<details>
<summary>Answer</summary>

Data must outlive releases. A `helm uninstall` or a broken chart change must never delete the database or its volume. The service and the database also change at different speeds and have different owners.

</details>

2. Why must `APP_HOST` be `0.0.0.0` in the cluster?

<details>
<summary>Answer</summary>

A program that listens on `127.0.0.1` accepts connections only from inside its own container. The Service sends traffic to the pod's IP (Internet Protocol) address, which works only if the program listens on all interfaces.

</details>

3. Which probe calls `/readyz` and which calls `/healthz`, and why not the other way round?

<details>
<summary>Answer</summary>

Readiness calls `/readyz` (is PostgreSQL reachable? if not, take the pod out of the Service). Liveness calls `/healthz` (is the process alive?). If liveness checked the database, a short database outage would make Kubernetes restart all service pods, which helps nobody (Lesson 21.5).

</details>

4. How does the chart get `ADMIN_TOKEN` without the token being in Git?

<details>
<summary>Answer</summary>

The Secret `shortener-secrets` is created separately (`kubectl create secret generic`, or OpenTofu with a sensitive variable). The chart only contains its name (`existingSecret`) and reads it with `envFrom: secretRef`.

</details>

5. Why is the image tagged with the version or the commit, never `latest`?

<details>
<summary>Answer</summary>

An exact tag says which code runs, makes rollbacks possible (`0.1.0` is still there), and lets Kubernetes notice a change in the pod template. `latest` changes under the same name and hides what is deployed.

</details>

6. What is the source of truth after `helm rollback` during an incident, and what must you do next?

<details>
<summary>Answer</summary>

Still the Git repository. The rollback fixed the cluster; now revert or fix the change in Git, so that the next pipeline run or `helm upgrade` does not bring the problem back (Lesson 21.12).

</details>

## Summary

- A production-like deployment separates image, configuration (ConfigMap), secrets (Secret) and data (PVC), and keeps all of it described in Git.
- Probes have different jobs: readiness controls traffic, liveness restarts dead processes.
- A Helm chart turns the manifests into versioned releases with `helm upgrade`, `helm history` and `helm rollback`; it names Secrets but never contains them.
- A CI pipeline checks code, chart, rules and workflow, and builds an image tagged with the commit.
- Metrics, an SLO with a tested burn-rate alert and a runbook make the service operable; a README makes it operable by others.

## Sources and further reading

- [Kubernetes — Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) — PVCs, PersistentVolumes and access modes.
- [Kubernetes — Resource management for pods and containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) — requests and limits.
- [Helm — Chart template guide](https://helm.sh/docs/chart_template_guide/) and [Chart best practices](https://helm.sh/docs/chart_best_practices/) — writing templates and values; the tip about rolling Deployments on ConfigMap changes is in the [tips and tricks](https://helm.sh/docs/howto/charts_tips_and_tricks/).
- [Docker Official Image "postgres"](https://hub.docker.com/_/postgres) — environment variables such as `POSTGRES_PASSWORD` and `PGDATA`.
- [GitHub Docs — Working with the Container registry](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry) and [docker/build-push-action](https://github.com/docker/build-push-action) — pushing images from a workflow.
- [kind — Quick start](https://kind.sigs.k8s.io/docs/user/quick-start/) — loading images and cluster options.

---

[🏠 Start](../START-HERE.md) · [← 21.12 Incidents, on-call and postmortems](12-incidents-on-call-and-postmortems.md) · [📋 Phase 21](README.md) · [21.14 Checkpoint →](14-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
