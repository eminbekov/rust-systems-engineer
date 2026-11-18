<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 22.26 Cloud-native track: Kubernetes operators](26-cloud-native-track-kubernetes-operators.md) · [📋 Phase 22](README.md) · [22.28 Cloud-native track: testing and running operators →](28-cloud-native-track-testing-and-running-operators.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/22-specializations/27-cloud-native-track-an-operator-with-kube-rs.md) <!-- nav -->

# Lesson 22.27 — Cloud-native track: an operator with kube-rs

> **Phase 22 — Specializations** · Lesson 27 of 30 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- How to describe a custom resource as a Rust struct with `#[derive(CustomResource)]` and generate its CRD.
- How an operator reads and writes objects through `kube::Client` and `Api<T>`.
- How owner references make Kubernetes delete child objects for you (garbage collection).
- How to write a reconcile function with server-side apply and a status update.
- How `Controller` from kube-runtime turns that function into a running control loop.

## Before you start

- You finished [Lesson 22.26 — Cloud-native track: Kubernetes operators](26-cloud-native-track-kubernetes-operators.md).
  The kind cluster `course-22-operators` is running (create it again as shown there if you deleted it), and
  `kubectl config current-context` prints `kind-course-22-operators`.
- You know serde and `thiserror` ([Lesson 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md)),
  derive macros ([Lesson 06.8](../06-rust-advanced/08-procedural-macros.md)) and `Arc`
  ([Lesson 05.10](../05-rust-intermediate/10-threads-channels-and-shared-state.md)).
- You know async functions and Tokio ([Lesson 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md)).

## New words

| Word | Meaning |
|---|---|
| **kube-rs** | The Rust project for Kubernetes. Its main crate `kube` re-exports the client (`kube-client`), the controller runtime (`kube-runtime`) and the derive macros (`kube-derive`). |
| **`Client`** | A connection to the API (Application Programming Interface) server. `Client::try_default()` reads your kubeconfig on your Mac, or the pod's service account inside a cluster. |
| **`Api<T>`** | A handle for one object type in one namespace (or all namespaces): `get`, `list`, `patch`, `delete`, … |
| **status subresource** | A separate part of a custom resource, `/status`, that only the operator writes. Users write `spec`, the operator reports in `status`. |
| **owner reference** | A field in a child object's metadata that names its owner (kind, name and uid, the unique ID that the API server gives every object). When the owner is deleted, Kubernetes deletes the child. |
| **garbage collection** | The Kubernetes controller that deletes objects whose owner no longer exists. |
| **SSA** (Server-Side Apply) | A way to write an object by sending the *whole desired object*. The API server merges it with what is stored and remembers which fields you own. `kubectl apply --server-side` does the same. |
| **field manager** | The name a writer signs its changes with in SSA (here `greeter-operator`), so the API server knows who owns which field. |
| **merge patch** | A partial update: you send only the fields you want to change, as JSON (JavaScript Object Notation). |
| **requeue** | Asking the controller to run reconcile for this object again after some time, even if nothing changes. |
| **error policy** | A function that decides what to do when reconcile returns an error, usually "try again in a few seconds". |
| **watch / watcher** | A long-running API request that streams every change of some objects. `watcher` is the kube-runtime helper around it. |
| **reflector / store** | A local copy of the watched objects inside the operator, kept up to date by the watcher. The controller reads from it instead of asking the API server each time. |
| **CRD** (CustomResourceDefinition) | The object that adds a new type to the Kubernetes API (Lesson 22.26). |

## 1. The project

The operator from this lesson makes, for every `Greeting`, a ConfigMap `<name>-greeting` with the message in it.
It is the simulation from Lesson 22.26, now against a real API server.

```bash
cd ~/projects/rust
cargo new code/p22-greeter-operator
cd code/p22-greeter-operator
mkdir -p src/bin k8s
```

Add the dependencies to `Cargo.toml`:

```toml
[dependencies]
kube = { version = "5.1", features = ["runtime", "derive"] }
k8s-openapi = { version = "0.29", features = ["latest", "schemars"] }
schemars = "1"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
serde-saphyr = "2"
tokio = { version = "1", features = ["macros", "rt-multi-thread"] }
futures = "0.3"
thiserror = "2"
```

