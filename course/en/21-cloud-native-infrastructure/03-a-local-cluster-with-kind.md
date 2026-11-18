<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 21.2 Kubernetes concepts: pods, Deployments and Services](02-kubernetes-concepts.md) · [📋 Phase 21](README.md) · [21.4 Deploying a Rust service to Kubernetes →](04-deploying-a-rust-service-to-kubernetes.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/21-cloud-native-infrastructure/03-a-local-cluster-with-kind.md) <!-- nav -->

# Lesson 21.3 — A local cluster with kind and kubectl

> **Phase 21 — Cloud-native infrastructure** · Lesson 3 of 14 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- How to install kind and kubectl on your Mac and create a real Kubernetes cluster inside Docker.
- What a kubeconfig file and a context are, and how kubectl knows which cluster to talk to.
- The everyday kubectl commands: `get`, `describe`, `logs`, `exec`, `delete`, `scale`, `expose`, `port-forward`, `run`.
- How to watch the controllers from Lesson 21.2 replace a deleted pod.
- How to move from typed commands to manifest files with `--dry-run=client -o yaml`, `apply` and `diff`.

## Before you start

- You finished [Lesson 21.2 — Kubernetes concepts](02-kubernetes-concepts.md).
- Docker Desktop is installed and running ([Lesson 11.12](../11-linux-shell-and-containers/12-containers-and-docker-basics.md)), with at least 4 GB (gigabytes) of memory in *Settings → Resources*.
- You know Homebrew, the package manager for macOS ([Lesson 00.5](../00-getting-started/05-install-rust-and-rustrover.md)), and strict bash scripts ([Lesson 11.9 — Robust bash scripts](../11-linux-shell-and-containers/09-robust-bash-scripts.md)).

## New words

| Word | Meaning |
|---|---|
| **kind** (Kubernetes IN Docker) | A tool that creates a Kubernetes cluster whose nodes are Docker containers. Made by the Kubernetes project for testing. |
| **kubectl** | The command-line client for Kubernetes. It sends your requests to the API (Application Programming Interface) server. People say "kube-control" or "kube-C-T-L". |
| **kubeconfig** | A file (by default `~/.kube/config`) that lists clusters, users with their credentials, and contexts. |
| **context** | A named pair "this cluster + this user (+ optionally a namespace)" in a kubeconfig. The *current context* decides where kubectl sends commands. |
| **event** | A short record that a component writes about an object, such as "Pulling image" or "Failed". `kubectl describe` shows them. |
| **port-forward** | A kubectl feature that opens a port on your computer and tunnels it through the API server to a pod or Service. |
| **ImagePullBackOff** | A pod status: the node could not download the image and waits longer and longer before trying again. |
| **dry run** | Running a command without making the change, to see what it would do. |
| **smoke test** | A very quick test that checks that something works at all ("does it answer?") before deeper tests. |

## 1. Install kind and kubectl

Install both tools with Homebrew:

```bash
brew install kind kubectl
```

Then check them. Your versions may be newer than the ones used to write this lesson:

```bash
kind version
kubectl version --client
```

```text
kind v0.33.0 go1.26.7 darwin/arm64
Client Version: v1.37.2
Kustomize Version: v5.8.1
```

`kubectl version --client` prints only the client version; without `--client` it would also ask a cluster for its version, and we have none yet. Kustomize is a tool built into kubectl; you will use it in Lesson 21.6.

> 💡 **Tip:** without Homebrew, you can download both programs from their official pages (links at the end): for kind, the file `kind-darwin-arm64` from the release page; for kubectl, `https://dl.k8s.io/release/v1.37.2/bin/darwin/arm64/kubectl`. Each has a `.sha256` file next to it. Compare the checksum (`shasum -a 256 <file>`) before you make the file executable and put it into a folder on your `PATH`.

## 2. Create a cluster

Make sure Docker Desktop is running, then:

```bash
kind create cluster --name course-21-03
```

```text
Creating cluster "course-21-03" ...
 ✓ Ensuring node image (kindest/node:v1.37.0) 🖼️
 ✓ Preparing nodes 📦 
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-course-21-03"
You can now use your cluster with:

kubectl cluster-info --context kind-course-21-03

Not sure what to do next? 😅  Check out https://kind.sigs.k8s.io/docs/user/quick-start/
```

