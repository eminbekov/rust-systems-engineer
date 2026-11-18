<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 21.5 ConfigMaps, Secrets and health probes](05-configmaps-secrets-and-probes.md) · [📋 Phase 21](README.md) · [21.7 CI/CD: from commit to production →](07-ci-cd-from-commit-to-production.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/21-cloud-native-infrastructure/06-helm-and-kustomize.md) <!-- nav -->

# Lesson 21.6 — Packaging with Helm and Kustomize

> **Phase 21 — Cloud-native infrastructure** · Lesson 6 of 14 · ⏱ about 3 hours (2 sittings)

## What you will learn

- Why plain manifest files become hard to manage when one service runs in several environments.
- How Helm works: charts, values, templates, releases and revisions. You will write a small chart for your service.
- How to check a chart before installing it (`helm lint`, `helm template`), and how to install, upgrade, roll back and remove a release.
- How Kustomize works: a base plus overlays, with patches instead of templates, built into kubectl.
- When to choose which tool.

## Before you start

- You finished [Lesson 21.5 — ConfigMaps, Secrets and health probes](05-configmaps-secrets-and-probes.md) and have `code/p21-hello` at version 0.3.0 with its `k8s/` folder and `admin-token.txt`.
- You remember the recommended labels `app.kubernetes.io/*` from [Lesson 21.2](02-kubernetes-concepts.md).

## New words

| Word | Meaning |
|---|---|
| **Helm** | A package manager for Kubernetes. It installs a set of manifests as one unit and remembers each version of it. |
| **chart** | A Helm package: a folder with `Chart.yaml`, `values.yaml` and templates of manifests. |
| **values** | The settings of a chart (`values.yaml`), which you can override at install time with `--set` or `-f file.yaml`. |
| **template** | A manifest with placeholders such as `{{ .Values.replicaCount }}` that Helm fills in. Helm uses the Go template language. |
| **release** | One installed copy of a chart in a cluster, with a name (for example `hello`). One chart can be installed many times under different names. |
| **revision** (Helm) | A numbered version of a release. Every install, upgrade and rollback creates a new one. |
| **Kustomize** | A tool, built into kubectl, that builds manifests from a *base* plus *overlays* that patch it. No templates. |
| **base** | In Kustomize, the shared set of manifests that every environment uses. |
| **overlay** | In Kustomize, a folder for one environment that points to the base and lists the changes for that environment. |
| **generator** | A Kustomize feature that creates a ConfigMap or Secret and adds a hash of its content to its name. |
| **environment** | One place where the service runs with its own settings, for example *dev*, *staging* or *production*. |
| **JSON Schema** (JSON: JavaScript Object Notation) | A standard way to describe which fields a JSON or YAML document may have and of which types; Helm uses it to check values. |

## 1. The problem with copies