The feature `schemars` of `k8s-openapi` lets schemars describe built-in types too. `serde-saphyr` writes YAML (YAML Ain't Markup
Language, the format of manifests; the older `serde_yaml` crate is deprecated). The project has three source files:

```text
code/p22-greeter-operator/
├── Cargo.toml
├── k8s/                  ← manifests (the CRD, example Greetings)
└── src/
    ├── lib.rs            ← the Greeting type and the pure "what should exist" logic
    ├── main.rs           ← the operator: reconcile + controller
    └── bin/crdgen.rs     ← a tiny program that prints the CRD as YAML
```

📏 **Convention:** keep the *pure* part (types, "build the desired object") in `lib.rs`, without any network calls.
Both binaries use it, and you can unit-test it without a cluster (you will in Exercise 2).

## 2. The Greeting type in Rust

In Lesson 22.26 you wrote the schema by hand in YAML. Now the Rust struct is the single source of truth.
Write `src/lib.rs`:

```rust
use std::collections::BTreeMap;

use k8s_openapi::api::core::v1::ConfigMap;
use kube::api::ObjectMeta;
use kube::{CustomResource, Resource, ResourceExt};
use schemars::JsonSchema;
use serde::{Deserialize, Serialize};

/// The `spec` of a Greeting: what the user wants.
#[derive(CustomResource, Clone, Debug, Deserialize, Serialize, JsonSchema)]
#[kube(group = "course.example.com", version = "v1", kind = "Greeting")]
#[kube(namespaced, shortname = "gr", status = "GreetingStatus")]
#[kube(printcolumn(name = "Message", type_ = "string", json_path = ".spec.message"))]
pub struct GreetingSpec {
    /// The text to put into the ConfigMap.
    #[schemars(length(min = 1, max = 100))]
    pub message: String,
    /// Write the message in capital letters.
    #[serde(default)]
    pub shout: bool,
}

/// The `status` of a Greeting: what the operator did.
#[derive(Clone, Debug, Default, Deserialize, Serialize, JsonSchema)]
#[serde(rename_all = "camelCase")]
pub struct GreetingStatus {
    pub config_map: Option<String>,
}

/// The name of the ConfigMap that belongs to a Greeting.
pub fn config_map_name(greeting: &Greeting) -> String {
    format!("{}-greeting", greeting.name_any())
}

/// Builds the ConfigMap that should exist for this Greeting.
pub fn desired_config_map(greeting: &Greeting) -> ConfigMap {
    let text = if greeting.spec.shout {
        greeting.spec.message.to_uppercase()
    } else {
        greeting.spec.message.clone()
    };
    ConfigMap {
        metadata: ObjectMeta {
            name: Some(config_map_name(greeting)),
            namespace: greeting.namespace(),
            owner_references: greeting.controller_owner_ref(&()).map(|r| vec![r]),
            ..ObjectMeta::default()
        },
        data: Some(BTreeMap::from([("message".to_string(), text)])),
        ..ConfigMap::default()
    }
}
```

**What happens:**

1. You write only `GreetingSpec`. `#[derive(CustomResource)]` **generates** a second struct, `Greeting`, the
   whole object: `metadata` (name, namespace, uid, labels, …), your `spec`, and `status`. It also gives it the
   method `Greeting::crd()` and `Greeting::new(name, spec)`.
2. The `#[kube(...)]` lines carry the same information as the CRD YAML: group, version, kind, `namespaced`, the short
   name and a printer column. `status = "GreetingStatus"` adds the status subresource.
3. `JsonSchema` (from schemars) builds the OpenAPI schema from the struct. Doc comments (`///`) become field
   descriptions, `#[schemars(length(min = 1, max = 100))]` becomes `minLength`/`maxLength`, and
   `#[serde(default)]` makes `shout` optional with default `false`.
4. `#[serde(rename_all = "camelCase")]` turns the Rust name `config_map` into `configMap`, the usual Kubernetes
   style for JSON fields.
5. `desired_config_map` builds the ConfigMap that *should* exist. It does not talk to the cluster. `ConfigMap` and
   `ObjectMeta` are structs from `k8s-openapi`; `..ObjectMeta::default()` fills every other field with `None`.
6. `controller_owner_ref(&())` builds an owner reference that points at this Greeting (more in Section 3). The `&()`
   is "no extra type information", which is always the case for types known at compile time.

The generated `Greeting` looks like this:

```text
Greeting
├── metadata: ObjectMeta              ← name, namespace, uid, labels, ownerReferences, …
├── spec:     GreetingSpec            ← your struct
└── status:   Option<GreetingStatus>  ← None until the operator writes it
```

Now the tiny CRD generator, `src/bin/crdgen.rs`:

```rust
use kube::CustomResourceExt;
use p22_greeter_operator::Greeting;

fn main() {
    let crd = Greeting::crd();
    let yaml = serde_saphyr::to_string(&crd).expect("the CRD can be written as YAML");
    print!("{yaml}");
}
```

Generate the CRD, look at it and apply it (it replaces the hand-written one from Lesson 22.26):

```bash
cargo run -q --bin crdgen > k8s/greeting-crd.yaml
cat k8s/greeting-crd.yaml
kubectl apply -f k8s/greeting-crd.yaml
```

```text
…
              shout:
                default: false
                description: Write the message in capital letters.
                type: boolean
            required:
            - message
            type: object
          status:
            description: "The `status` of a Greeting: what the operator did."
            nullable: true
            properties:
              configMap:
                nullable: true
                type: string
            type: object
        required:
        - spec
        title: Greeting
        type: object
    served: true
    storage: true
    subresources:
      status: {}
customresourcedefinition.apiextensions.k8s.io/greetings.course.example.com configured
```

The output above shows the end of the file (`…` marks the part left out). It is the CRD from Lesson 22.26 plus three
things: descriptions from your doc comments, the `status` part of the schema and `subresources: status: {}`.
`configured` means the existing CRD was updated.

> 🧪 **Testing:** commit `k8s/greeting-crd.yaml` to Git. When you change the struct, run `crdgen` again; the Git diff
> shows exactly how the API of your operator changed. A CI (continuous integration, Lesson 09.12) job can fail if
> the committed file is out of date.

## 3. Owner references and garbage collection

Who deletes the ConfigMap when the user deletes the Greeting? Not your operator. The ConfigMap carries an **owner
reference** in `metadata.ownerReferences`: the owner's `apiVersion`, `kind`, `name` and `uid` (so it points at this
exact object, not at any Greeting called `hello`), plus `controller: true` (this owner is the one that manages it).
You will see it in the real ConfigMap in Section 6.

When the Greeting is deleted, the **garbage collector** of Kubernetes sees a ConfigMap whose owner is gone and
deletes it. This also works if your operator is not running at that moment. Two rules: owner and child must be in
the same namespace, and the owner must already exist (it needs a `uid`), which is always true inside reconcile.

## 4. The reconcile function

Now `src/main.rs`. First the reconcile function and the error policy:

```rust
use std::sync::Arc;
use std::time::Duration;

use futures::StreamExt;
use k8s_openapi::api::core::v1::ConfigMap;
use kube::api::{Patch, PatchParams};
use kube::runtime::controller::Action;
use kube::runtime::{Controller, watcher};
use kube::{Api, Client, ResourceExt};
use p22_greeter_operator::{Greeting, GreetingStatus, desired_config_map};

/// The name we sign our changes with (the "field manager").
const MANAGER: &str = "greeter-operator";

#[derive(Debug, thiserror::Error)]
enum Error {
    #[error("Kubernetes API error: {0}")]
    Kube(#[from] kube::Error),
}

/// Things every reconcile call needs.
struct Context {
    client: Client,
}

async fn reconcile(greeting: Arc<Greeting>, ctx: Arc<Context>) -> Result<Action, Error> {
    let ns = greeting
        .namespace()
        .unwrap_or_else(|| "default".to_string());
    let name = greeting.name_any();
    println!("reconcile Greeting {ns}/{name}");

    // 1. Make the ConfigMap look exactly like we want it.
    let cm = desired_config_map(&greeting);
    let cm_name = cm.name_any();
    let config_maps: Api<ConfigMap> = Api::namespaced(ctx.client.clone(), &ns);
    let params = PatchParams::apply(MANAGER).force();
    config_maps
        .patch(&cm_name, &params, &Patch::Apply(&cm))
        .await?;

    // 2. Tell the user what we did, in the status.
    let greetings: Api<Greeting> = Api::namespaced(ctx.client.clone(), &ns);
    let status = GreetingStatus {
        config_map: Some(cm_name),
    };
    let patch = serde_json::json!({ "status": status });
    greetings
        .patch_status(&name, &PatchParams::default(), &Patch::Merge(&patch))
        .await?;

    // 3. Look again in 5 minutes, even if nothing changes.
    Ok(Action::requeue(Duration::from_secs(300)))
}

fn error_policy(greeting: Arc<Greeting>, error: &Error, _ctx: Arc<Context>) -> Action {
    eprintln!("reconcile of {} failed: {error}", greeting.name_any());
    Action::requeue(Duration::from_secs(5))
}
```