(Each step first prints a line with `•` and then replaces it with `✓`; we show only the final lines. The last line is a random friendly message.) The first run downloads the node image (more than 1 GB on disk), so it can take a few minutes. Later runs take 15–30 seconds.

**What happens:**

1. kind starts **one Docker container** from the image `kindest/node`. Inside it run systemd, containerd and the kubelet; it is a whole Kubernetes node.
2. It starts the control plane in that container (API server, etcd, scheduler, controller manager), installs the network plug-in (CNI, Container Network Interface) and a simple storage provider.
3. It writes the address and credentials of the new cluster into your **kubeconfig** and makes it the current context.

The node is just a container. Docker shows it (we filter by name so that you see only course objects):

```bash
docker ps --filter name=course-21-03 --format 'table {{.Names}}\t{{.Image}}\t{{.Ports}}'
```

```text
NAMES                        IMAGE                  PORTS
course-21-03-control-plane   kindest/node:v1.37.0   127.0.0.1:62668->6443/tcp
```

The API server listens on port 6443 inside the container, and Docker publishes it on a random port of your Mac, only on `127.0.0.1`. Your port number will differ.

## 3. kubeconfig and contexts

kubectl itself knows nothing about clusters. Every time it runs, it reads the **kubeconfig** file (`~/.kube/config`, unless the environment variable `KUBECONFIG` points elsewhere) and uses the **current context**:

```bash
kubectl config current-context
kubectl config get-contexts
```

```text
kind-course-21-03
CURRENT   NAME                CLUSTER             AUTHINFO            NAMESPACE
*         kind-course-21-03   kind-course-21-03   kind-course-21-03   
```

A context joins a **cluster** (an address and the certificate of the API server) with an **AuthInfo** (the user and their credentials). If you later work with several clusters, you will see several lines here and switch with `kubectl config use-context <name>`. The `*` marks the current one.

```bash
kubectl cluster-info
```

```text
Kubernetes control plane is running at https://127.0.0.1:62668
CoreDNS is running at https://127.0.0.1:62668/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

> 🔒 **Security:** a kubeconfig contains credentials. The one kind writes gives **full administrator rights** to its cluster. Never commit a kubeconfig to Git, never paste it into a chat, and never copy one from a real cluster to a shared computer. For kind it does not matter much (the cluster lives only on your Mac and is gone when you delete it), but the habit matters.

The node needs a few seconds after creation to become ready. Wait for it, then look at it:

```bash
kubectl wait --for=condition=Ready node --all --timeout=120s
kubectl get nodes -o wide
```

```text
node/course-21-03-control-plane condition met
NAME                         STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                       KERNEL-VERSION            CONTAINER-RUNTIME
course-21-03-control-plane   Ready    control-plane   21s   v1.37.0   172.30.0.3    <none>        Debian GNU/Linux 13 (trixie)   7.0.14-linuxkit (arm64)   containerd://2.3.4
```

`-o wide` adds columns. The node runs Debian 13 inside the Linux virtual machine of Docker Desktop (`linuxkit` kernel) on arm64, with containerd as the container runtime: exactly the picture from Lesson 21.2.

Every `kubectl get` follows the same pattern: `kubectl get <kind> [name] [options]`. Try `kubectl get namespaces`:

```text
NAME                 STATUS   AGE
default              Active   22s
kube-node-lease      Active   22s
kube-public          Active   22s
kube-system          Active   22s
local-path-storage   Active   18s
```

## 4. Your first Deployment

For exploring, kubectl has *imperative* shortcuts that create objects without a file. Create a Deployment with two copies of nginx:

```bash
kubectl create deployment web --image=nginx:1.30-alpine --replicas=2
kubectl rollout status deployment/web
```

```text
deployment.apps/web created
Waiting for deployment "web" rollout to finish: 0 of 2 updated replicas are available...
Waiting for deployment "web" rollout to finish: 1 of 2 updated replicas are available...
deployment "web" successfully rolled out
```

`kubectl rollout status` waits until the Deployment reaches its desired state, which is handy in scripts. Now list the pods:

```bash
kubectl get pods -o wide
```

```text
NAME                   READY   STATUS    RESTARTS   AGE   IP           NODE                         NOMINATED NODE   READINESS GATES
web-6f5d6d9c94-wdj7t   1/1     Running   0          30s   10.244.0.6   course-21-03-control-plane   <none>           <none>
web-6f5d6d9c94-zxxt8   1/1     Running   0          30s   10.244.0.5   course-21-03-control-plane   <none>           <none>
```

`READY 1/1` means one of one containers is ready. Each pod has its own IP (Internet Protocol) address. Your pod names will differ (the suffixes are random).

### Looking inside: describe, logs, exec

`kubectl describe` prints everything about one object, including the **events** that the components wrote about it. Use your own pod name:

```bash
kubectl describe pod web-6f5d6d9c94-wdj7t
```

```text
Name:             web-6f5d6d9c94-wdj7t
Namespace:        default
Node:             course-21-03-control-plane/172.30.0.3
Labels:           app=web
                  pod-template-hash=6f5d6d9c94
