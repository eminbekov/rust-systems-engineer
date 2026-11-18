<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 23.5 Capstone option 3: a message broker](05-capstone-message-broker.md) · [📋 Phase 23](README.md) · [23.7 Capstone option 5: your own idea →](07-capstone-your-own-idea.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/23-capstone-projects/06-capstone-observability-pipeline.md) <!-- nav -->

# Lesson 23.6 — Capstone option 4: an observability pipeline

> **Phase 23 — Capstone projects** · Lesson 6 of 10 · ⏱ about 50 hours (25 sittings)

## What you will learn

- What an observability pipeline (an agent such as Vector or the OpenTelemetry Collector) does, and why it is a hard and valuable systems project.
- A reference architecture: sources, transforms and sinks connected by bounded buffers, with checkpoints and acknowledgements.
- The contract: the command line, the configuration graph, the event model, and the rules for delivery, backpressure and file tailing.
- Seven milestones from a configuration checker to robust file tailing, reliable HTTP delivery, buffers, metrics and a Kubernetes DaemonSet.
- The quality bar for this option, stretch goals, and the real systems to study.

## Before you start

- You finished [Lesson 23.1](01-how-capstone-projects-work.md) and [Lesson 23.2](02-writing-a-design-document.md).
- You used observability tools from the other side: [Lesson 09.10 — Logging and tracing](../09-testing-debugging-benchmarking/10-logging-and-tracing.md), [Lesson 16.12 — Logs, metrics and traces with OpenTelemetry](../16-backend-services/12-logs-metrics-and-traces.md), [Lesson 21.9 — Metrics with Prometheus and Grafana](../21-cloud-native-infrastructure/09-metrics-with-prometheus-and-grafana.md) and [Lesson 21.10 — Logs and traces in a cluster](../21-cloud-native-infrastructure/10-logs-and-traces-in-a-cluster.md). Now you build the part in the middle.
- You will use again: file descriptors and inodes from [Lesson 13.4](../13-operating-systems/04-files-and-file-descriptors.md), Tokio channels from [Lesson 14.13](../14-concurrency-and-async/13-tokio-tasks-and-channels.md), backpressure from [Lesson 20.8](../20-reliability-and-hardening/08-overload-limits-and-backpressure.md), retries from [Lesson 18.3](../18-distributed-systems/03-retries-backoff-and-idempotency.md), topological sort from [Lesson 07.14](../07-algorithms-and-data-structures/14-shortest-paths-and-topological-sort.md) and snapshot tests from [Lesson 09.5](../09-testing-debugging-benchmarking/05-snapshot-testing.md).

## New words