**What happens:**

1. `reconcile` gets the Greeting as `Arc<Greeting>` (the controller shares one copy from its store) and a shared
   `Context` with the `Client`. It returns either an `Action` (what to do next) or our `Error`.
2. `Api::namespaced(client, &ns)` is a handle for ConfigMaps in the Greeting's namespace. Cloning a `Client` is cheap:
   all clones share one connection pool.
3. `Patch::Apply` with `PatchParams::apply(MANAGER)` is **server-side apply**. We send the *whole* desired
   ConfigMap. If it does not exist, the API server creates it; if it differs, it updates it; if it is equal, nothing
   changes. That is exactly the idempotent "make it so" from Lesson 22.26, done by the API server for us.
   `.force()` takes over fields that another manager (say, a person with `kubectl edit`) changed: the operator wins.
4. `patch_status` writes only the status subresource, with a **merge patch** (`Patch::Merge`): just the JSON
   `{"status": {"configMap": "hello-greeting"}}`. Users cannot change `status` through normal writes to the Greeting.
5. `Action::requeue(300 s)` asks for another reconcile in five minutes. This catches drift that no watch reports.
6. The `?` after each call turns a `kube::Error` into our `Error` (thanks to `#[from]`) and returns early. Then the
   controller calls `error_policy`, which logs and asks to retry in five seconds.

> ⚡ **Performance:** reconcile builds the full desired object and lets the API server compare. There is no
> "get, compare, then decide" step in our code, so one call does the job and there is no race between reading and
> writing.

## 5. The controller

The rest of `src/main.rs` starts the control loop:

```rust
#[tokio::main]
async fn main() -> Result<(), kube::Error> {
    let client = Client::try_default().await?;
    let greetings: Api<Greeting> = Api::all(client.clone());
    let config_maps: Api<ConfigMap> = Api::all(client.clone());

    println!("greeter-operator started");
    Controller::new(greetings, watcher::Config::default())
        .owns(config_maps, watcher::Config::default())
        .shutdown_on_signal()
        .run(reconcile, error_policy, Arc::new(Context { client }))
        .for_each(|result| async move {
            match result {
                Ok((object, _action)) => println!("done: {}", object.name),
                Err(error) => eprintln!("controller error: {error}"),
            }
        })
        .await;
    println!("greeter-operator stopped");
    Ok(())
}
```

**What happens:**

1. `Client::try_default()` finds the cluster: on your Mac it reads the kubeconfig and its current context; inside a
   pod it uses the pod's service account (Lesson 22.28).
2. `Api::all` watches Greetings in **all** namespaces.
3. `Controller::new` starts a watcher on Greetings and keeps a local store (reflector) of them. Every change of a
   Greeting puts it into a queue for reconcile.
4. `.owns(config_maps, …)` also watches ConfigMaps. When a ConfigMap with an owner reference to a Greeting changes or
   disappears, the controller reconciles **the owner**. That is how a hand-deleted ConfigMap comes back within a second.
5. `.shutdown_on_signal()` stops cleanly on Ctrl+C (SIGINT) or SIGTERM: no new reconciles start, running ones finish.
6. `.run(...)` returns a *stream* of results, one per finished reconcile. `for_each` consumes it; the program lives as
   long as this stream does.

The controller also guarantees that reconcile never runs twice **at the same time for the same object**, and it
merges many quick changes of one object into one reconcile.

## 6. Run the operator

The operator runs on your Mac and talks to the kind cluster through your kubeconfig. Open **two terminals** in
`code/p22-greeter-operator`. In terminal 1:

```bash
cargo run
```

```text
     Running `target/debug/p22-greeter-operator`
greeter-operator started
```

In terminal 2, copy the Greeting `hello` from Lesson 22.26 and apply it:

```bash
cp ../p22-reconcile-sim/k8s/hello.yaml k8s/
kubectl apply -f k8s/hello.yaml
kubectl get configmaps
kubectl get greeting hello -o jsonpath='{.status}'; echo
```