Status:           Running
IP:               10.244.0.6
Controlled By:  ReplicaSet/web-6f5d6d9c94
Containers:
  nginx:
    Image:          nginx:1.30-alpine
    State:          Running
    Ready:          True
    Restart Count:  0
...
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  30s   default-scheduler  Successfully assigned default/web-6f5d6d9c94-wdj7t to course-21-03-control-plane
  Normal  Pulling    30s   kubelet            spec.containers{nginx}: Pulling image "nginx:1.30-alpine"
  Normal  Pulled     0s    kubelet            spec.containers{nginx}: Successfully pulled image "nginx:1.30-alpine" in 1.354s (29.467s including waiting). Image size: 25987805 bytes.
  Normal  Created    0s    kubelet            spec.containers{nginx}: Container created
  Normal  Started    0s    kubelet            spec.containers{nginx}: Container started
```

(Shortened: the real output also lists conditions, volumes and more.) Read the events from top to bottom and you see Lesson 21.2 happen: the **scheduler** assigned the pod to the node, then the **kubelet** pulled the image, created the container and started it. `Controlled By: ReplicaSet/...` is the owner reference. When a pod does not start, the events are the first place to look.

`kubectl logs` shows what the container wrote to standard output. With `deployment/web` kubectl picks one pod of the Deployment for you:

```bash
kubectl logs deployment/web --tail=3
```

```text
Found 2 pods, using pod/web-6f5d6d9c94-zxxt8
2026/11/18 08:06:47 [notice] 1#1: start worker process 42
2026/11/18 08:06:47 [notice] 1#1: start worker process 43
2026/11/18 08:06:47 [notice] 1#1: start worker process 44
```

`kubectl exec` runs a command inside a running container, like `docker exec`:

```bash
kubectl exec deployment/web -- nginx -v
```

```text
nginx version: nginx/1.30.6
```

Everything after `--` is the command for the container. Use `exec` to look, not to change: remember the cattle rule from Lesson 21.1.

## 5. Watch the controllers work

Now the most important experiment of the lesson. Delete one pod (use one of your pod names):

```bash
kubectl delete pod web-6f5d6d9c94-wdj7t
kubectl get pods
```

```text
pod "web-6f5d6d9c94-wdj7t" deleted from default namespace
NAME                   READY   STATUS    RESTARTS   AGE
web-6f5d6d9c94-jczm7   1/1     Running   0          2s
web-6f5d6d9c94-zxxt8   1/1     Running   0          32s
```

There are still two pods. The deleted one is gone, and a **new** pod (`jczm7`, 2 seconds old) took its place. The ReplicaSet controller saw "desired 2, actual 1" and acted: the reconciliation loop from Lesson 21.1, working for real.

> 💡 **Tip:** to watch changes live, run `kubectl get pods --watch` (or `-w`) in a second terminal tab before you delete the pod. Every change prints a new line; stop it with Ctrl+C.

Change the desired state with `scale`:

```bash
kubectl scale deployment web --replicas=4
kubectl rollout status deployment/web
kubectl get deployment web
```

```text
deployment.apps/web scaled
Waiting for deployment "web" rollout to finish: 2 of 4 updated replicas are available...
Waiting for deployment "web" rollout to finish: 3 of 4 updated replicas are available...
deployment "web" successfully rolled out
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
web    4/4     4            4           33s
```

`READY 4/4`: four of four pods are ready. `scale` only changed the number `spec.replicas` in the Deployment; the controllers did the rest.

## 6. Reach the pods

The pods have IP addresses inside the cluster only. To reach them, first give them a Service. `kubectl expose` creates one that selects the Deployment's pods:

```bash
kubectl expose deployment web --port=80
kubectl get service web
```

```text
service/web exposed
NAME   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
web    ClusterIP   10.96.27.137   <none>        80/TCP    0s
```

A ClusterIP address is not reachable from your Mac. **Port-forwarding** opens a local port and tunnels it through the API server to the Service. Run it in a **second terminal tab**, because it keeps running until you press Ctrl+C:

```bash
kubectl port-forward service/web 8088:80
```

```text
Forwarding from 127.0.0.1:8088 -> 80
Forwarding from [::1]:8088 -> 80
```

`8088:80` means "local port 8088 to port 80 of the Service". In the first tab:

```bash
curl -s http://localhost:8088 | head -4
```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
```