| Word | Meaning |
|---|---|
| **observability pipeline** | A program that collects logs, metrics and traces from many places, changes them on the way, and sends them to storage systems. Often it runs as an **agent** on every machine. |
| **event** | One unit of data in the pipeline: a log line with its fields, or one metric value. |
| **source** | A component that produces events: a log file, standard input, an HTTP endpoint, host metrics. |
| **transform** | A component that changes events: parse, add fields, filter, redact, sample, turn logs into metrics. |
| **sink** | A component that sends events away: standard output, a file, an HTTP backend. |
| **topology** | The graph of sources, transforms and sinks, connected by their `inputs`. It must have no cycles. |
| **logfmt** | A simple log format of `key=value` pairs on one line, for example `level=info msg="user logged in"`. |
| **tailing** | Reading a file that keeps growing, line by line, as new lines are written — like `tail -f`. |
| **log rotation** | Replacing a log file that became large: the old file is renamed (or copied and truncated) and a new, empty one is started. |
| **inode** | The number by which a Unix file system identifies a file, independent of its name. A renamed file keeps its inode. |
| **checkpoint** | A saved file position ("read up to byte 81,920 of inode 1234"), so that the agent continues there after a restart. |
| **end-to-end acknowledgement** | A sink tells the source that a batch is safely delivered; only then may the source advance its checkpoint. |
| **buffer** (pipeline) | A bounded queue between components, in memory or on disk, with a rule for when it is full: block or drop. |
| **cardinality** | The number of different label combinations of a metric. High cardinality (a label per user ID, the identifier of a user) makes metric storage slow and expensive. |
| **OTLP** (OpenTelemetry Protocol) | The protocol in which OpenTelemetry tools send logs, metrics and traces, over gRPC (a remote procedure call protocol) or HTTP. |
| **DaemonSet** | A Kubernetes object that runs one copy of a pod on every node — the usual way to deploy a node agent. |
| **HTTP** (HyperText Transfer Protocol) | The request-and-response protocol of the web; HTTPS is HTTP over an encrypted connection. |
| **CPU** (Central Processing Unit) | The processor; "CPU use" is how much of its time a program takes. |
| **TOML** (Tom's Obvious Minimal Language) | The simple configuration format that `Cargo.toml` uses. |
| **JSON** (JavaScript Object Notation) | A text format for structured data, such as `{"error":"no route"}`. |
| **CI** (Continuous Integration) | A server that runs your checks automatically on every push (Lesson 09.12). |
| **VM** (Virtual Machine) | A whole computer simulated by software; the course's Linux VM is called `dev` (Lesson 11.1). |
| **TCP** (Transmission Control Protocol) | The internet protocol that gives two programs a reliable, ordered stream of bytes. |

## 1. What you build and why it matters

You build `pipe`, a small agent in the spirit of Vector, Fluent Bit or the OpenTelemetry Collector. It reads log lines from files that are being written and rotated, parses them into structured events, enriches and filters them, turns some of them into metrics, and delivers them in batches to backends over HTTP. It runs on every machine of a cluster, so it must be **cheap** (little CPU and memory), **safe** (never take the machine down with it) and **honest** (it never loses data silently: every dropped event is counted).

This looks like plumbing, but it is a dense systems problem:

- **Files are tricky.** Files are renamed, truncated and replaced while you read them; the agent restarts; lines are half-written. Reading every line exactly once (or at least once after a crash) needs inodes, checkpoints and care.
- **Backends fail.** The storage system is down for five minutes. Do you block, buffer in memory, buffer on disk, or drop? Each answer has a cost, and the agent must make the cost visible.
- **Throughput matters.** A busy machine produces tens of thousands of lines per second; the agent must keep up without using a whole CPU core.

Why it matters: almost every organisation that runs services runs such pipelines, and many outages are found (or missed) because of them. If you chose the eBPF or cloud-native track in Phase 22, this capstone extends it directly: an eBPF source or a Kubernetes metadata transform is a natural stretch goal.

## 2. Reference architecture

```text
                        pipe.toml --> config check (graph, cycles) --> topology
 sources                     transforms                        sinks
+------------------+       +---------------------+          +----------------------+
| file: app/*.log  |--+    | parse (logfmt/json) |          | http: batch, retry,  |
|  inode, offset,  |  +--->|  -> add_fields      |--------->|  backoff, ack        |
|  rotation        |  |    |  -> drop / redact   |    +---->| (to a log backend)   |
+--------+---------+  |    +----------+----------+    |     +----------+-----------+
         ^            |               |               |                |
         |   +--------+---------+     v               |     +----------+-----------+
         |   | stdin            |  +--------------+   |     | prometheus endpoint  |
         |   +------------------+  | log_to_metric|---+---->| /metrics (pull)      |
         |                         +--------------+         +----------------------+
         |       every arrow is a bounded buffer: memory or disk, block or drop
         |
   checkpoints  <------------ acknowledgements from sinks (batch delivered)

 self-observability: /metrics with events in/out, buffer fill, dropped, retries
```

The parts and their jobs:

1. **Sources** read data and turn it into events. The file source remembers, per file, the inode and the byte offset it has read up to.
2. **Transforms** are small, pure steps. Most of them take one event and emit zero, one or many events — that is easy to test without any I/O (input/output).
3. **Sinks** collect events into batches and deliver them, with retries. When a batch is delivered, the sink **acknowledges** it.
4. **Buffers** sit on every edge of the graph. They are bounded: when one is full, it either *blocks* (the producer waits, and the pressure moves back to the source — backpressure) or *drops* (and counts).
5. **Checkpoints** move forward only for data that every sink has acknowledged, so a crash can cause duplicates but never loss.

> 💡 **Tip:** Draw your topology as a graph in the design document and give every edge two numbers: its capacity and its behaviour when full. Most of the hard questions of this capstone are questions about those numbers.

## The project

### What you will build

A Cargo workspace `code/p23-pipe`:

| Crate | Kind | Job |
|---|---|---|
| `pipe-core` | library | the event model, the `Transform` trait and all transforms — pure, no I/O |
| `pipe-config` | library | the configuration, graph validation (unknown inputs, cycles) |
| `pipe-io` | library | sources, sinks, buffers, checkpoints |
| `pipe` | binary | wiring, signals, self-metrics, `pipe check` |
| `pipe-tests` | tests only | end-to-end tests with a test HTTP backend that can fail on command |

**Non-goals:** no traces (logs and metrics only), no query language for transforms (a fixed list of transform types), no cluster-wide aggregation, no user interface.

### Contract

**Command line:**

```text
pipe --config <FILE> [--data-dir <DIR>]     run; checkpoints and disk buffers live in --data-dir
pipe check --config <FILE>                  validate the file and the graph; print "config ok: …" or the error
pipe --version
```

**Configuration** (TOML): every component has a unique name, a `type` and — for transforms and sinks — `inputs`:

```toml
[sources.app]
type = "file"
include = ["/var/log/app/*.log"]

[transforms.parse]
type = "parse_logfmt"
inputs = ["app"]

[transforms.prod]
type = "add_fields"
inputs = ["parse"]
fields = { env = "prod" }

[transforms.quiet]
type = "drop_levels"
inputs = ["prod"]
levels = ["debug"]

[sinks.backend]
type = "http"
inputs = ["quiet"]
url = "http://127.0.0.1:9000/ingest"
batch = { max_events = 500, timeout_ms = 1000 }
buffer = { type = "disk", max_bytes = 268435456, when_full = "block" }
```

**Event model:** a log event has a timestamp, a message and fields (string keys, string values); a metric event has a name, a kind (`counter` or `gauge`), a value and tags. Your design document fixes the exact types and the JSON form that sinks send.

**Rules:**

1. **Graph:** a configuration is rejected if an input names an unknown component, if a source has inputs, or if the graph has a cycle; the error names the cycle (for example `cycle: a -> b -> a`). Components that no sink reaches are reported as a warning.
2. **Order:** events from one file reach each sink in the order of the lines in the file.
3. **Delivery:** at-least-once from file sources to acknowledging sinks. A file checkpoint moves past a line only when every sink that receives events from that line has acknowledged them. After a crash, reading continues from the checkpoint; some events may be delivered twice, none are lost.
4. **Buffers:** every edge has a bounded buffer. `when_full = "block"` makes the upstream component wait; `"drop_newest"` drops the incoming event and increases `pipe_events_dropped_total{component="…"}`. Memory use stays below the configured limits whatever the backend does.
5. **File source:** follows new files that match `include`; follows rotation by rename (keeps reading the old inode to its end, then switches) and by truncation (starts again at offset 0); identifies files by device and inode, not by name; lines longer than `max_line_bytes` (default 102,400) are cut and counted.
6. **Sinks:** send a batch when it reaches `max_events` or `timeout_ms`; retry timeouts, connection errors and `5xx` with exponential backoff and jitter; do not retry `4xx` (except `429`): drop the batch, count it, and log one error line.
7. **Self-observability:** `GET /metrics` on `127.0.0.1:9598` with at least events in and out per component, buffer fill, dropped events, sink retries and sink request latency; `GET /health`.
8. **Shutdown:** on `SIGTERM`, stop the sources, flush the buffers to the sinks for at most `drain_timeout_ms`, save the checkpoints, exit with code 0.
9. **Overhead:** the README states CPU and memory use at 10,000 events per second on stated hardware.

**The transform contract in code.** Transforms are the part with the most rules and the least I/O, so they are where you start. Try the model in a scratch package `code/p23-pipe-model`:

```rust
use std::collections::BTreeMap;

/// One log event moving through the pipeline.
#[derive(Clone, Debug, PartialEq)]
pub struct Event {
    pub message: String,
    pub fields: BTreeMap<String, String>,
}

impl Event {
    pub fn new(message: &str, fields: &[(&str, &str)]) -> Event {
        let fields = fields
            .iter()
            .map(|(k, v)| (k.to_string(), v.to_string()))
            .collect();
        Event {
            message: message.to_string(),
            fields,
        }
    }
}

/// One step of the pipeline: takes one event, emits zero, one or many.
pub trait Transform {
    fn apply(&mut self, event: Event, out: &mut Vec<Event>);
}

/// Adds the same field to every event.
pub struct AddField {
    pub key: String,
    pub value: String,
}

impl Transform for AddField {
    fn apply(&mut self, mut event: Event, out: &mut Vec<Event>) {
        event.fields.insert(self.key.clone(), self.value.clone());
        out.push(event);
    }
}

/// Drops events whose `level` field is in the list.
pub struct DropLevels {
    pub levels: Vec<String>,
}

impl Transform for DropLevels {
    fn apply(&mut self, event: Event, out: &mut Vec<Event>) {
        let dropped = event
            .fields
            .get("level")
            .is_some_and(|level| self.levels.contains(level));
        if !dropped {
            out.push(event);
        }
    }
}

/// Runs a batch of events through a chain of transforms, in order.
pub fn run_chain(chain: &mut [Box<dyn Transform>], batch: Vec<Event>) -> Vec<Event> {
    let mut current = batch;
    for transform in chain.iter_mut() {
        let mut next = Vec::with_capacity(current.len());
        for event in current {
            transform.apply(event, &mut next);
        }
        current = next;
    }
    current
}

fn main() {
    let mut chain: Vec<Box<dyn Transform>> = vec![
        Box::new(AddField {
            key: "host".into(),
            value: "web-1".into(),
        }),
        Box::new(DropLevels {
            levels: vec!["debug".into()],
        }),
    ];
    let input = vec![
        Event::new("user logged in", &[("level", "info")]),
        Event::new("cache miss", &[("level", "debug")]),
        Event::new("disk almost full", &[("level", "error")]),
    ];
    for event in run_chain(&mut chain, input) {
        println!("{} {:?}", event.message, event.fields);
    }
}
```

**What happens:**

1. `Event` keeps its fields in a `BTreeMap`, which is sorted by key. Output is then always in the same order, which makes tests and snapshot files stable.
2. `Transform::apply` takes the event **by value** and writes results into `out`. Moving the event avoids copies; writing into a `Vec` lets one transform drop an event (write nothing) or split it into several.
3. `&mut self` allows transforms with state, for example a sampler that keeps every tenth event, or a transform that counts events to produce a metric.
4. `DropLevels` uses `Option::is_some_and`: an event with no `level` field is kept.
5. `run_chain` passes the whole batch through one transform after the other. `Box<dyn Transform>` (a trait object, Lesson 06.1) lets the configuration choose the transforms at run time.

```bash
cargo run
```

```text
user logged in {"host": "web-1", "level": "info"}
disk almost full {"host": "web-1", "level": "error"}
```

In the capstone the chain is built from the configuration, the batch comes from a buffer, and the result goes into the next buffer — but the transforms themselves stay exactly this testable.

### Step-by-step plan

The outputs below are **part of the contract**: what your finished agent must print. They are not output produced by the course. Write each check as an acceptance test before the milestone.

#### M0 — Walking skeleton and `pipe check`

**Goal.** The workspace with CI; the event model and the transforms above with their tests; `pipe check` that reads the configuration and validates the graph (rule 1).

<details>
<summary>Hint</summary>

A cycle check is a topological sort (Lesson 07.14): repeatedly remove components that have no remaining inputs; if components are left at the end, they are on a cycle. To *name* the cycle, run a depth-first search from one of the remaining components and print the path when you come back to a component that is already on it.

</details>

**Done when:** `pipe check --config pipe.toml` prints `config ok: 1 source(s), 3 transform(s), 1 sink(s)`; a configuration in which `parse` has `inputs = ["quiet"]` prints `invalid config: cycle: parse -> prod -> quiet -> parse` and exits with 1.

#### M1 — A first end-to-end pipeline

**Goal.** Sources `stdin` and `file` (read once from the start), transforms `parse_logfmt`, `parse_json`, `add_fields`, `drop_levels`, `redact`, and sinks `stdout` and `file` (one JSON object per line).

<details>
<summary>Hint</summary>

Keep a set of example input files in `tests/data/` and check the output with snapshot tests (`insta`, Lesson 09.5). One snapshot per transform type and one for the whole example configuration show every change of behaviour in review.

</details>

**Done when:**

```bash
printf 'level=info msg="user logged in" user=ana\nlevel=debug msg="cache miss"\n' | pipe --config stdin-to-stdout.toml
```

```text
{"message":"user logged in","fields":{"env":"prod","level":"info","user":"ana"}}
```

(`stdin-to-stdout.toml` is the example configuration above with the `file` source replaced by a `stdin` source and the `http` sink by a `stdout` sink.) Also: a line that is not valid logfmt is passed on with the field `parse_error` instead of being dropped; the snapshot tests cover every transform.

#### M2 — Tailing files robustly

**Goal.** The file source of rule 5: follow, discover new files, handle rotation by rename and by truncation, save checkpoints in `--data-dir`, and continue from them after a restart.

<details>
<summary>Hint</summary>

Test rotation with real files in a temporary directory (`tempfile`): write numbered lines (`line 1`, `line 2`, …), rename the file, create a new one, keep writing, and at the end check that the sink saw every number exactly once. Polling the directory every 250 ms is simpler and more portable than file-system notifications; start with it.

</details>

**Done when:** a test writes 100,000 numbered lines while rotating the file by rename every 10,000 lines and once by truncation, and the `file` sink receives every number exactly once and in order; a test stops `pipe` cleanly (`SIGTERM`) in the middle, starts it again, and the combined output still has every number exactly once.

#### M3 — Reliable HTTP delivery

**Goal.** The `http` sink of rule 6 with batching, retries, backoff with jitter, and end-to-end acknowledgements so that checkpoints follow rule 3.

<details>
<summary>Hint</summary>

Give each batch a list of the (file, offset) positions it contains. When the sink gets a `2xx`, it sends this list back to the source, which advances the checkpoint to the highest position below which *everything* is acknowledged. The test backend (in `pipe-tests`) should be able to fail a given percentage of requests and to be slow on command.

</details>

**Done when:** with a test backend that fails 30% of requests with `503`, all 100,000 events arrive at least once and the duplicates are counted; a test kills `pipe` with `SIGKILL` (`kill -9`) at a random moment, restarts it, and the backend has received every line at least once; a `400` answer drops exactly that batch and increases `pipe_sink_batches_dropped_total`.

#### M4 — Buffers and backpressure

**Goal.** Memory and disk buffers with `block` and `drop_newest` (rule 4), and proof that memory stays bounded.

<details>
<summary>Hint</summary>

A disk buffer is a small write-ahead log (Lesson 17.4): append batches with a length and a checksum, keep a read position, delete fully acknowledged segment files. Measure memory with the resident set size of the process (`ps -o rss= -p <pid>` in a test helper) while the backend is stopped.

</details>

**Done when:** with the backend stopped for 60 seconds at 5,000 events per second: with a memory buffer and `block`, the process's memory stays below the limit you documented, the source falls behind and then catches up after the backend returns, and no event is lost; with `drop_newest`, the dropped counter equals exactly the number of missing events; with a 256 MiB disk buffer, all events of the outage are delivered after the backend returns, also after a restart of `pipe` during the outage.

#### M5 — Metrics

**Goal.** A `log_to_metric` transform (for example: count events by `level`, or a histogram of a numeric field), a `prometheus` sink that serves the metrics for scraping, and an `otlp` sink that sends logs to an OpenTelemetry Collector container as in Lesson 16.12.

<details>
<summary>Hint</summary>

Decide in your design document which fields may become metric labels, and refuse the rest: a label like `user` or `request_id` creates one time series per value (high cardinality). Count refused labels in a self-metric so the user sees why a metric is missing.

</details>

**Done when:** with the `prometheus` sink configured to listen on `127.0.0.1:9599` (the agent's own metrics of rule 7 stay on port 9598), `curl -s 127.0.0.1:9599/metrics` shows `app_events_total{level="error"}` growing while error lines are written; the collector container (named `course-otelcol`) logs your events with its `debug` exporter; a `log_to_metric` configuration that uses a field marked as forbidden for labels is rejected by `pipe check`.

#### M6 — Performance and deployment

**Goal.** Measured throughput and overhead, and a deployment as a Kubernetes DaemonSet in a local `kind` cluster (Lessons 21.3 and 21.4).

<details>
<summary>Hint</summary>

Generate load with a small program that writes logfmt lines at a fixed rate to a file. Measure CPU with `ps -o %cpu` or `/proc/<pid>/stat` in your Linux VM `dev`, and profile at the highest rate you test (Lesson 19.3). In the DaemonSet, mount `/var/log/pods` read-only from the node and give the container only the permissions it needs.

</details>

**Done when:** `docs/results.md` reports events per second per CPU core and memory use at 1,000, 10,000 and 50,000 events per second with the example configuration, with a flame graph and an explanation of the largest cost; in a `kind` cluster named `course-pipe`, the DaemonSet collects the logs of a test pod and delivers them to the collector; the pod runs as a non-root user with a read-only root file system.

### Quality bar

| Area | What it means here |
|---|---|
| Unit and snapshot tests | every transform; snapshot tests for parsing and the JSON output |
| Property tests | `parse_logfmt` never panics and round-trips events it produced itself; the graph check accepts every acyclic graph and rejects every cyclic one |
| Failure tests | rotation, truncation, crash and restart (M2, M3), backend down, slow and returning `4xx`/`5xx` (M3, M4) |
| Fuzzing | the logfmt and JSON parsers, the configuration loader, the disk buffer reader |
| Benchmarks | events per second per core for the example pipeline; `criterion` for each parser |
| Observability | the self-metrics of rule 7; one log line per state change, never one per event |
| Robustness | bounded buffers everywhere, maximum line length, timeouts on every HTTP request |

> 🔒 **Security:** Logs often contain secrets that should never have been logged: passwords, tokens, card numbers. A `redact` transform close to the source is a real feature, not a toy. Also treat log lines as hostile input: a parser that allocates according to a number in the line, or a regular expression that can take exponential time, lets one bad line stop the agent.

### Stretch goals

- **An eBPF source** that counts TCP retransmits or process starts (Lessons 22.6–22.9) and emits them as metric events.
- **Kubernetes metadata:** add pod name, namespace and labels to events from `/var/log/pods`, with a cache fed by the Kubernetes API (Application Programming Interface; Lesson 22.27).
- **Prometheus remote write** sink (protobuf — Protocol Buffers, a compact binary format — with Snappy compression) to VictoriaMetrics or Prometheus.
- **Hot reload** of the configuration without losing buffered events.
- **`pipe tap <component>`**: watch the events that pass one component, live, for debugging.

## Real systems to study

| System | What to look at |
|---|---|
| [Vector](https://vector.dev/docs/) | a pipeline written in Rust; read its [buffering model](https://vector.dev/docs/architecture/buffering-model/) and [end-to-end acknowledgements](https://vector.dev/docs/architecture/end-to-end-acknowledgements/), then its file source in the [repository](https://github.com/vectordotdev/vector) |
| [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) | receivers, processors and exporters — the same graph with different names |
| [Fluent Bit](https://docs.fluentbit.io/manual) | a very small agent; its documentation on tailing, buffering and backpressure |
| [Prometheus Remote-Write spec](https://prometheus.io/docs/specs/prw/remote_write_spec/) | the wire format most metric backends accept |

## Common mistakes

**1. Advancing the checkpoint when a line is read.** The agent reads a line, saves the position, crashes before the sink delivered it — the line is lost. Rule 3: the checkpoint follows acknowledgements, not reads.

**2. Identifying files by name.** After rotation, `app.log` is a new file; the agent continues at the old offset in the new file and skips its beginning (or reads garbage). Fix: device and inode, plus a check for truncation (the file is shorter than your offset).

**3. Unbounded channels.** `tokio::sync::mpsc::unbounded_channel` between a fast source and a slow sink: during an outage, memory grows until the machine kills the agent — and maybe other programs. Fix: rule 4, everywhere.

**4. One log line per event about the agent itself.** "dropped event" logged ten thousand times per second makes the agent its own biggest log source. Fix: counters in metrics, and rate-limited log lines.

**5. Retrying `400 Bad Request` forever.** The batch can never succeed, and it blocks everything behind it. Fix: rule 6 — drop, count, log once.

**6. High-cardinality labels.** Turning `user_id` into a metric label creates millions of time series in the backend. Fix: an allow-list of label keys (M5).

## Exercises

Do these in `code/p23-pipe-model` before M0.

### Exercise 1 — When may the checkpoint move? (easy)

Lines 1–100 of a file go into a batch that the `http` sink sends; lines 101–200 go into a second batch. The second batch is acknowledged first, then the agent crashes before the first one is acknowledged. Up to which line may the checkpoint have moved, and what happens after the restart?

<details>
<summary>Hint</summary>

Rule 3 says "past a line only when every sink … has acknowledged it". Is everything before line 200 acknowledged?

</details>

<details>
<summary>Answer</summary>

The checkpoint must still be before line 1 (at the position where line 1 starts), because line 1 is not acknowledged, even though lines 101–200 are. After the restart, the agent reads lines 1–200 again; lines 101–200 are delivered a second time. That is the price of at-least-once delivery, and it is why the checkpoint is "the highest position below which everything is acknowledged".

</details>

### Exercise 2 — Write tests for `DropLevels` (medium)

Add a test module with two tests: a `debug` event is dropped and a `warn` event is kept; an event without a `level` field is kept.

<details>
<summary>Hint</summary>

Call `apply` directly with an empty `Vec` as `out`, then look at the `Vec`. A small helper function that builds the transform keeps the tests short.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn drop_debug() -> DropLevels {
        DropLevels {
            levels: vec!["debug".into()],
        }
    }

    #[test]
    fn debug_events_are_dropped_and_others_kept() {
        let mut out = Vec::new();
        drop_debug().apply(Event::new("a", &[("level", "debug")]), &mut out);
        drop_debug().apply(Event::new("b", &[("level", "warn")]), &mut out);
        assert_eq!(out, vec![Event::new("b", &[("level", "warn")])]);
    }

    #[test]
    fn events_without_a_level_are_kept() {
        let mut out = Vec::new();
        drop_debug().apply(Event::new("no level here", &[]), &mut out);
        assert_eq!(out.len(), 1);
    }
}
```

```text
running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: transforms have no I/O, so a test is just "call, then compare". The second test pins a decision (keep events without a level) that a later change could easily break.

</details>

### Exercise 3 — A `Redact` transform, test first (medium)

Write a transform `Redact { keys: Vec<String> }` that replaces the **value** of every listed field with `***` and keeps everything else. Start with the test `secrets_are_redacted`: an event with `user = ana` and `password = hunter2` must come out with `user = ana` and `password = ***`.

<details>
<summary>Hint</summary>

Before `Redact` exists, the test does not compile — that is the "red" step for a new type. `BTreeMap::get_mut` returns `Option<&mut String>`; assign through it with `*value = …`.

</details>

<details>
<summary>Answer</summary>

The test, inside `mod tests`:

```rust
#[test]
fn secrets_are_redacted() {
    let mut redact = Redact {
        keys: vec!["password".into(), "token".into()],
    };
    let mut out = Vec::new();
    let event = Event::new("login", &[("user", "ana"), ("password", "hunter2")]);
    redact.apply(event, &mut out);
    assert_eq!(
        out,
        vec![Event::new("login", &[("user", "ana"), ("password", "***")])]
    );
}
```

Before the transform exists, `cargo test` stops with:

```text
error[E0422]: cannot find struct, variant or union type `Redact` in this scope
   --> src/main.rs:118:26
    |
118 |         let mut redact = Redact {
    |                          ^^^^^^ not found in this scope
```

The transform (put it before `fn main`):

```rust
/// Replaces the values of secret fields with `***`.
pub struct Redact {
    pub keys: Vec<String>,
}

impl Transform for Redact {
    fn apply(&mut self, mut event: Event, out: &mut Vec<Event>) {
        for key in &self.keys {
            if let Some(value) = event.fields.get_mut(key) {
                *value = "***".to_string();
            }
        }
        out.push(event);
    }
}
```

Why it works: only fields that exist are changed (`token` is not in the event, so nothing happens for it), and the field stays present, so a reader still sees that a password was there. The message text itself is not redacted — a real `redact` transform also needs patterns for the message, which your design document should discuss.

</details>

### Exercise 4 — Size the buffer (hard)

Your agent receives 2,000 events per second, about 300 bytes each. The backend may be down for up to 5 minutes. (a) How large must a buffer be to hold the whole outage? (b) Is a memory buffer a good idea on a machine where the agent should use less than 100 MiB? (c) Write a function `buffer_bytes(events_per_second, bytes_per_event, outage_seconds) -> u64` with a test for this case.

<details>
<summary>Hint</summary>

Multiply rate, size and time. 1 MiB is 1,048,576 bytes.

</details>

<details>
<summary>Answer</summary>

(a) 2,000 × 300 × 300 s = 180,000,000 bytes, about 172 MiB. (b) No: that is more than the agent's whole memory budget. Use a disk buffer for the outage (the events are written sequentially, which disks do well) and a small memory buffer in front of it; or decide to drop after a limit and say so. Either way the decision and its numbers go into the design document.

```rust
/// Bytes a buffer must hold to survive an outage of the sink without dropping events.
pub fn buffer_bytes(events_per_second: u64, bytes_per_event: u64, outage_seconds: u64) -> u64 {
    events_per_second * bytes_per_event * outage_seconds
}

#[test]
fn five_minutes_at_two_thousand_events_per_second() {
    assert_eq!(buffer_bytes(2_000, 300, 5 * 60), 180_000_000);
}
```

Why it works: it is the same back-of-the-envelope arithmetic as in Lesson 23.2, kept as a tested function so that you can rerun it with your own numbers (and add it to `code/p23-estimate`).

</details>

## Quiz

1. What are sources, transforms and sinks, and how are they connected?

<details>
<summary>Answer</summary>

Sources produce events, transforms change them, sinks send them away. They form a graph through the `inputs` of transforms and sinks, and every edge of the graph is a bounded buffer.

</details>

2. Why does the file source identify files by device and inode, not by name?

<details>
<summary>Answer</summary>

Because rotation renames files and creates new files with the old name. The inode stays with the file's content, so the agent can finish the old file and start the new one at offset 0.

</details>

3. What does "at-least-once from file to sink" require of checkpoints?

<details>
<summary>Answer</summary>

A checkpoint may move past a line only after every sink that received events from that line has acknowledged them. Then a crash can cause duplicates, never loss.

</details>

4. What is the difference between `when_full = "block"` and `"drop_newest"`, and what must the agent do in the second case?

<details>
<summary>Answer</summary>

With `block`, the upstream component waits, so the pressure moves back to the source (which stops reading). With `drop_newest`, incoming events are discarded; the agent must count every dropped event in a metric, so the loss is never silent.

</details>

5. Why is the `Transform` trait designed to take the event by value and write into `out: &mut Vec<Event>`?

<details>
<summary>Answer</summary>

Taking the event by value avoids copies, and writing into `out` lets one transform emit zero events (filter), one (change) or many (split) with the same signature.

</details>

6. Why is a high-cardinality label a problem, and how does the contract prevent it?

<details>
<summary>Answer</summary>

Every distinct label value creates a new time series in the metrics backend, which makes it slow and expensive. M5 uses an allow-list (or a deny-list) of label keys, checked by `pipe check`.

</details>

7. Why should an agent's own problems be reported as metrics rather than as one log line per event?

<details>
<summary>Answer</summary>

One log line per dropped event can produce thousands of lines per second and make the agent its own largest log source. Counters show the same information cheaply; log lines about state changes are enough.

</details>

## Summary

- Option 4 is an observability agent: sources, transforms and sinks in a validated graph, connected by bounded buffers.
- The file source follows rotation and truncation by inode and saves checkpoints; checkpoints move only with end-to-end acknowledgements, which gives at-least-once delivery.
- Buffers block or drop, in memory or on disk; every drop is counted, and memory stays bounded whatever the backend does.
- The contract fixes the command line, the configuration graph, the event model and nine rules; the `Transform` trait and its tests are the first code.
- Milestones: M0 `pipe check`, M1 first pipeline, M2 robust tailing, M3 reliable HTTP delivery, M4 buffers, M5 metrics, M6 performance and a DaemonSet.
- Study Vector, the OpenTelemetry Collector, Fluent Bit and the remote-write specification after designing each part yourself.

## Sources and further reading

- [Vector — buffering model](https://vector.dev/docs/architecture/buffering-model/) — memory and disk buffers, and the choice between blocking and dropping.
- [Vector — end-to-end acknowledgements](https://vector.dev/docs/architecture/end-to-end-acknowledgements/) — how a source learns that its data was delivered.
- [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) — the architecture of receivers, processors and exporters.
- [OTLP specification](https://opentelemetry.io/docs/specs/otlp/) — the protocol of the `otlp` sink in M5.
- [logfmt](https://brandur.org/logfmt) — a short description of the `key=value` log format parsed in M1.
- [insta](https://insta.rs/) — snapshot testing for the transform outputs.

---

[🏠 Start](../START-HERE.md) · [← 23.5 Capstone option 3: a message broker](05-capstone-message-broker.md) · [📋 Phase 23](README.md) · [23.7 Capstone option 5: your own idea →](07-capstone-your-own-idea.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