```text
greeting.course.example.com/hello created
NAME               DATA   AGE
hello-greeting     1      3s
kube-root-ca.crt   1      6m18s
{"configMap":"hello-greeting"}
```

The ConfigMap exists and the status says so. Look at the ConfigMap with `kubectl get configmap hello-greeting -o yaml`:

```text
apiVersion: v1
data:
  message: Hello from a custom resource
kind: ConfigMap
metadata:
  creationTimestamp: "2026-11-18T07:50:06Z"
  name: hello-greeting
  namespace: default
  ownerReferences:
  - apiVersion: course.example.com/v1
    controller: true
    kind: Greeting
    name: hello
    uid: 2109072e-b11e-4ee3-b9c2-0638eed3357b
  resourceVersion: "1067"
  uid: a89b332d-f0d8-4e8b-a707-7610526e00e2
```

Now change the wish, break things, and delete:

```bash
kubectl patch greeting hello --type merge -p '{"spec":{"shout":true}}'
kubectl get configmap hello-greeting -o jsonpath='{.data.message}'; echo
kubectl delete configmap hello-greeting
kubectl get configmaps
kubectl delete greeting hello
kubectl get configmaps
```

```text
greeting.course.example.com/hello patched
HELLO FROM A CUSTOM RESOURCE
configmap "hello-greeting" deleted from default namespace
NAME               DATA   AGE
hello-greeting     1      2s
kube-root-ca.crt   1      6m22s
greeting.course.example.com "hello" deleted from default namespace
NAME               DATA   AGE
kube-root-ca.crt   1      6m25s
```

Update, self-healing (the ConfigMap is 2 seconds old: it was created again), and garbage collection all work. Press
Ctrl+C in terminal 1. Its full output:

```text
greeter-operator started
reconcile Greeting default/hello
done: hello
reconcile Greeting default/hello
done: hello
reconcile Greeting default/hello
done: hello
reconcile Greeting default/hello
done: hello
reconcile Greeting default/hello
done: hello
reconcile Greeting default/hello
done: hello
controller error: tried to reconcile object Greeting.v1.course.example.com/hello.default that was not found in local store
greeter-operator stopped
```

**What happens:**

1. There are more reconciles than commands. Our own writes cause watch events too: creating the ConfigMap (an owned
   object) and writing the status (a change of the Greeting) each trigger another reconcile. That one finds nothing
   to change, and the API server ignores writes that change nothing, so the loop calms down. This is why idempotency
   is not optional.
2. The `controller error` line comes after the delete: the garbage collector deleted the ConfigMap, the controller
   wanted to reconcile its owner, but the owner was already gone from the store. It is harmless here.
3. `greeter-operator stopped` shows the clean shutdown after Ctrl+C.

## Common mistakes

> ⚠️ **Common mistake: starting the operator before the CRD is installed.** The watcher cannot list Greetings and
> retries forever:
>
> ```text
> controller error: event queue error: failed to perform initial object list: ApiError: 404 page not found: Failed to parse error data (Status { status: Some(Failure), code: 404, message: "404 page not found", metadata: None, reason: "Failed to parse error data", details: None })
> ```
>
> Run `kubectl apply -f k8s/greeting-crd.yaml` first. The `404` means "this URL (the Greeting API) does not exist".

> ⚠️ **Common mistake: no owner reference.** If you remove `owner_references` from `desired_config_map`, everything
> seems to work, but deleting a Greeting leaves its ConfigMap behind forever, and `.owns()` cannot map a ConfigMap
> change back to its Greeting.

> ⚠️ **Common mistake: forgetting `.owns(...)`.** Without it, a ConfigMap deleted by hand comes back only at the next
> requeue, five minutes later. The owner reference is for deletion; `.owns()` is for watching.

## Try it yourself

1. Start the operator in terminal 1 (`cargo run`).
2. In terminal 2, create a second namespace and a Greeting in it: `kubectl create namespace demo`, then
   `kubectl apply -n demo -f k8s/hello.yaml`.
3. `kubectl get configmaps -n demo` shows `hello-greeting`: one operator serves all namespaces.
4. `kubectl edit configmap hello-greeting -n demo`, change the message and save. Within a second
   `kubectl get configmap hello-greeting -n demo -o jsonpath='{.data.message}'` shows the original message again:
   `.force()` and `.owns()` at work.
