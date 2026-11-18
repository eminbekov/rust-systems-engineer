<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 21.7 CI/CD: from commit to production](07-ci-cd-from-commit-to-production.md) · [📋 Phase 21](README.md) · [21.9 Metrics with Prometheus and Grafana →](09-metrics-with-prometheus-and-grafana.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/21-cloud-native-infrastructure/08-infrastructure-as-code-with-opentofu.md) <!-- nav -->

# Lesson 21.8 — Infrastructure as code with OpenTofu

> **Phase 21 — Cloud-native infrastructure** · Lesson 8 of 14 · ⏱ about 3 hours (2 sittings)

## What you will learn

- What infrastructure as code (IaC) is, and why a team describes servers and clusters in files instead of typing commands.
- How OpenTofu works: providers, resources, variables, outputs, and the `init` → `plan` → `apply` → `destroy` cycle.
- What the state file is, why OpenTofu needs it, and why it must be protected like a password.
- How OpenTofu finds *drift* (changes made by hand) and puts things back.
- How to check a configuration with `tofu fmt`, `tofu validate` and automatic tests with `tofu test`.

## Before you start

- You finished [Lesson 21.7 — CI/CD: from commit to production](07-ci-cd-from-commit-to-production.md).
- You can create a kind cluster and use `kubectl` ([Lesson 21.3](03-a-local-cluster-with-kind.md)), and you know Deployments, Services, namespaces and ConfigMaps ([Lessons 21.2](02-kubernetes-concepts.md) and [21.5](05-configmaps-secrets-and-probes.md)).
- Docker Desktop is running. Your code for this lesson goes in `~/projects/rust/code/p21-infra`.
- Everything in this lesson runs against a local kind cluster. You need no cloud account, and nothing costs money.

## New words

