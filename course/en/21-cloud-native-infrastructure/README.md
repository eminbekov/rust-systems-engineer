<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 20.11 Checkpoint](../20-reliability-and-hardening/11-checkpoint.md) · [21.1 What cloud native means →](01-what-cloud-native-means.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/21-cloud-native-infrastructure/README.md) <!-- nav -->

# Phase 21 — Cloud-native infrastructure

> **Stage 5 — Mastery and career** · ⏱ about 4–5 weeks at ~10 hours per week

## What this phase is about

Until now you have run your services on one machine: with `cargo run`, with Docker, with Docker Compose. Real systems run many copies of many services on many machines, and they change every day. Doing that by hand does not scale: someone has to place the containers, restart the ones that crash, replace them during an update and notice when something goes wrong.

This phase teaches the tools and habits that let machines do this work. You start with the idea behind them all — a *desired state* and a *reconciliation loop* — and then run a real Kubernetes cluster on your Mac with kind. You deploy your own Rust service, configure it with ConfigMaps and Secrets, let Kubernetes check its health, package it with Helm and Kustomize, and build a pipeline that tests, builds and deploys every commit.

The second half is about knowing what your system does in production: infrastructure as code with OpenTofu, metrics with Prometheus and Grafana, logs and traces through an OpenTelemetry collector, and the basics of SRE (site reliability engineering): SLIs (service level indicators), SLOs (service level objectives), error budgets, incidents and postmortems.

Everything runs locally and for free, in Docker on your Mac. You never need a cloud account.

## What you will be able to do

- [ ] Explain desired state, controllers and the reconciliation loop, and why services are treated as cattle, not pets.
- [ ] Create and delete a local Kubernetes cluster with kind, and use kubectl to inspect, debug and change what runs in it.
- [ ] Deploy a Rust service with a Deployment and a Service, with resource limits, a strict security context, probes and a graceful shutdown.
- [ ] Configure a service with ConfigMaps and Secrets, and change the configuration without downtime.
- [ ] Package manifests as a Helm chart and as Kustomize overlays, and install, upgrade and roll back releases.
- [ ] Build a delivery pipeline that tests, builds an image once, deploys it, smoke-tests it and rolls back a bad release.
- [ ] Describe infrastructure as code, collect metrics, logs and traces from a cluster, and define an SLO with an error budget.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 21.1 | [What cloud native means](01-what-cloud-native-means.md) | 135 min |
| 21.2 | [Kubernetes concepts: pods, Deployments and Services](02-kubernetes-concepts.md) | 2.5 h |
| 21.3 | [A local cluster with kind and kubectl](03-a-local-cluster-with-kind.md) | 2.5 h |
| 21.4 | [Deploying a Rust service to Kubernetes](04-deploying-a-rust-service-to-kubernetes.md) | 2.5 h |
| 21.5 | [ConfigMaps, Secrets and health probes](05-configmaps-secrets-and-probes.md) | 3 h |
| 21.6 | [Packaging with Helm and Kustomize](06-helm-and-kustomize.md) | 3 h |
| 21.7 | [CI/CD: from commit to production](07-ci-cd-from-commit-to-production.md) | 2.5 h |
| 21.8 | [Infrastructure as code with OpenTofu](08-infrastructure-as-code-with-opentofu.md) | 3 h |
| 21.9 | [Metrics with Prometheus and Grafana](09-metrics-with-prometheus-and-grafana.md) | 3 h |
| 21.10 | [Logs and traces in a cluster](10-logs-and-traces-in-a-cluster.md) | 3 h |
| 21.11 | [SRE basics: SLIs, SLOs and error budgets](11-sre-basics-slis-slos-and-error-budgets.md) | 120 min |
| 21.12 | [Incidents, on-call and postmortems](12-incidents-on-call-and-postmortems.md) | 105 min |
| 21.13 | [Project: run your service on Kubernetes](13-project-run-your-service-on-kubernetes.md) | 12 h |
| 21.14 | [Checkpoint](14-checkpoint.md) | 90 min |

## Projects in this phase

- **`p21-hello`** (Lessons 21.4–21.7): a small axum service that grows with the phase — from a first Deployment to configuration, probes, a Helm chart, Kustomize overlays and a delivery pipeline.
- **Project 21.13 — run your service on Kubernetes**: your URL shortener from Lesson 16.15 (or another small service) on kind, with a Deployment, a Service, a ConfigMap and a Secret, probes, a Helm chart, a CI pipeline that builds the image, Prometheus metrics and one SLO.

## How to study this phase

- **One cluster at a time.** Each lesson creates a cluster named `course-21-NN` and deletes it at the end. Give Docker Desktop at least 4 GB of memory, and delete clusters you no longer need (`kind get clusters`, `kind delete cluster --name …`).
- **Keep everything in files and in Git.** Use imperative `kubectl` commands to explore, but write the real setup as manifests, charts and scripts. If a cluster breaks, delete it and apply your files again: that is the cloud-native way.
- **When something does not work, read in this order:** `kubectl get pods` (status and restarts), `kubectl describe pod` (events at the end), `kubectl logs` (and `--previous` after a restart), the Service's EndpointSlice. Most problems show up in the events.
- **Type the manifests yourself** at least once. Indentation and field names are learned by the fingers; `kubectl explain` and `kubectl diff` are your safety nets.
- **Use your earlier phases.** The service code comes from Phase 16, the Docker images from Phase 11, the CI from Phase 09 and the reliability ideas from Phase 20. Go back to those lessons when a term feels unfamiliar.

---

[🏠 Start](../START-HERE.md) · [← 20.11 Checkpoint](../20-reliability-and-hardening/11-checkpoint.md) · [21.1 What cloud native means →](01-what-cloud-native-means.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