5. Clean up with `kubectl delete namespace demo` (this deletes everything in it) and stop the operator.

## Exercises

### Exercise 1 — One more key (easy)

Add a second key `length` to the ConfigMap data: the number of characters of the stored text, as a string. Run the
operator and check it with `kubectl get configmap hello-greeting -o jsonpath='{.data}'`.

<details>
<summary>Hint</summary>

ConfigMap data values are always strings. `text.chars().count()` counts characters; `.to_string()` turns the number
into a `String`. Compute it before `text` is moved into the map.

</details>

<details>
<summary>Answer</summary>

In `desired_config_map`:

```rust
        data: Some(BTreeMap::from([
            ("length".to_string(), text.chars().count().to_string()),
            ("message".to_string(), text),
        ])),
```

```text
{"length":"28","message":"Hello from a custom resource"}
```

Why it works: the array is built from left to right, so `text` is still available when the length is computed, and
moved into the map only in the second pair. Server-side apply updates the existing ConfigMap on the next reconcile.

</details>

### Exercise 2 — Unit tests without a cluster (medium)

Add a `#[cfg(test)] mod tests` to `src/lib.rs` with three tests for `desired_config_map`: the name and namespace are
right and the message is copied; `shout` gives capital letters; the owner reference points at the Greeting
(kind, name, uid, `controller: true`). Run `cargo test --lib`.

<details>
<summary>Hint</summary>

`Greeting::new("hello", spec)` creates an object with an empty `metadata` except the name. Set
`greeting.metadata.namespace` and `greeting.metadata.uid` yourself: without a uid there is no owner reference.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn greeting(message: &str, shout: bool) -> Greeting {
        let spec = GreetingSpec {
            message: message.to_string(),
            shout,
        };
        let mut greeting = Greeting::new("hello", spec);
        greeting.metadata.namespace = Some("default".to_string());
        greeting.metadata.uid = Some("1234".to_string());
        greeting
    }

    #[test]
    fn config_map_has_name_namespace_and_message() {
        let cm = desired_config_map(&greeting("Hi", false));
        assert_eq!(cm.metadata.name.as_deref(), Some("hello-greeting"));
        assert_eq!(cm.metadata.namespace.as_deref(), Some("default"));
        assert_eq!(cm.data.unwrap()["message"], "Hi");
    }

    #[test]
    fn shout_makes_capital_letters() {
        let cm = desired_config_map(&greeting("Hi", true));
        assert_eq!(cm.data.unwrap()["message"], "HI");
    }

    #[test]
    fn config_map_is_owned_by_the_greeting() {
        let cm = desired_config_map(&greeting("Hi", false));
        let owners = cm
            .metadata
            .owner_references
            .expect("owner reference is set");
        assert_eq!(owners.len(), 1);
        assert_eq!(owners[0].kind, "Greeting");
        assert_eq!(owners[0].name, "hello");
        assert_eq!(owners[0].uid, "1234");
        assert_eq!(owners[0].controller, Some(true));
    }
}
```

```text
running 3 tests
test tests::shout_makes_capital_letters ... ok
test tests::config_map_has_name_namespace_and_message ... ok
test tests::config_map_is_owned_by_the_greeting ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: `desired_config_map` is a pure function, so a test only builds a Greeting in memory and checks the
result. These tests run in milliseconds and need no cluster. (Tests may print in a different order: they run in
parallel.)

</details>

### Exercise 3 — Watch one namespace (hard)

Teams often run one operator per namespace. Make the operator read the environment variable `WATCH_NAMESPACE`: if it
is set, watch Greetings and ConfigMaps only in that namespace and print `watching namespace <ns>`; otherwise watch
all namespaces as before. Check it: run with `WATCH_NAMESPACE=default cargo run`, create a namespace `other` and apply
`hello.yaml` in both; only `default` gets a ConfigMap.

<details>
<summary>Hint</summary>

`std::env::var("WATCH_NAMESPACE")` returns a `Result`. Both arms of a `match` must produce the same types, so return
a tuple `(Api<Greeting>, Api<ConfigMap>)` from each arm.

</details>

<details>
<summary>Answer</summary>

In `main`, replace the two `Api::all` lines:

```rust
    let (greetings, config_maps) = match std::env::var("WATCH_NAMESPACE") {
        Ok(ns) => {
            println!("watching namespace {ns}");
            let greetings: Api<Greeting> = Api::namespaced(client.clone(), &ns);
            let config_maps: Api<ConfigMap> = Api::namespaced(client.clone(), &ns);
            (greetings, config_maps)
        }
        Err(_) => {
            println!("watching all namespaces");
            (Api::all(client.clone()), Api::all(client.clone()))
        }
    };
```

After `kubectl create namespace other` and applying `hello.yaml` in both namespaces,
`kubectl get configmaps -n other` lists only `kube-root-ca.crt`. The operator printed `watching namespace default` and reconciled only `default/hello`. Clean up with
`kubectl delete namespace other` and `kubectl delete greeting hello`.

Why it works: an `Api` made with `namespaced` sends its list and watch requests to the URL (web address) of one namespace, so
Greetings in other namespaces never reach the controller. In Lesson 22.28 you will see the security side: such an
operator needs permissions in one namespace only.

</details>

## Quiz

1. You wrote only `GreetingSpec`. Where does the struct `Greeting` come from, and what does it contain?

<details>
<summary>Answer</summary>

`#[derive(CustomResource)]` generates it. It contains `metadata` (an `ObjectMeta`), your `spec` and, because of
`status = "GreetingStatus"`, an `Option<GreetingStatus>`. It also implements `Resource` and has `crd()` and `new()`.

</details>

2. Why does the operator use server-side apply instead of "get the ConfigMap, compare, then create or update"?

<details>
<summary>Answer</summary>

One apply call is idempotent: create, update or no change, decided by the API server. Get-compare-write takes several
calls and can race with other writers between the read and the write. SSA also records the operator as the field
manager of the fields it set.

</details>

3. Who deletes `hello-greeting` when you delete the Greeting `hello`, and does the operator need to run for that?

<details>
<summary>Answer</summary>

The Kubernetes garbage collector, because the ConfigMap has an owner reference to the Greeting. The operator does not
need to run.

</details>

4. One `kubectl apply` caused several reconciles of `hello`. Is that a bug?

<details>
<summary>Answer</summary>

No. The operator's own writes (the ConfigMap, the status) cause watch events too. Because reconcile is idempotent,
the extra runs change nothing and the loop settles.

</details>

## Summary

- `#[derive(CustomResource)]` on a `…Spec` struct generates the full resource type and its CRD; a small `crdgen`
  binary prints the CRD as YAML, so Rust is the single source of truth.
- `Client::try_default()` connects through your kubeconfig; `Api::namespaced` and `Api::all` give typed handles.
- Child objects carry an **owner reference**; the garbage collector deletes them with their owner.
- Reconcile builds the desired object and writes it with **server-side apply** (idempotent), then reports in the
  **status** subresource with a merge patch, and returns an `Action` (for example `requeue`).
- `Controller::new(...).owns(...).run(reconcile, error_policy, ctx)` is the control loop: watches, a queue, no parallel
  reconciles of one object, retries through the error policy.
- Extra reconciles caused by your own writes are normal; idempotency keeps them harmless.

## Sources and further reading

- [kube.rs — The Application](https://kube.rs/controllers/application/) — the official step-by-step guide to a kube-rs controller.
- [kube.rs — The Reconciler](https://kube.rs/controllers/reconciler/) — how to think about reconcile, actions and errors.
- [docs.rs — kube 5.1](https://docs.rs/kube/5.1.0/kube/) — API reference: `Client`, `Api`, `CustomResource`, `runtime::Controller`.
- [Kubernetes — Server-Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/) — field managers, conflicts and forcing.
- [Kubernetes — Owners and Dependents](https://kubernetes.io/docs/concepts/overview/working-with-objects/owners-dependents/) — owner references and their rules.
- [Kubernetes — Garbage Collection](https://kubernetes.io/docs/concepts/architecture/garbage-collection/) — how dependents are deleted.
- [kube-rs/controller-rs](https://github.com/kube-rs/controller-rs) — a complete example operator by the kube-rs maintainers.

---

[🏠 Start](../START-HERE.md) · [← 22.26 Cloud-native track: Kubernetes operators](26-cloud-native-track-kubernetes-operators.md) · [📋 Phase 22](README.md) · [22.28 Cloud-native track: testing and running operators →](28-cloud-native-track-testing-and-running-operators.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