| Word | Meaning |
|---|---|
| **IaC** (Infrastructure as Code) | Describing infrastructure (clusters, networks, databases, the objects inside a cluster) in text files that a tool turns into real things. The files live in Git like normal code. |
| **declarative** | You describe the *result* you want ("3 copies of the web server"), not the steps to get there. The tool works out the steps. |
| **OpenTofu** | An open-source IaC tool, run by the Linux Foundation. Its command is `tofu`. It started as a fork (a copy that continues on its own) of Terraform and reads the same kind of files. |
| **HCL** (HashiCorp Configuration Language) | The language of OpenTofu files (`.tf`): blocks with names and `key = value` lines. |
| **provider** | A plugin that teaches OpenTofu how to talk to one system: Kubernetes, Docker, a cloud. It turns resources into API (Application Programming Interface) calls. |
| **resource** | One thing that OpenTofu creates and manages, such as one namespace or one Deployment. |
| **state** | A file (`terraform.tfstate`) in which OpenTofu remembers which real objects it created for which resources. |
| **plan** | A preview: the list of changes OpenTofu *would* make to bring reality to the description. |
| **apply** | Making the changes from the plan for real. |
| **drift** | A difference between the description and reality, usually because somebody changed something by hand. |
| **variable / output** | An input value of a configuration / a value that a configuration prints and returns after `apply`. |
| **YAML** (YAML Ain't Markup Language) / **JSON** (JavaScript Object Notation) | Two text formats for structured data. Kubernetes manifests are YAML; the OpenTofu state is JSON. |
| **DNS** (Domain Name System) | The system that turns names such as `example.com` into network addresses. |
| **S3** (Simple Storage Service) | A popular protocol for storing files in "buckets" over the network; many storage systems speak it. |
| **CI** (Continuous Integration) | A server that runs checks (and here `tofu plan`) automatically for every change (Lesson 21.7). |
| **lock file** | `.terraform.lock.hcl`: the exact provider versions (and their checksums) that `tofu init` chose. Commit it to Git. |

## 1. Why describe infrastructure in files

So far you created things in the cluster in two ways: with commands such as `kubectl create namespace`, and with YAML (YAML Ain't Markup Language) manifests and `kubectl apply`. Helm and Kustomize ([Lesson 21.6](06-helm-and-kustomize.md)) packaged those manifests. That is already "infrastructure in files" for the *inside* of one cluster.

A real system needs more: the cluster itself, the network around it, a managed database, DNS (Domain Name System) names, storage buckets, access rights. Clicking them together in a web console works once. Then the questions start: *Who changed this setting? Can we build the same thing again for a test environment? What exactly will happen if I change it?*

**IaC** (Infrastructure as Code) answers these questions in the way you already know from code:

| Problem with hand-made infrastructure | What IaC gives you |
|---|---|
| Nobody knows the current setup | The files *are* the description, kept in Git |
| Changes are invisible | Every change is a commit and a pull request that somebody reviews |
| A second environment is "almost the same" | The same files with different variables |
| Risky changes | A **plan** shows every change before anything happens |
| Hand changes are forgotten | The tool finds **drift** and shows it |

IaC tools are **declarative**: you write *what* should exist, and the tool compares it with *what does* exist and makes only the difference. This is the same idea as a Kubernetes Deployment, where you say "3 replicas" and a controller keeps it true. The difference: Kubernetes controllers work all the time, while OpenTofu runs only when you (or a CI pipeline) start it.

```text
   your .tf files            state file               reality
  (what you want)      (what OpenTofu made)     (what exists now)
        │                      │                       │
        └──────────┬───────────┴───────────┬───────────┘
                   ▼                       ▼
              tofu plan  ── compare ──►  list of changes
                   │
                   ▼
              tofu apply ──► provider ──► API calls (Kubernetes, Docker, cloud…)
```

## 2. OpenTofu: install and a first configuration

**OpenTofu** is an open-source IaC tool. Its files are written in **HCL** (HashiCorp Configuration Language). OpenTofu itself knows nothing about Kubernetes or clouds: **providers** (plugins downloaded from the OpenTofu registry) do the real work. There are providers for Kubernetes, Helm, Docker and every big cloud.

Install it with Homebrew, as described on the official install page, and check the version:

```bash
brew install opentofu
tofu version
```

```text
OpenTofu v1.13.9
on darwin_arm64
```

Create a fresh cluster for this lesson (the output is the same as in Lesson 21.3) and the project folder:

```bash
kind create cluster --name course-21-08
mkdir -p ~/projects/rust/code/p21-infra
cd ~/projects/rust/code/p21-infra
```

OpenTofu reads **all** `.tf` files in the current folder as one configuration. The file names are only for people; the usual split is `versions.tf`, `variables.tf`, `main.tf` and `outputs.tf`.

`versions.tf` says which tool and provider versions we need, and how the provider connects:

```hcl
terraform {
  required_version = ">= 1.13"

  required_providers {
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 3.5"
    }
  }
}

provider "kubernetes" {
  config_path    = "~/.kube/config"
  config_context = "kind-course-21-08"
}
```

**What happens:**

1. The block is called `terraform` for compatibility: OpenTofu reads Terraform-style files unchanged.
2. `required_providers` names the provider `hashicorp/kubernetes` from the registry. `~> 3.5` means "3.5 or newer, but below 4.0": new bug fixes are welcome, a new major version (which may break things) is not.
3. The `provider` block configures it: use the kubeconfig file that kind wrote, and the context of *this* cluster. Naming the context explicitly protects you from changing the wrong cluster.

`variables.tf` declares the inputs:

```hcl
variable "namespace" {
  description = "The namespace for the web application."
  type        = string
  default     = "web"
}

variable "replicas" {
  description = "How many copies (pods) of the web server to run."
  type        = number
  default     = 2

  validation {
    condition     = var.replicas >= 1 && var.replicas <= 5
    error_message = "replicas must be between 1 and 5."
  }
}
```

A variable has a type, an optional default and optional **validation** rules. Inside the configuration you read it as `var.replicas`.

`main.tf` describes three resources: a namespace, a ConfigMap with a web page, and a Deployment of the official `nginx` web server image that shows this page. We use `nginx` so that this lesson does not depend on your own image; the ideas are exactly the same for your Rust service.

```hcl
resource "kubernetes_namespace_v1" "web" {
  metadata {
    name = var.namespace
  }
}

resource "kubernetes_config_map_v1" "page" {
  metadata {
    name      = "page"
    namespace = kubernetes_namespace_v1.web.metadata[0].name
  }

  data = {
    "index.html" = "<h1>Created by OpenTofu</h1>\n"
  }
}

resource "kubernetes_deployment_v1" "web" {
  metadata {
    name      = "web"
    namespace = kubernetes_namespace_v1.web.metadata[0].name
  }

  spec {
    replicas = var.replicas

    selector {
      match_labels = { app = "web" }
    }

    template {
      metadata {
        labels = { app = "web" }
      }

      spec {
        container {
          name  = "nginx"
          image = "nginx:1.30-alpine"

          port {
            container_port = 80
          }

          volume_mount {
            name       = "page"
            mount_path = "/usr/share/nginx/html"
          }
        }

        volume {
          name = "page"
          config_map {
            name = kubernetes_config_map_v1.page.metadata[0].name
          }
        }
      }
    }
  }
}
```

**What happens:**

1. `resource "kubernetes_namespace_v1" "web"` has two labels: the **resource type** (defined by the provider) and a **local name** (yours). Together they form the address `kubernetes_namespace_v1.web`.
2. The blocks inside mirror the Kubernetes YAML you know: `metadata`, `spec`, `template`, `container`. Only the spelling differs (`match_labels` instead of `matchLabels`).
3. `kubernetes_namespace_v1.web.metadata[0].name` is a **reference** to another resource. OpenTofu sees it and knows the order: first the namespace, then the ConfigMap, then the Deployment. You never write the order yourself.
4. The ConfigMap holds a file `index.html`; the Deployment mounts it as a volume where `nginx` looks for its start page.

`outputs.tf` says what to print at the end:

```hcl
output "namespace" {
  value = kubernetes_namespace_v1.web.metadata[0].name
}

output "replicas" {
  value = kubernetes_deployment_v1.web.spec[0].replicas
}
```

## 3. The cycle: init, plan, apply

**`tofu init`** prepares the folder: it downloads the providers and writes the lock file. Run it once, and again whenever you add a provider or a module.

```bash
tofu init
```

```text
Initializing provider plugins...
- Finding hashicorp/kubernetes versions matching "~> 3.5"...
- Installed hashicorp/kubernetes v3.5.0 (signed, key ID 0C0AF313E5FD9F80)
...
OpenTofu has created a lock file .terraform.lock.hcl to record the provider
...
OpenTofu has been successfully initialized!
```

The provider is now in the hidden folder `.terraform/`. Check the files, then look at the plan:

```bash
tofu fmt -check
tofu validate
tofu plan
```

Shortened output (the real plan lists about 100 lines, most of them default values):

```text
Success! The configuration is valid.
...
  # kubernetes_config_map_v1.page will be created
  + resource "kubernetes_config_map_v1" "page" {
      + data = {
          + "index.html" = <<-EOT
                <h1>Created by OpenTofu</h1>
            EOT
        }
      + id   = (known after apply)
  ...
  # kubernetes_deployment_v1.web will be created
          + replicas                  = "2"
  ...
  # kubernetes_namespace_v1.web will be created
  ...
Plan: 3 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + namespace = "web"
  + replicas  = "2"
```

**What happens:**

1. `tofu fmt -check` prints nothing when the files are formatted the standard way (without `-check` it fixes them, like `cargo fmt`). `tofu validate` checks the syntax and the references without contacting the cluster.
2. `tofu plan` compares the files with the state (empty now) and the cluster. `+` means "will be created", `~` "will be changed in place", `-` "will be destroyed", and `-/+` "must be destroyed and created again".
3. `(known after apply)` marks values that only the cluster can give, such as the unique id.
4. The summary line is the most important line of every plan. Read it before every `apply`.

Now make it real. `tofu apply` shows the same plan again and asks for confirmation; only the word `yes` is accepted:

```bash
tofu apply
```

```text
...
Do you want to perform these actions?
  OpenTofu will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

kubernetes_namespace_v1.web: Creation complete after 0s [id=web]
kubernetes_config_map_v1.page: Creation complete after 0s [id=web/page]
kubernetes_deployment_v1.web: Still creating... [20s elapsed]
kubernetes_deployment_v1.web: Creation complete after 26s [id=web/web]

Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

namespace = "web"
replicas = "2"
```

(We removed the plan at the top and the "Creating..." lines.)

The resources were created in dependency order, and the provider waited until the Deployment's pods were ready. Check with the tools you know:

```bash
kubectl get deployments -n web
kubectl -n web port-forward deployment/web 8088:80
```

```text
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    2/2     2            2           26s
```

While the port-forward runs, `curl -s localhost:8088` in a second terminal prints `<h1>Created by OpenTofu</h1>`. Stop the port-forward with Ctrl+C.

> 📏 **Convention:** In a team, nobody runs `tofu apply` from a laptop against production. A CI (Continuous Integration) pipeline ([Lesson 21.7](07-ci-cd-from-commit-to-production.md)) runs `tofu plan` on every pull request and posts the plan for review; after the merge, the pipeline runs `tofu apply` with exactly that saved plan (`tofu plan -out=plan.bin`, then `tofu apply plan.bin`).

## 4. Changing things, and finding drift

Change a value the declarative way: say what you want. A variable can be set on the command line:

```bash
tofu apply -var replicas=3
```

```text
  # kubernetes_deployment_v1.web will be updated in-place
  ~ resource "kubernetes_deployment_v1" "web" {
        id               = "web/web"
      ~ spec {
          ~ replicas                  = "2" -> "3"
            # (4 unchanged attributes hidden)
            # (3 unchanged blocks hidden)
        }
        # (1 unchanged block hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

After `yes`, the plan says `Apply complete! Resources: 0 added, 1 changed, 0 destroyed.` Only the one changed field was touched; the namespace and the ConfigMap were left alone.

A value given with `-var` is not remembered: the next `tofu plan` without it would want to go back to 2. Values that should stay go into the file `terraform.tfvars`, which OpenTofu reads automatically:

```bash
echo 'replicas = 3' > terraform.tfvars
tofu plan
```

```text
kubernetes_namespace_v1.web: Refreshing state... [id=web]
kubernetes_config_map_v1.page: Refreshing state... [id=web/page]
kubernetes_deployment_v1.web: Refreshing state... [id=web/web]

No changes. Your infrastructure matches the configuration.
```

"Refreshing state" means OpenTofu asked the cluster about every resource it manages. That is how it notices **drift**. Make a change by hand, as a colleague might do during a busy night, and plan again:

```bash
kubectl -n web scale deployment web --replicas=1
tofu plan
```

```text
deployment.apps/web scaled
...
          ~ replicas                  = "1" -> "3"
...
Plan: 0 to add, 1 to change, 0 to destroy.
```

OpenTofu found that reality (1) differs from the description (3), and `tofu apply` would put it back. This is why a team agrees on one rule: **change managed infrastructure only through the files.** A hand fix that is not written into the files is undone at the next `apply`. Run `tofu apply` now and type `yes` to remove the drift.

> ⚠️ **Common mistake:** Two tools managing the same object. If a Helm chart *and* OpenTofu both manage the Deployment `web`, each one undoes the other's changes. Every object should have exactly one owner.

## 5. The state file

After `apply` there is a new file `terraform.tfstate` in your folder. It is JSON (JavaScript Object Notation): a list of every resource address and the real object behind it.

```bash
tofu state list
tofu state show kubernetes_config_map_v1.page
```

Shortened output:

```text
kubernetes_config_map_v1.page
kubernetes_deployment_v1.web
kubernetes_namespace_v1.web
# kubernetes_config_map_v1.page:
resource "kubernetes_config_map_v1" "page" {
    ...
    id          = "web/page"
    metadata {
        name             = "page"
        namespace        = "web"
        resource_version = "582"
        uid              = "ddf06963-6a2d-4f8c-b99d-2d8000f3c293"
    ...
```

Why does OpenTofu need the state at all, if it can ask the cluster? Because of what you *remove* from the files. If you delete the ConfigMap block, the files no longer mention it. Only the state remembers "I created `web/page` for this block", so the next plan can say `- destroy`. Without the state, OpenTofu could not tell its own objects from everybody else's.

Rules for the state:

- **Never edit it by hand.** Use `tofu state` commands if you really must change it.
- **Never lose it.** Without it, OpenTofu forgets what it owns. On a team, the state lives in a shared **backend** (for example a storage bucket that speaks the S3 (Simple Storage Service) protocol or a PostgreSQL database), not on one laptop. A good backend also **locks** the state, so two people cannot apply at the same time.
- **Do not commit it to Git.** Add these lines to the project's `.gitignore`:

```text
.terraform/
*.tfstate
*.tfstate.backup
```

The lock file `.terraform.lock.hcl`, on the other hand, **is** committed, like `Cargo.lock`.

> 🔒 **Security:** The state contains every attribute of every resource, *including secret values*: the data of a Kubernetes Secret, database passwords, private keys. Anyone who can read the state can read those secrets. Keep it in a backend with access control, never in Git, never in a public bucket. OpenTofu can also encrypt the state itself (the `encryption` block, see the sources).

## 6. Automatic checks: `tofu test`

You check Rust with `cargo fmt`, `cargo clippy` and `cargo test`. OpenTofu has the same three levels: `tofu fmt -check`, `tofu validate`, and `tofu test`.

A test file ends in `.tftest.hcl` and lives in a `tests/` folder. Each `run` block runs a plan (or an apply) with given variables and checks conditions. Create `tests/web.tftest.hcl`:

```hcl
run "uses_the_replicas_variable" {
  command = plan

  variables {
    replicas = 4
  }

  assert {
    condition     = kubernetes_deployment_v1.web.spec[0].replicas == "4"
    error_message = "the Deployment must use var.replicas"
  }
}

run "rejects_zero_replicas" {
  command = plan

  variables {
    replicas = 0
  }

  expect_failures = [var.replicas]
}
```

**What happens:**

1. `command = plan` makes the test only *plan*. Nothing in the cluster is created or changed, so the test is fast and safe. (With `command = apply`, OpenTofu creates real resources for the test and destroys them afterwards.)
2. `variables` sets the inputs for this run. They win over `terraform.tfvars`.
3. `assert` checks a condition on the planned values. The provider stores `replicas` as text, so we compare with `"4"`.
4. `expect_failures = [var.replicas]` says: "this run *must* fail, and the failure must come from the validation of `var.replicas`". It tests that bad input is rejected.

```bash
tofu test
```

```text
tests/web.tftest.hcl... pass
  run "uses_the_replicas_variable"... pass
  run "rejects_zero_replicas"... pass

Success! 2 passed, 0 failed.
```

To see that the test can fail, change `replicas = var.replicas` in `main.tf` to `replicas = 2` and run `tofu test` again:

```text
tests/web.tftest.hcl... fail
  run "uses_the_replicas_variable"... fail

Error: Test assertion failed
  on tests/web.tftest.hcl line 9, in run "uses_the_replicas_variable":
   9:     condition     = kubernetes_deployment_v1.web.spec[0].replicas == "4"
    │ kubernetes_deployment_v1.web.spec[0].replicas is "2"

the Deployment must use var.replicas
  run "rejects_zero_replicas"... pass

Failure! 1 passed, 1 failed.
```

(Shortened: we removed the frame lines and the diff.)

Put it back to `var.replicas`.

> 🧪 **Testing:** In CI, run `tofu fmt -check -recursive`, `tofu validate` and `tofu test` on every pull request, before anybody looks at the plan. Tests with `command = plan` still need a provider connection (here: the kind cluster); in a pipeline you create a kind cluster first, as in Lesson 21.7.

## 7. Destroying everything

`tofu destroy` removes every resource in the state, in reverse dependency order. It asks for `yes` too. When you finish this lesson (after the exercises), run:

```bash
tofu destroy
```

```text
Plan: 0 to add, 0 to change, 3 to destroy.
...
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes
...
kubernetes_namespace_v1.web: Destruction complete after 6s

Destroy complete! Resources: 3 destroyed.
```

(Your counts are higher if you did the exercises.) Then delete the cluster: `kind delete cluster --name course-21-08`.

> 💡 **Tip:** When you need the same group of resources several times (a test and a production environment), move them into a **module**: a folder of `.tf` files with its own variables, used like a function with `module "test" { source = "./modules/web" ... }`. The same configuration style also works for other systems. The `kreuzwerker/docker` provider manages local Docker containers and networks; cloud providers manage real clusters. Only the resource types change; `init`, `plan`, `apply`, state and drift work the same everywhere.

## Common mistakes

> ⚠️ **Common mistake:** Running `tofu plan` in a new folder (or after adding a provider) without `tofu init`:

```text
Error: Inconsistent dependency lock file
...
  - provider registry.opentofu.org/hashicorp/kubernetes: required by this configuration but no version is selected
...
  tofu init
```

Fix: run `tofu init`.

> ⚠️ **Common mistake:** A wrong context name in the `provider` block (here `kind-course-21.08` with a dot instead of a dash):

```text
Error: Provider configuration: cannot load Kubernetes client config
...
context "kind-course-21.08" does not exist
```

Fix: `kubectl config get-contexts` lists the real names.

> ⚠️ **Common mistake:** Deleting `terraform.tfstate` "to start fresh". The objects in the cluster stay, but OpenTofu forgets them. The next `apply` tries to create them again and fails with "already exists". Use `tofu destroy` instead.

## Try it yourself

1. Change the text in the ConfigMap to `<h1>Changed by OpenTofu</h1>\n` and run `tofu plan`. Which symbol and which summary line do you see? (`~`, `Plan: 0 to add, 1 to change, 0 to destroy.`) Apply it, check the page with a port-forward, then change the text back and apply again.
2. Delete the namespace by hand: `kubectl delete namespace web`. Run `tofu plan`. OpenTofu notices that all three objects are gone and plans to create them again (`3 to add`). Apply to repair.
3. Run `tofu output replicas`. It prints only that one output, which is handy in scripts.

## Exercises

### Exercise 1 — Add a Service (easy)

Add a resource `kubernetes_service_v1` named `web` in the same namespace. It selects the pods with `app = "web"` and forwards port 80 to port 80. `tofu apply` must show `Plan: 1 to add, 0 to change, 0 to destroy.`, and `kubectl -n web port-forward service/web 8088:80` plus `curl -s localhost:8088` must print the page.

<details>
<summary>Hint</summary>

The structure is the same as the YAML Service from Lesson 21.2: `metadata` with a name and a namespace, and `spec` with `selector` and one `port` block with `port` and `target_port`.

</details>

<details>
<summary>Answer</summary>

Add to `main.tf`:

```hcl
resource "kubernetes_service_v1" "web" {
  metadata {
    name      = "web"
    namespace = kubernetes_namespace_v1.web.metadata[0].name
  }

  spec {
    selector = { app = "web" }

    port {
      port        = 80
      target_port = 80
    }
  }
}
```

```text
Plan: 1 to add, 0 to change, 0 to destroy.
...
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

Why it works: the reference to the namespace makes OpenTofu create the Service after the namespace. Only the new resource appears in the plan, because the others already match.

</details>

### Exercise 2 — A variable with a test (medium)

Add a variable `message` (a string, default `"Created by OpenTofu"`, 1 to 80 characters) and use it in the page: `"<h1>${var.message}</h1>\n"` (`${...}` inserts a value into a string). Then **write a test** file `tests/page.tftest.hcl` with two runs: one checks that the planned page is exactly `<h1>Hello from a test</h1>\n` when `message = "Hello from a test"`, the other checks that an empty message is rejected. `tofu test` must pass all runs, and `tofu plan` must still say "No changes".

<details>
<summary>Hint</summary>

Use `length(var.message)` in the validation condition. In the test, read the planned page as `kubernetes_config_map_v1.page.data["index.html"]`.

</details>

<details>
<summary>Answer</summary>

In `variables.tf`:

```hcl
variable "message" {
  description = "The heading shown on the page."
  type        = string
  default     = "Created by OpenTofu"

  validation {
    condition     = length(var.message) > 0 && length(var.message) <= 80
    error_message = "message must have 1 to 80 characters."
  }
}
```

In `main.tf`, the ConfigMap data becomes `"index.html" = "<h1>${var.message}</h1>\n"`. The test file:

```hcl
run "page_shows_the_message" {
  command = plan

  variables {
    message = "Hello from a test"
  }

  assert {
    condition     = kubernetes_config_map_v1.page.data["index.html"] == "<h1>Hello from a test</h1>\n"
    error_message = "the page must contain var.message inside <h1>"
  }
}

run "rejects_an_empty_message" {
  command = plan

  variables {
    message = ""
  }

  expect_failures = [var.message]
}
```

```text
tests/page.tftest.hcl... pass
  run "page_shows_the_message"... pass
  run "rejects_an_empty_message"... pass
tests/web.tftest.hcl... pass
  run "uses_the_replicas_variable"... pass
  run "rejects_zero_replicas"... pass

Success! 4 passed, 0 failed.
```

Why it works: the default produces the same text as before, so the real cluster needs no change. The tests use plan mode, so they check the configuration without touching the cluster.

</details>

### Exercise 3 — Remove a resource (hard)

Remove the ConfigMap from the files: delete its `resource` block, and the `volume_mount` and `volume` blocks of the Deployment that use it. **Before** you run anything, write down what the plan will say: which resources, which symbols, which summary line. Then run `tofu plan` and compare. Finally explain in two sentences which part of OpenTofu makes this plan possible. (Undo your edit afterwards, or keep it and apply.)

<details>
<summary>Hint</summary>

The files no longer mention the ConfigMap at all. Where does OpenTofu learn that it exists?

</details>

<details>
<summary>Answer</summary>

Shortened output:

```text
  # kubernetes_config_map_v1.page will be destroyed
  # (because kubernetes_config_map_v1.page is not in configuration)
  - resource "kubernetes_config_map_v1" "page" {
  ...
  # kubernetes_deployment_v1.web will be updated in-place
  ~ resource "kubernetes_deployment_v1" "web" {
  ...
                      - volume_mount {
  ...
                  - volume {
  ...
Plan: 0 to add, 1 to change, 1 to destroy.
```

Why it works: the **state** still lists `kubernetes_config_map_v1.page` with the real object `web/page`. OpenTofu compares the state with the files, sees a resource that is no longer described, and plans to destroy it ("not in configuration"); the Deployment only loses two blocks, so it is changed in place, not replaced.

</details>

## Quiz

1. What does "declarative" mean for an IaC tool?

<details>
<summary>Answer</summary>

You describe the result you want, not the steps. The tool compares the description with reality and works out which changes are needed.

</details>

2. Why does OpenTofu need a state file when it can ask the cluster?

<details>
<summary>Answer</summary>

To know which real objects belong to which resource blocks. When you remove a block from the files, only the state remembers that the object exists and must be destroyed.

</details>

3. Why must the state never go into Git?

<details>
<summary>Answer</summary>

It contains every attribute of every resource, including secret values. It also changes on every apply and must be locked while somebody applies, which Git cannot do.

</details>

4. A colleague scaled a Deployment by hand to 6 replicas; the files say 3. What happens at the next `tofu apply`?

<details>
<summary>Answer</summary>

The plan shows the drift (`6 -> 3`) and the apply sets the replicas back to 3. If 6 is right, change the files, not the cluster.

</details>

5. What does `command = plan` in a test do, and what does `expect_failures` check?

<details>
<summary>Answer</summary>

`command = plan` runs only a plan, so nothing is created. `expect_failures` says the run must fail because of the named check, for example a variable's validation.

</details>

## Summary

- IaC describes infrastructure in files kept in Git, so changes are reviewed, repeatable and visible before they happen.
- OpenTofu reads `.tf` files in HCL; providers turn resources into API calls. References between resources set the order.
- The cycle is `tofu init` → `tofu plan` → `tofu apply` → (finally) `tofu destroy`. Always read the plan's summary line.
- The state maps resources to real objects. Protect it like a password, keep it in a locked backend, never in Git.
- `tofu plan` finds drift. Change managed things only through the files, and give every object one owner.
- Check configurations with `tofu fmt -check`, `tofu validate`, variable validation and `tofu test`.

## Sources and further reading

- [OpenTofu — What is OpenTofu?](https://opentofu.org/docs/intro/) — the official introduction and the core workflow.
- [OpenTofu — Installing with Homebrew](https://opentofu.org/docs/intro/install/homebrew/) and [standalone installation](https://opentofu.org/docs/intro/install/standalone/) — how to install the `tofu` command.
- [OpenTofu — State](https://opentofu.org/docs/language/state/), [Backends](https://opentofu.org/docs/language/state/backends/) and [State and plan encryption](https://opentofu.org/docs/language/state/encryption/) — why state exists, where to keep it and how to encrypt it.
- [OpenTofu — Input variables](https://opentofu.org/docs/language/values/variables/) and [Modules](https://opentofu.org/docs/language/modules/) — variables, validation and reusable modules.
- [OpenTofu — The `tofu test` command](https://opentofu.org/docs/cli/commands/test/) — test files, `run` blocks, `assert` and `expect_failures`.
- [The Kubernetes provider in the OpenTofu registry](https://search.opentofu.org/provider/hashicorp/kubernetes/latest) — every resource type and its fields.

---

[🏠 Start](../START-HERE.md) · [← 21.7 CI/CD: from commit to production](07-ci-cd-from-commit-to-production.md) · [📋 Phase 21](README.md) · [21.9 Metrics with Prometheus and Grafana →](09-metrics-with-prometheus-and-grafana.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