After Lesson 21.5 your service has three manifest files in YAML (YAML Ain't Markup Language). Now you need it in two environments: *dev* with 1 replica and the greeting `privet`, *production* with 3 replicas and `salam`. The obvious solution, two copies of the folder, has known problems:

- A fix (for example a new probe) must be made in every copy, and sooner or later one copy is forgotten.
- A reviewer cannot easily see what is *different* between dev and production.
- "What exactly is installed in the cluster, and what was installed before?" has no answer; `kubectl apply` keeps no history of the whole set.

Two tools solve this in two different ways:

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

Helm fills placeholders in templates; Kustomize starts from complete files and changes some fields. Both end with ordinary manifests that the API (Application Programming Interface) server receives.

## 2. Helm: install and first look

Install Helm with Homebrew and check it:

```bash
brew install helm
helm version --short
```

```text
v4.3.2+g8346aaa
```

This lesson uses Helm 4. Helm talks to the cluster of your current kubectl context, with the same kubeconfig. Unlike old versions, it has no component inside the cluster; it stores the history of each release as Secrets in the release's namespace.

A chart is a folder. Create this layout inside `code/p21-hello` (the chart is named `hello`):

```text
chart/hello/
├── Chart.yaml          ← name and versions of the chart
├── values.yaml         ← default settings
└── templates/
    ├── _helpers.tpl    ← named pieces of templates (no manifest)
    ├── configmap.yaml
    ├── deployment.yaml
    └── service.yaml
```

> 💡 **Tip:** `helm create <name>` generates a complete example chart with many more files (an Ingress, an autoscaler, a service account, tests). It is a good reference, but for learning it is clearer to write the few files below yourself.

`chart/hello/Chart.yaml`:

```yaml
apiVersion: v2
name: hello
description: The p21-hello service from the Rust course
type: application
version: 0.1.0
appVersion: "0.3.0"
```

`version` is the version of the **chart** (the packaging); `appVersion` is the version of the **program** inside. They change independently: a new probe in the chart is a new chart version with the same program.

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

The Secret stays outside the chart, created with `kubectl create secret` as in Lesson 21.5: secret values do not belong in `values.yaml`, which lives in Git.

## 3. Templates

`chart/hello/templates/_helpers.tpl` defines two reusable pieces:

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

`chart/hello/templates/configmap.yaml` and `service.yaml`:

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

`chart/hello/templates/deployment.yaml` is the Deployment of Lesson 21.5 with placeholders:

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

**What happens:**

1. Everything between `{{` and `}}` is template code; the rest is copied as it is. `{{-` and `-}}` also remove the white space (and line breaks) next to them.
2. `.Values` is the merged values (defaults from `values.yaml`, then `-f` files, then `--set`). `.Release.Name` is the name you give at install time, so two releases of the same chart do not clash. `.Chart` is `Chart.yaml`.
3. `define "hello.labels"` names a piece of template; `include "hello.labels" .` inserts it, and `| nindent 4` puts it on a new line indented by 4 spaces. The `.` passes the current data to the piece. The pipe `|` sends the result of the left side into the function on the right, like a shell pipe.
4. `| quote` puts a value in double quotes. ConfigMap values must be strings, and without quotes a value like `3000` (a number) or `yes` could be read as another type.
5. `{{ .Values.image.tag | default .Chart.AppVersion }}` uses the tag from the values, or `appVersion` when it is empty. So by default the image version follows the chart's `appVersion`.
6. `toYaml .Values.resources | nindent 12` writes a whole block of values as YAML, so users can change `resources` without the template knowing every field.
7. `checksum/config` is the trick from Lesson 21.5: Helm renders the ConfigMap template, computes its SHA-256 (Secure Hash Algorithm, 256 bits) checksum, and writes it into the pod template. A new greeting gives a new checksum, the pod template changes, and the Deployment rolls the pods automatically.
8. The selector uses only `hello.selectorLabels` (name and instance), which never change. The pod labels also contain the version, which changes with every release.

## 4. Check the chart before you install it

`helm lint` finds mistakes in the chart's structure; `helm template` renders the manifests on your Mac without any cluster:

```bash
helm lint chart/hello
helm template hello chart/hello
```

```text
==> Linting chart/hello
[INFO] Chart.yaml: icon is recommended

1 chart(s) linted, 0 chart(s) failed
```

The rendered ConfigMap and the start of the Deployment (shortened):

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

Read the rendered output whenever you change a template: most chart bugs are visible here, as wrong indentation or a missing value.

> 🧪 **Testing:** `helm template` output is plain YAML, so you can test a chart without a cluster: pipe it into `kubectl apply --dry-run=server -f -` against a test cluster, check it with a policy tool, or compare it with a stored copy (a snapshot test, Lesson 09.5). Exercise 2 shows Helm's own test feature.

## 5. Install, upgrade, roll back

Create the cluster and the Secret, load the image (as in Lesson 21.5), then install the chart as the release `hello`:

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

`--wait` makes Helm wait until the new objects are ready, so a `deployed` status means the pods really run. The pods carry the new labels:

```bash
kubectl get pods -l app.kubernetes.io/instance=hello
```

```text
NAME                     READY   STATUS    RESTARTS   AGE
hello-76cf754449-2zp79   1/1     Running   0          6s
hello-76cf754449-k6gk2   1/1     Running   0          6s
hello-76cf754449-lw55n   1/1     Running   0          6s
```

Now change two settings for this release only, without editing any file:

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

The greeting changed the ConfigMap, the checksum changed the pod template, and the Deployment rolled the pods with no `rollout restart`: a request now prints `privet from hello-59f549fc88-df4jd (version 0.3.0)`. Helm remembers every revision:

```bash
helm history hello
```

```text
REVISION	UPDATED                 	STATUS    	CHART      	APP VERSION	DESCRIPTION     
1       	Wed Nov 18 13:57:45 2026	superseded	hello-0.1.0	0.3.0      	Install complete
2       	Wed Nov 18 13:58:00 2026	deployed  	hello-0.1.0	0.3.0      	Upgrade complete
```

Roll back to revision 1:

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

A rollback is a **new** revision (3) with the values of revision 1; history is never rewritten. Unlike `kubectl rollout undo` (Lesson 21.4), it restores the whole release: ConfigMap, Service and Deployment together. `helm get values hello --all` prints all values the release runs with, including the defaults.

> 📏 **Convention:** in scripts and CI (Continuous Integration) use `helm upgrade --install <release> <chart> --wait`: it installs the release if it does not exist and upgrades it otherwise, so the same command works the first time and every time after. For an environment, keep the overrides in a file in Git (`-f values-dev.yaml`) instead of long `--set` lists.

When you are done with the release, `helm uninstall hello` removes every object it created (the Secret you made by hand stays).

## 6. Kustomize: a base and an overlay

Kustomize needs no new tool: `kubectl kustomize` renders and `kubectl apply -k` applies. Create this layout inside `code/p21-hello`, copying `deployment.yaml` and `service.yaml` from `k8s/` into `kustomize/base/` (not `configmap.yaml`: a generator replaces it):

```text
kustomize/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml     ← copied from k8s/
│   └── service.yaml        ← copied from k8s/
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

**What happens:**

1. The base lists its files under `resources` and **generates** the ConfigMap `hello-config` from literal values.
2. The overlay takes the whole base (`../../base`) and changes it: every object goes into the namespace `hello-dev`; the Deployment `hello` gets 1 replica; the generated ConfigMap is **merged** with a new `GREETING`; and every image named `course-p21-hello` gets the tag `0.3.0` (this is how each environment chooses its version).
3. The base files stay complete, valid manifests. You can still `kubectl apply -f` them; Kustomize only changes fields.

Render the overlay to see the result (shortened):

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

The generator added a **hash of the content** to the ConfigMap's name (`-bg8b5cd5bg`) and changed the reference in the Deployment to match. Apply it to the cluster (the namespace needs its own copy of the Secret):

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

A request in that namespace prints `privet from hello-7d7994f67b-h84fv (version 0.3.0)`. Now change `GREETING=privet` to `GREETING=zdravstvuy` in the overlay and apply again:

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

New content, new name, new reference in the pod template: the Deployment rolled the pod by itself, the same effect as Helm's checksum. The old ConfigMap is left behind, unused, which makes going back to the previous version safe; clean such leftovers from time to time. Remove the dev environment with `kubectl delete namespace hello-dev`.

## 7. Helm or Kustomize?

| | Helm | Kustomize |
|---|---|---|
| Idea | Templates + values | Base + patches |
| Extra tool | Yes (`helm`) | No (built into kubectl) |
| Release history and rollback | Yes (`helm history`, `helm rollback`) | No (use Git and `kubectl apply`) |
| Sharing with others | Charts in repositories (also as OCI, Open Container Initiative, images in registries) | Folders in Git |
| Readability | Templates can become hard to read | Files stay plain YAML |
| Typical use | Installing third-party software; one chart for many teams | Your own services with a few environments |

Many teams use both: they install third-party software (Prometheus, for example, which you meet in Lesson 21.9) with its official Helm chart, and manage their own services with Kustomize or with a small chart of their own. OCI is the image standard from Lesson 21.4; Helm can store charts in the same registries as images.

## 8. Clean up

```bash
helm uninstall hello
kind delete cluster --name course-21-06
docker image rm course-p21-hello:0.3.0
```

Keep the `chart/` and `kustomize/` folders: Lesson 21.7 deploys the chart from a pipeline.

## Common mistakes

**Changing the selector labels of an installed release.** The selector of a Deployment cannot be changed after it is created. If you add a label to `hello.selectorLabels` and upgrade, the API server refuses:

```text
Error: UPGRADE FAILED: server-side apply failed for object default/hello apps/v1, Kind=Deployment: Deployment.apps "hello" is invalid: spec.selector: Invalid value: {"matchLabels":{"app.kubernetes.io/component":"web","app.kubernetes.io/instance":"hello","app.kubernetes.io/name":"hello"}}: field is immutable
```

The failed upgrade stays in `helm history` as a `failed` revision, and the running version is not touched. Fix: never change selector labels; put new labels into `hello.labels` only. If you really must, delete the Deployment and install again (with a short outage).

**Any pod with the selector labels gets traffic.** A Service selects *pods*, not "pods of my Deployment". In an early version of Exercise 2, the test pod got the full `hello.labels`, so the Service also sent requests to the test pod itself, which listens on no port. The EndpointSlices showed the test pod's address as a fourth endpoint, and some requests failed:

```text
NAME          ADDRESSTYPE   PORTS     ENDPOINTS                             AGE
hello-jvldb   IPv4          8080      10.244.0.16,10.244.0.14,10.244.0.15   82s
hello-qv956   IPv4          <unset>   10.244.0.17                           1s
```

```text
wget: download timed out
```

Fix: give helper pods (tests, jobs, debugging pods) labels that do not match any Service selector.

**Secrets in `values.yaml`.** `values.yaml` is in Git, and `helm get values` shows the values to anyone who may read the release. Pass only the *name* of a Secret, as this chart does, and create the Secret separately.

## Try it yourself

1. Render the chart with another release name: `helm template shop chart/hello`. Which object names change?
2. Create `values-dev.yaml` with `greeting: privet` and `replicaCount: 1`, and render with `-f values-dev.yaml`. Compare with Exercise 1.
3. Install the chart twice in one cluster, as releases `hello` and `shop` (both can use the Secret `hello-admin`). Look at the object names and the selector labels: do the two releases clash?

## Exercises

### Exercise 1 — Render with overrides (easy)

Without a cluster, render the chart with `greeting=privet` and `replicaCount=1`, and show only the lines with `GREETING`, `replicas:` and the checksum. Compare the checksum with the default one from Section 4.

<details>
<summary>Hint</summary>

`helm template hello chart/hello --set ... --set ... | grep -E 'GREETING|replicas:|checksum'`.

</details>

<details>
<summary>Answer</summary>

```bash
helm template hello chart/hello --set greeting=privet --set replicaCount=1 | grep -E 'GREETING|replicas:|checksum'
```

```text
  GREETING: "privet"
  replicas: 1
        # A new checksum after every config change starts a rolling update.
        checksum/config: 164c134ea9938e8ee94c0f5dd2ce33cd8e033e7f24abb3f0cd4067342f4f231e
```

The checksum differs from the default (`95cd6335...`) because the ConfigMap's content changed; `replicaCount` is not in the ConfigMap, so alone it would not change the checksum. The same output comes from a file: `-f values-dev.yaml` with the two values.

</details>

### Exercise 2 — A Helm test (medium)

Add a **Helm test**: a pod in `templates/tests/test-connection.yaml`, marked with the annotation `"helm.sh/hook": test`, that fetches `http://<release name>/` and succeeds only if the answer starts with the configured greeting. Run it with `helm test hello --logs`, then upgrade with `--set greeting=privet` and run it again. Finally make it fail on purpose with `kubectl scale deployment hello --replicas=0`.

<details>
<summary>Hint</summary>

A test pod has `restartPolicy: Never` and one container whose command exits with `0` for success. Use `busybox:1.37` and `sh -c "wget -qO- -T 5 http://... | grep '... from'"`. Read "Common mistakes" before you choose the pod's labels. The annotation `"helm.sh/hook-delete-policy": before-hook-creation` deletes the old test pod before the next run.

</details>

<details>
<summary>Answer</summary>

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

The end of the output:

```text
TEST SUITE:     hello-test
Last Started:   Wed Nov 18 14:04:32 2026
Last Completed: Wed Nov 18 14:04:34 2026
Phase:          Succeeded

POD LOGS: hello-test (wget)
salam from hello-76cf754449-q27jh (version 0.3.0)
```

After `helm upgrade hello chart/hello --set greeting=privet --wait`, the test checks for `privet from` and passes again (`Phase: Succeeded`, `privet from hello-59f549fc88-mgbk8 (version 0.3.0)`). With 0 replicas it fails, and `helm test` exits with a non-zero code:

```text
Phase:          Failed

POD LOGS: hello-test (wget)
wget: can't connect to remote host (10.96.1.205): Connection refused

Error: resource Pod/default/hello-test not ready. status: Failed, message: pod hello-test failed
```

Why it works: Helm creates the pods with the `test` hook only when you run `helm test`, waits for them and reports their result. Because the expected text comes from `.Values.greeting`, the test always checks the configuration of the current revision. Run it in CI right after `helm upgrade` (Lesson 21.7). Note that `helm uninstall` does not delete hook pods; remove a leftover with `kubectl delete pod hello-test`.

</details>

### Exercise 3 — Validate the values (hard)

A user runs `helm upgrade hello chart/hello --set replicaCount=0` or `--set replicaCount=two`. Make Helm refuse both, *before* anything reaches the cluster, by adding `chart/hello/values.schema.json`: `replicaCount` must be an integer of at least 1, `greeting` a non-empty string, `startupDelayMs` an integer of at least 0.

<details>
<summary>Hint</summary>

Helm checks the merged values against `values.schema.json` (a JSON Schema) in `lint`, `template`, `install` and `upgrade`. Use `"type": "integer"`, `"minimum"`, `"minLength"` and `"required"`.

</details>

<details>
<summary>Answer</summary>

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

`helm lint chart/hello --set replicaCount=0` reports the same problem as an `[ERROR]`, so CI can catch it before any deployment. With valid values, `helm lint chart/hello` passes as before.

Why it works: the schema turns `values.yaml` into a checked contract, like types for a function's parameters. Without it, `replicaCount: two` would reach the API server and fail there with a less clear message, or, worse, `0` would be accepted and quietly stop the service.

</details>

## Quiz

1. What is the difference between a chart, a release and a revision?

<details>
<summary>Answer</summary>

A chart is the package (templates and default values). A release is one installed copy of a chart in a cluster, with a name. A revision is one numbered version of a release; each install, upgrade and rollback adds one.

</details>

2. In which order are values merged, and which wins?

<details>
<summary>Answer</summary>

First `values.yaml` of the chart, then files given with `-f` (in order), then `--set`. Later sources override earlier ones, so `--set` wins.

</details>

3. Why does the chart put a checksum of the ConfigMap into the pod template?

<details>
<summary>Answer</summary>

Because pods read the ConfigMap only at start. When the configuration changes, the checksum changes, the pod template changes, and the Deployment rolls the pods automatically.

</details>

4. What does `helm rollback hello 1` create, and how does it differ from `kubectl rollout undo`?

<details>
<summary>Answer</summary>

It creates a new revision with the configuration of revision 1. It restores all objects of the release (ConfigMap, Service, Deployment), while `kubectl rollout undo` only changes a Deployment's pod template.

</details>

5. What is a base and what is an overlay in Kustomize?

<details>
<summary>Answer</summary>

The base is the shared, complete set of manifests. An overlay is a folder for one environment that refers to the base and lists changes (namespace, replicas, images, merged ConfigMap values, patches).

</details>

6. Why does a ConfigMap made by a Kustomize generator have a name like `hello-config-bg8b5cd5bg`?

<details>
<summary>Answer</summary>

The suffix is a hash of its content. A content change gives a new name, Kustomize updates the references in the pod template, and the Deployment rolls the pods; the old ConfigMap remains for a safe way back.

</details>

7. Give one reason to choose Helm and one to choose Kustomize.

<details>
<summary>Answer</summary>

Helm: release history and rollback, or installing third-party software from a chart repository. Kustomize: no extra tool and no template language; the files stay plain YAML, which suits your own services with a few environments.

</details>

## Summary

- Copies of manifests per environment drift apart. Helm (templates + values) and Kustomize (base + overlays) both produce ordinary manifests from one source.
- A Helm chart has `Chart.yaml` (chart `version` and program `appVersion`), `values.yaml` and templates; `.Values`, `.Release` and `.Chart` fill the placeholders; `include`, `nindent`, `quote`, `default` and `toYaml` do most of the work.
- Check with `helm lint` and `helm template`; deploy with `helm upgrade --install --wait`; inspect with `helm history` and `helm get values`; go back with `helm rollback` (a new revision).
- A checksum annotation (Helm) or a hashed generator name (Kustomize) makes configuration changes roll the pods automatically.
- Never change selector labels; keep helper pods' labels away from Service selectors; keep secret values out of `values.yaml`.
- `values.schema.json` makes Helm reject bad values before they reach the cluster.

## Sources and further reading

- [Helm — Quickstart Guide](https://helm.sh/docs/intro/quickstart/) — installing Helm and the first commands.
- [Helm — Charts](https://helm.sh/docs/topics/charts/) — the chart layout, `Chart.yaml` and `values.schema.json`.
- [Helm — Chart Template Guide](https://helm.sh/docs/chart_template_guide/getting_started/) — the template language step by step.
- [Helm — Chart Tips and Tricks](https://helm.sh/docs/howto/charts_tips_and_tricks/) — includes the checksum annotation for automatic rollouts.
- [Helm — Chart Tests](https://helm.sh/docs/topics/chart_tests/) — writing and running `helm test`.
- [Helm — Labels and Annotations](https://helm.sh/docs/chart_best_practices/labels/) — which labels a chart should set.
- [Kubernetes documentation — Declarative Management of Kubernetes Objects Using Kustomize](https://kubernetes.io/docs/tasks/manage-kubernetes-objects/kustomization/) — bases, overlays and generators.
- [Kustomize reference — kustomization](https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/) — every field of `kustomization.yaml`.

---

[🏠 Start](../START-HERE.md) · [← 21.5 ConfigMaps, Secrets and health probes](05-configmaps-secrets-and-probes.md) · [📋 Phase 21](README.md) · [21.7 CI/CD: from commit to production →](07-ci-cd-from-commit-to-production.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