Port-forwarding is a tool for developers. It picks **one** pod behind the Service and keeps using it, so it is not a load balancer, and it stops when that pod goes away. Stop it now with Ctrl+C in the second tab.

The other way is to test from **inside** the cluster with a temporary pod. `kubectl run` creates a single pod; `--rm -i --restart=Never` makes it run once, show its output and be deleted; `-q` hides kubectl's own messages:

```bash
kubectl run client -q --rm -i --restart=Never --image=busybox:1.37 -- wget -qO- http://web | grep title
```

```text
<title>Welcome to nginx!</title>
```

`busybox` is a small official image with basic Unix tools, including `wget`. Inside the cluster the name `web` resolves to the Service (Lesson 21.2).

## 7. From commands to files

Imperative commands are good for exploring, but the real way to work is declarative: manifests in Git, applied with `kubectl apply`. You do not have to write the first manifest from memory. Delete what you made, then let kubectl write the YAML (YAML Ain't Markup Language, the format of manifests) for you:

```bash
kubectl delete deployment,service web
kubectl create deployment web --image=nginx:1.30-alpine --replicas=2 --dry-run=client -o yaml > web.yaml
cat web.yaml
```

```text
deployment.apps "web" deleted from default namespace
service "web" deleted from default namespace
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: web
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  strategy: {}
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - image: nginx:1.30-alpine
        name: nginx
        resources: {}
status: {}
```

`--dry-run=client` means "build the object but do not send it"; `-o yaml` prints it. The empty `strategy: {}`, `resources: {}` and `status: {}` can be deleted; they are placeholders. (kubectl writes list items without extra indentation under their key; both styles are valid YAML.)

Now apply the file:

```bash
kubectl apply -f web.yaml
```

```text
deployment.apps/web created
```

Change `replicas: 2` to `replicas: 3` in `web.yaml`. Before you apply a change, ask the cluster what *would* change:

```bash
kubectl diff -f web.yaml
```

```text
@@ -6,7 +6,7 @@
...
-  generation: 1
+  generation: 2
...
@@ -15,7 +15,7 @@
 spec:
   progressDeadlineSeconds: 600
-  replicas: 2
+  replicas: 3
   revisionHistoryLimit: 10
```

(Shortened: the real output starts with the names of two temporary files, and shows a few unchanged lines around each change.) `diff` compares the object in the cluster with the object your file would produce. Lines with `-` go away, lines with `+` arrive. `generation` is a counter of changes to `spec`. Now apply, twice:

```bash
kubectl apply -f web.yaml
kubectl apply -f web.yaml
```

```text
deployment.apps/web configured
deployment.apps/web unchanged
```

The second `apply` changes nothing: `apply` is **idempotent**, the property from Lesson 21.1. This is why manifests in Git plus `kubectl apply` is a safe way to work: you can apply the whole folder again at any time.

> 🧪 **Testing:** `kubectl diff` exits with code `1` when there are differences and `0` when there are none. A CI (Continuous Integration) job can run it to check that the cluster still matches Git (this is called *drift detection*), and the `--dry-run=server` option of `apply` asks the API server to validate a manifest without saving it.

## 8. Clean up

Delete the objects, then the whole cluster:

```bash
kubectl delete -f web.yaml
kind delete cluster --name course-21-03
```

```text
deployment.apps "web" deleted from default namespace
Deleting cluster "course-21-03" ...
Deleted nodes: ["course-21-03-control-plane"]
```

`kind delete cluster` removes the node container and the cluster's entry from your kubeconfig. Your `web.yaml` stays; you can create a new cluster and apply it again in a minute. That is the reason to keep everything in files.

## Common mistakes

**No cluster, or the wrong one.** If kubectl has no context (no cluster was created, or it was deleted), it falls back to `localhost:8080`, where nothing listens:

```text
E1118 13:01:11.694393   15979 memcache.go:381] "Couldn't get current server API group list" err="Get \"http://localhost:8080/api?timeout=32s\": dial tcp [::1]:8080: connect: connection refused"
The connection to the server localhost:8080 was refused - did you specify the right host or port?
```

(The first line appears several times.) Fix: create the cluster, or check `kubectl config current-context`. The opposite mistake is worse: with several contexts, a command meant for your test cluster goes to another cluster. Always look at the current context before `delete` or `apply`.

**The wrong namespace.** `kubectl get pods` shows only the namespace of the current context (here `default`). In a fresh cluster it prints `No resources found in default namespace.`, although the system pods run in `kube-system`. Use `kubectl get pods -n kube-system`, or `-A` for all namespaces.

**Forgetting that port-forward is still running.** A second `kubectl port-forward` on the same local port fails because the port is in use. Stop the old one with Ctrl+C.

## Try it yourself

1. Create a cluster `course-21-03` again and apply your `web.yaml`.
2. Open a second tab with `kubectl get pods --watch`. In the first tab, run `kubectl scale deployment web --replicas=1` and then `--replicas=3`. Watch the lines appear.
3. Delete a pod while the watch runs, and count how many seconds it takes until a new pod is `Running`. Then clean up: `kind delete cluster --name course-21-03`.

## Exercises

### Exercise 1 — ImagePullBackOff (easy)

Create a Deployment with a typo in the tag: `kubectl create deployment broken --image=nginx:1.30-alpin`. Find out with kubectl *why* the pod does not start, then fix the Deployment without deleting it.

<details>
<summary>Hint</summary>

`kubectl get pods -l app=broken` shows the status; `kubectl describe pod <name>` shows the events. `kubectl set image deployment/broken <container>=<image>` changes the image; the container name is `nginx` (kubectl took it from the image name).

</details>

<details>
<summary>Answer</summary>

```bash
kubectl get pods -l app=broken
```

```text
NAME                      READY   STATUS             RESTARTS   AGE
broken-6f597897d6-r24v9   0/1     ImagePullBackOff   0          25s
```

The events of `kubectl describe pod broken-6f597897d6-r24v9` end with:

```text
  Warning  Failed     11s (x2 over 23s)  kubelet            spec.containers{nginx}: Failed to pull image "nginx:1.30-alpin": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/library/nginx:1.30-alpin": failed to resolve reference "docker.io/library/nginx:1.30-alpin": docker.io/library/nginx:1.30-alpin: not found
  Warning  Failed     11s (x2 over 23s)  kubelet            spec.containers{nginx}: Error: ErrImagePull
```

The tag `1.30-alpin` does not exist. Fix and watch the rollout:

```bash
kubectl set image deployment/broken nginx=nginx:1.30-alpine
kubectl rollout status deployment/broken
kubectl delete deployment broken
```

```text
deployment.apps/broken image updated
Waiting for deployment "broken" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "broken" rollout to finish: 1 old replicas are pending termination...
Waiting for deployment "broken" rollout to finish: 1 old replicas are pending termination...
deployment "broken" successfully rolled out
deployment.apps "broken" deleted from default namespace
```

`ErrImagePull` is the first failure; `ImagePullBackOff` means the kubelet now waits longer and longer (10 s, 20 s, 40 s, … up to 5 minutes) between tries. Changing the image created a new ReplicaSet with a working template.

</details>

### Exercise 2 — A smoke test script (medium)

Write `smoke.sh`: a strict bash script that checks that the Service `web` answers inside the cluster with a page that contains `Welcome to nginx`. It must exit with `0` when everything is fine and with a non-zero code (and a message on standard error) when not. Test it twice: once with the Service present, once after `kubectl delete service web`.

<details>
<summary>Hint</summary>

Use `set -euo pipefail` (Lesson 11.9), run the temporary pod with `kubectl run ... -q --rm -i --restart=Never --image=busybox:1.37 -- sh -c 'sleep 2; wget -qO- -T 5 http://web'`, and check the text with `grep -q`. `kubectl run` exits with the exit code of the pod's command. The `sleep 2` matters: `kubectl run -i` attaches to the pod only after it has started, and output printed before that moment can be lost.

</details>

<details>
<summary>Answer</summary>

```bash
#!/usr/bin/env bash
# smoke.sh: does the Service "web" answer inside the cluster?
set -euo pipefail

# "sleep 2" gives kubectl time to attach before wget prints anything.
if ! body=$(kubectl run smoke -q --rm -i --restart=Never --image=busybox:1.37 \
    -- sh -c 'sleep 2; wget -qO- -T 5 http://web' 2>/dev/null); then
  echo "smoke: the service did not answer" >&2
  exit 1
fi

if ! grep -q 'Welcome to nginx' <<<"$body"; then
  echo "smoke: unexpected page" >&2
  exit 1
fi
echo "smoke: ok"
```

With the Service (after `kubectl expose deployment web --port=80`):

```text
smoke: ok
```

After `kubectl delete service web`, the name `web` no longer exists, `wget` fails, and the script prints `smoke: the service did not answer` and exits with `1`.

Why it works: `kubectl run --rm -i` waits for the pod and returns the exit code of the command (`sh`, which returns the exit code of `wget`); `if ! body=$(...)` catches a failure without `set -e` stopping the script first. The timeout `-T 5` keeps the test fast. Lesson 21.7 runs a test like this in CI after every deployment.

</details>

### Exercise 3 — Count pods by phase in Rust (hard)

Write a Rust program `code/p21-podstat` that runs

```text
kubectl get pods --no-headers -o custom-columns=NAME:.metadata.name,PHASE:.status.phase
```

and prints how many pods are in each phase (`Running`, `Pending`, …), sorted by phase name. Put the parsing into a function `count_phases(output: &str) -> BTreeMap<String, usize>` and write a unit test for it that needs no cluster.

<details>
<summary>Hint</summary>

`std::process::Command::new("kubectl").args([...]).output()` runs a program and collects its output (you used `Command` in Lesson 10.5; Lesson 11.5 explains exit codes). `output.status.success()` tells you whether it worked. `*map.entry(key).or_insert(0) += 1` counts.

</details>

<details>
<summary>Answer</summary>

```rust
use std::collections::BTreeMap;
use std::process::Command;

/// Counts pods per phase from the output of
/// `kubectl get pods --no-headers -o custom-columns=NAME:.metadata.name,PHASE:.status.phase`.
fn count_phases(output: &str) -> BTreeMap<String, usize> {
    let mut counts = BTreeMap::new();
    for line in output.lines() {
        let mut columns = line.split_whitespace();
        if let (Some(_name), Some(phase)) = (columns.next(), columns.next()) {
            *counts.entry(phase.to_string()).or_insert(0) += 1;
        }
    }
    counts
}

fn main() {
    let output = Command::new("kubectl")
        .args([
            "get",
            "pods",
            "--no-headers",
            "-o",
            "custom-columns=NAME:.metadata.name,PHASE:.status.phase",
        ])
        .output()
        .expect("could not start kubectl; is it installed?");
    if !output.status.success() {
        eprintln!("kubectl failed: {}", String::from_utf8_lossy(&output.stderr));
        std::process::exit(1);
    }
    let text = String::from_utf8_lossy(&output.stdout);
    for (phase, count) in count_phases(&text) {
        println!("{phase}: {count}");
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn counts_each_phase() {
        let output = "web-1   Running\nweb-2   Running\nweb-3   Pending\n";
        let counts = count_phases(output);
        assert_eq!(counts["Running"], 2);
        assert_eq!(counts["Pending"], 1);
        assert_eq!(counts.len(), 2);
    }

    #[test]
    fn ignores_empty_lines() {
        assert!(count_phases("\n\n").is_empty());
    }
}
```

With `web` (3 replicas) and the broken Deployment of Exercise 1 still failing, `cargo run` printed:

```text
Pending: 1
Running: 3
```

and `cargo test` printed `test result: ok. 2 passed`. A pod in `ImagePullBackOff` has the phase `Pending`: the phase is a coarse summary, and the detailed reason is in the container status.

Why it works: the program does not talk to Kubernetes itself; it reuses kubectl, which already knows the kubeconfig and the context. The parsing is a pure function, so the test needs no cluster (the "decide versus act" split of Lesson 21.1). For a real tool you would use the `kube` crate, which talks to the API server directly.

</details>

## Quiz

1. What is a kind "node", physically, on your Mac?

<details>
<summary>Answer</summary>

A Docker container (here `course-21-03-control-plane`) started from the image `kindest/node`, running systemd, containerd, the kubelet and the control-plane components.

</details>

2. How does kubectl know which cluster to talk to?

<details>
<summary>Answer</summary>

It reads the kubeconfig (`~/.kube/config` or the file in `KUBECONFIG`) and uses the current context, which names a cluster (address and the certificate of the CA, the Certificate Authority that signed the API server's certificate) and a user (credentials).

</details>

3. Which command shows *why* a pod does not start, and which part of its output do you read first?

<details>
<summary>Answer</summary>

`kubectl describe pod <name>`, and first the `Events` section at the end.

</details>

4. You delete a pod of a Deployment with 2 replicas. What happens, and which component does it?

<details>
<summary>Answer</summary>

A new pod with a new name is created within seconds, because the ReplicaSet controller sees 1 pod where 2 are desired. The scheduler then places it and the kubelet starts it.

</details>

5. Why is `kubectl port-forward` not a way to serve real users?

<details>
<summary>Answer</summary>

It is a debugging tunnel through the API server from your computer: it uses one pod only (no load balancing), stops when that pod goes away or when you close it, and runs only as long as your terminal does.

</details>

6. You run `kubectl apply -f web.yaml` twice without changing the file. What does the second run print, and what property does this show?

<details>
<summary>Answer</summary>

`deployment.apps/web unchanged`. It shows that `apply` is idempotent: applying the same desired state again changes nothing.

</details>

## Summary

- kind runs a whole Kubernetes cluster in Docker containers; `kind create cluster --name …` and `kind delete cluster --name …` take seconds.
- kubectl reads the kubeconfig and uses the current context. A kubeconfig holds credentials: keep it out of Git. Check the context before you change anything.
- Daily commands: `get` (with `-o wide`, `-n`, `-A`, `-l`), `describe` (read the events), `logs`, `exec`, `delete`, `scale`, `rollout status`.
- Delete a pod and a new one appears: the ReplicaSet controller reconciles.
- Reach pods with a Service; from your Mac use `port-forward`, from inside the cluster use a temporary pod (`kubectl run --rm -i`).
- Work declaratively: generate a manifest with `--dry-run=client -o yaml`, keep it in Git, check with `kubectl diff`, change with `kubectl apply` (idempotent).

## Sources and further reading

- [kind — Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/) — installing kind and creating clusters, including multi-node clusters.
- [kind — Releases](https://github.com/kubernetes-sigs/kind/releases) — downloads with checksums.
- [Kubernetes documentation — Install and Set Up kubectl on macOS](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/) — the official ways to install kubectl, with checksum checks.
- [Kubernetes documentation — kubectl Quick Reference](https://kubernetes.io/docs/reference/kubectl/quick-reference/) — the commands of this lesson and many more.
- [Kubernetes documentation — Organizing Cluster Access Using kubeconfig Files](https://kubernetes.io/docs/concepts/configuration/organize-cluster-access-kubeconfig/) — clusters, users and contexts.
- [Kubernetes documentation — Version Skew Policy](https://kubernetes.io/releases/version-skew-policy/) — kubectl should stay within one minor version of the cluster.

---

[🏠 Start](../START-HERE.md) · [← 21.2 Kubernetes concepts: pods, Deployments and Services](02-kubernetes-concepts.md) · [📋 Phase 21](README.md) · [21.4 Deploying a Rust service to Kubernetes →](04-deploying-a-rust-service-to-kubernetes.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
