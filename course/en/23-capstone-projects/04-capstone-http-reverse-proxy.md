<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 23.3 Capstone option 1: a distributed key-value store](03-capstone-distributed-key-value-store.md) · [📋 Phase 23](README.md) · [23.5 Capstone option 3: a message broker →](05-capstone-message-broker.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/23-capstone-projects/04-capstone-http-reverse-proxy.md) <!-- nav -->

# Lesson 23.4 — Capstone option 2: a high-performance HTTP reverse proxy

> **Phase 23 — Capstone projects** · Lesson 4 of 10 · ⏱ about 50 hours (25 sittings)

## What you will learn

- What a production HTTP reverse proxy does between clients and backends, and why it is a strong portfolio project.
- A reference architecture: listeners, a router, upstream pools with health and balancing, and a configuration that can change while traffic flows.
- The contract: the command line, the configuration file, the forwarding rules, the error codes and the admin API.
- Seven milestones from a configuration checker to TLS, hot reload, rate limits and a measured, profiled proxy.
- The quality bar for this option, stretch goals, and the real proxies to study.

## Before you start

- You finished [Lesson 23.1](01-how-capstone-projects-work.md) and [Lesson 23.2](02-writing-a-design-document.md).
- This capstone grows out of [Lesson 15.14 — Proxies and load balancers](../15-networking/14-proxies-and-load-balancers.md), [Lesson 15.15 — Project: a TCP load balancer](../15-networking/15-project-tcp-load-balancer.md) (`p15-lb`) and, if you chose the networking track, [Lesson 22.21 — Networking track: project and next steps](../22-specializations/21-networking-track-project-and-next-steps.md) (`p22-relay`).
- You will use again: `hyper` from [Lesson 15.9](../15-networking/09-http-with-hyper-and-reqwest.md), TLS with `rustls` from [Lesson 15.12](../15-networking/12-tls-in-rust-servers.md), graceful shutdown from [Lesson 14.14](../14-concurrency-and-async/14-cancellation-timeouts-and-graceful-shutdown.md), limits and backpressure from [Lesson 20.8](../20-reliability-and-hardening/08-overload-limits-and-backpressure.md), load tests from [Lesson 19.2](../19-performance-and-profiling/02-load-testing.md) and flame graphs from [Lesson 19.3](../19-performance-and-profiling/03-profilers-and-flame-graphs.md).

## New words

| Word | Meaning |
|---|---|
| **upstream** / **backend** | A server behind the proxy that really answers the request. A group of equal backends is a **pool** (Envoy calls it a *cluster*). |
| **downstream** | The client side of the proxy: the browser, `curl` or another service that sends the request. |
| **route** | A rule that says which pool gets a request, based on the host name and the path. |
| **hop-by-hop header** | A header that is meant only for one connection, such as `Connection` or `Transfer-Encoding`. A proxy removes it and never forwards it. |
| **`X-Forwarded-For`** | A header in which proxies write the client's IP (Internet Protocol) address, so that the backend knows who really sent the request. |
| **active health check** | The proxy itself sends a request (for example `GET /healthz`) to each backend at intervals. |
| **passive health check** / **outlier detection** | The proxy watches real traffic and removes a backend for a while after several errors in a row. |
| **retry budget** | A limit on retries as a share of all requests (for example 10%), so that retries cannot multiply the load during an outage. |
| **idempotent method** | An HTTP method whose repetition has the same effect as one call: `GET`, `HEAD`, `OPTIONS`, `PUT`, `DELETE`. `POST` is not. |
| **SNI** (Server Name Indication) | The host name that a TLS (Transport Layer Security) client sends in its first message, so that one address can serve several certificates. |
| **ALPN** (Application-Layer Protocol Negotiation) | The TLS extension in which client and server agree on HTTP/1.1 or HTTP/2. |
| **hot reload** | Applying a new configuration while the proxy runs, without dropping connections or requests. |
| **`ArcSwap`** | A type from the `arc-swap` crate: an `Arc` that can be replaced atomically while many threads read it, without a lock. Ideal for "the current configuration". |
| **token bucket** | A rate limiter: a bucket refills with tokens at a fixed rate; each request takes one; with an empty bucket the request is refused (`429`). |
| **slowloris** | An attack that opens many connections and sends headers very slowly, to use up the server's connections. A header read timeout stops it. |
| **HTTP** (HyperText Transfer Protocol) | The request-and-response protocol of the web; HTTPS is HTTP over an encrypted connection. |
| **JSON** (JavaScript Object Notation) | A text format for structured data, such as `{"error":"no route"}`. |
| **CI** (Continuous Integration) | A server that runs your checks automatically on every push (Lesson 09.12). |
| **RFC** (Request for Comments) | Here: a numbered internet standard, such as RFC 9110 for HTTP semantics. |
| **VM** (Virtual Machine) | A whole computer simulated by software; the course's Linux VM is called `dev` (Lesson 11.1). |

## 1. What you build and why it matters

You build `proxy`, an HTTP reverse proxy in the spirit of Envoy or Pingora. It accepts HTTP/1.1 and HTTP/2 from clients, with or without TLS, picks a pool by host and path, picks a healthy backend with a balancing algorithm, forwards the request, and returns the answer. Around this core it protects the backends and itself: timeouts, safe retries, health checks, rate limits, connection limits, and a configuration that can be replaced without dropping a single request.

You already built the base. `p15-lb` copied bytes between TCP (Transmission Control Protocol) connections; `p22-relay` (in the networking track) forwarded HTTP/1.1 with round robin. The capstone adds what makes a proxy usable in front of real services:

- **HTTP/2 and TLS** with several certificates on one address (SNI) and protocol choice (ALPN);
- **routing** by host and path to several pools with different algorithms;
- **health**: active checks and outlier detection;
- **safe retries** with a budget, and clear error codes (`502`, `503`, `504`);
- **hot reload** of the configuration, rate limiting and an admin API;
- **numbers**: proxy overhead in p50/p99/p99.9 latency (the median, the 99th and the 99.9th percentile, Lesson 19.1), and a flame graph that explains it.

Why it matters: most requests to a web service pass through several proxies — load balancers, API (Application Programming Interface) gateways, service-mesh sidecars. They are the place where latency is added, where outages spread or stop, and where security rules are enforced. A proxy that you built, measured and explained shows that you understand networking, concurrency and performance at the same time.

## 2. Reference architecture

```text
                       proxy.toml --(SIGHUP / file change)--> config loader
                                                                  | validate
                                                                  v
 clients                                                  ArcSwap<Config>
 HTTP/1.1, HTTP/2                                                 | read per request
 TLS (SNI, ALPN)                                                  |
    |      +---------------+    +-------------+    +--------------v----------+
    +----->| listeners     |--->| limits:     |--->| router: host + prefix   |
           | accept, TLS,  |    | conns, rate,|    |   -> route -> pool      |
           | header timeout|    | body size   |    +--------------+----------+
           +---------------+    +-------------+                   |
                                                                  v
                     +-------------------------------------------------------+
                     | upstream pool "api"   algorithm: least_requests        |
                     |   backend A (up)   backend B (up)   backend C (ejected)|
                     |   connection pool per backend, timeouts, retries      |
                     +--------------------------+----------------------------+
                                                ^
                health checker: GET /healthz ---+   outlier detector: errors in a row
 admin API (127.0.0.1:9900): GET /pools, GET /metrics, POST /reload
```

The parts and their jobs:

1. **Listeners** accept connections, do the TLS handshake (choosing the certificate by SNI and the protocol by ALPN), and limit how long a client may take to send its headers.
2. **Limits** come before any real work: a maximum number of connections, a token bucket per client address, a maximum body size.
3. **The router** finds the first route whose host and path prefix match, and so the pool.
4. **The pool** chooses a healthy backend with its algorithm, reuses connections to it, applies the route's timeout, and retries only when the rules allow it.
5. **The health checker and the outlier detector** keep the "up" / "ejected" state of each backend.
6. **The configuration** lives behind an `ArcSwap`. Each request loads the current version once at the start and uses it until the end, so a reload never changes the rules in the middle of a request.

> ⚡ **Performance:** A proxy's job is to add as little as possible. Measure the *overhead*: run the same load test once directly against a backend and once through the proxy, and report the difference at p50, p99 and p99.9. Never measure the proxy without the baseline.

## The project

### What you will build

A Cargo workspace `code/p23-proxy`:

| Crate | Kind | Job |
|---|---|---|
| `proxy-config` | library | the configuration types, `load` and validation (the contract code below) |
| `proxy-core` | library | router, balancing algorithms, retry and health logic — pure, without I/O (input/output) |
| `proxy` | binary | listeners, TLS, forwarding with `hyper`, admin API, signals |
| `test-backend` | binary | a small backend for tests: answers with its own name, can be told to be slow or fail |

**Non-goals:** no response caching, no WebSockets, no HTTP/3, no dynamic service discovery (backends come from the configuration file), no `Lua`-style plug-in scripting.

### Contract

**Command line:**

```text
proxy --config <FILE>            run the proxy
proxy check --config <FILE>      validate the file, print "config ok: …", exit 0; or print the error, exit 1
proxy --version
```

**Configuration file** (TOML, Tom's Obvious Minimal Language — the format of `Cargo.toml`). The minimal form, which M0 already reads:

```toml
[[listener]]
address = "127.0.0.1:8080"

[[route]]
host = "api.example.test"
path_prefix = "/v1/"
pool = "api"
timeout_ms = 2000
retries = 1

[[route]]
path_prefix = "/"
pool = "web"

[[pool]]
name = "api"
algorithm = "least_requests"
backends = ["127.0.0.1:9001", "127.0.0.1:9002"]

[[pool]]
name = "web"
algorithm = "round_robin"
backends = ["127.0.0.1:9101"]
```

Later milestones add (each must be documented in your README's configuration reference): `tls = { cert = "…", key = "…" }` entries per listener and host name (M4), `[pool.health_check]` with `path`, `interval_ms`, `unhealthy_after`, `healthy_after` (M3), `rate_limit_per_second` per route (M5), and a `[limits]` table with `max_connections`, `header_timeout_ms`, `max_body_bytes`, `drain_timeout_ms`.

**Forwarding rules:**

1. **Routing:** routes are checked in file order; the first route matches when its `host` (if given) equals the request's host without the port, and its `path_prefix` is a prefix of the path. No match: `404`.
2. **Headers:** hop-by-hop headers (`Connection`, `Keep-Alive`, `Proxy-Connection`, `TE`, `Trailer`, `Transfer-Encoding`, `Upgrade`, and every header named in `Connection`) are removed in both directions. The proxy appends the client address to `X-Forwarded-For`, sets `X-Forwarded-Proto` to `http` or `https`, and adds `x-request-id` if the request has none.
3. **Errors** have a small JSON body `{"error":"…"}`: no healthy backend `503`; connection to the backend failed after all retries `502`; route timeout `504`; body too large `413`; rate limit `429` with `Retry-After`; too many connections — the connection is closed.
4. **Retries:** only for `GET`, `HEAD` and `OPTIONS`, only if no byte of the response has reached the client, at most `retries` times, always to a *different* backend, and only while the pool's retry budget (10% of its requests in the last 10 s) allows it. (`PUT` and `DELETE` are idempotent by the HTTP standard, but a proxy cannot know whether every backend really implements them that way, so it does not retry them either.)
5. **Health:** a backend is *down* after `unhealthy_after` failed active checks and *up* again after `healthy_after` good ones. Five failed requests in a row eject it for 30 s. Only *up* backends receive traffic.
6. **Reload:** on `SIGHUP`, a change of the file, or `POST /reload`, the new file is validated. If it is invalid, the old configuration stays and the error is logged and returned. If it is valid, new requests use it; requests in flight finish with the old one. Health state is kept for backends that exist in both.
7. **Shutdown:** on `SIGTERM` or `Ctrl+C`, stop accepting, finish requests in flight (at most `drain_timeout_ms`), then exit with code 0.

**Admin API** on a separate address (default `127.0.0.1:9900`): `GET /pools` → JSON with each backend's state, in-flight requests and recent errors; `GET /metrics` → Prometheus text; `POST /reload` → `200 {"reloaded":true}` or `400 {"error":"…"}`.

**The configuration contract in code.** M0 starts with the part that everything else depends on: reading and checking the file. Try it in a scratch package:

```bash
cd ~/projects/rust
cargo new code/p23-proxy-config
cd code/p23-proxy-config
cargo add serde@1 --features derive
cargo add toml@1
```

Save the file above as `proxy.toml` in the package folder. `src/main.rs`:

```rust
use serde::Deserialize;
use std::collections::HashSet;
use std::net::SocketAddr;

#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct Config {
    pub listener: Vec<Listener>,
    pub route: Vec<Route>,
    pub pool: Vec<Pool>,
}

#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct Listener {
    pub address: SocketAddr,
}

#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct Route {
    /// `None` matches every host.
    pub host: Option<String>,
    pub path_prefix: String,
    pub pool: String,
    #[serde(default = "default_timeout_ms")]
    pub timeout_ms: u64,
    #[serde(default)]
    pub retries: u32,
}

fn default_timeout_ms() -> u64 {
    5_000
}

#[derive(Debug, Deserialize)]
#[serde(rename_all = "snake_case")]
pub enum Algorithm {
    RoundRobin,
    LeastRequests,
    ConsistentHash,
}

#[derive(Debug, Deserialize)]
#[serde(deny_unknown_fields)]
pub struct Pool {
    pub name: String,
    pub algorithm: Algorithm,
    pub backends: Vec<SocketAddr>,
}

/// Parses and checks a configuration. Every error names what is wrong.
pub fn load(text: &str) -> Result<Config, String> {
    let config: Config = toml::from_str(text).map_err(|e| e.to_string())?;
    let pools: HashSet<&str> = config.pool.iter().map(|p| p.name.as_str()).collect();
    for route in &config.route {
        if !pools.contains(route.pool.as_str()) {
            return Err(format!(
                "route {:?} uses unknown pool {:?}",
                route.path_prefix, route.pool
            ));
        }
        if route.timeout_ms == 0 {
            return Err(format!(
                "route {:?}: timeout_ms must be > 0",
                route.path_prefix
            ));
        }
    }
    for pool in &config.pool {
        if pool.backends.is_empty() {
            return Err(format!("pool {:?} has no backends", pool.name));
        }
    }
    Ok(config)
}

fn main() {
    let path = std::env::args().nth(1).unwrap_or_else(|| "proxy.toml".to_string());
    let text = std::fs::read_to_string(&path).expect("cannot read the config file");
    match load(&text) {
        Ok(c) => println!(
            "config ok: {} listener(s), {} route(s), {} pool(s)",
            c.listener.len(),
            c.route.len(),
            c.pool.len()
        ),
        Err(e) => {
            eprintln!("invalid config {path}: {e}");
            std::process::exit(1);
        }
    }
}
```

**What happens:**

1. Each `[[listener]]`, `[[route]]` and `[[pool]]` table in the file becomes one element of a `Vec`. `#[derive(Deserialize)]` from `serde` writes the reading code for us.
2. `#[serde(deny_unknown_fields)]` turns a typo into an error instead of silently ignoring the line. For a proxy this is important: an ignored `timeout_ms` would mean the default is used without anyone noticing.
3. Addresses are read straight into `SocketAddr`, so `"127.0.0.1:99999"` is rejected while reading. *Parse, don't validate* (Lesson 08.5): after `load`, the types guarantee the format.
4. `#[serde(default = "default_timeout_ms")]` fills in 5 seconds when a route has no timeout; `#[serde(default)]` gives `retries` the value 0.
5. `rename_all = "snake_case"` maps `"least_requests"` in the file to `Algorithm::LeastRequests`.
6. `load` then checks what types alone cannot: every route names a pool that exists, timeouts are positive, pools have backends.

```bash
cargo run
```

```text
config ok: 1 listener(s), 2 route(s), 2 pool(s)
```

A typo in a key (`algoritm` instead of `algorithm`), saved as `typo.toml`:

```bash
cargo run -- typo.toml
```

```text
invalid config typo.toml: TOML parse error at line 22, column 1
   |
22 | algoritm = "round_robin"
   | ^^^^^^^^
unknown field `algoritm`, expected one of `name`, `algorithm`, `backends`
```

And a route that points to a pool that does not exist (`pool = "www"`):

```text
invalid config unknown.toml: route "/" uses unknown pool "www"
```

In the capstone this code moves into the `proxy-config` crate, `main` becomes `proxy check`, and the same `load` runs on every reload — so an invalid file can never replace a working configuration (rule 6).

### Step-by-step plan

The outputs below are **part of the contract**: what your finished proxy must print or return. They are not produced by the course. Write each check as an acceptance test first; the tests start `test-backend` processes on free ports and the proxy with a generated configuration.

#### M0 — Walking skeleton and `proxy check`

**Goal.** The workspace, CI, `proxy --version`, and `proxy check` built on the configuration code above, with tests for every validation rule. Stubs (`#[ignore]`) for the acceptance tests of M1–M6.

**Done when:** `proxy check --config proxy.toml` prints `config ok: 1 listener(s), 2 route(s), 2 pool(s)` and exits with 0; each invalid example of this lesson exits with 1 and the shown message.

#### M1 — Forward HTTP/1.1 to one pool

**Goal.** Accept HTTP/1.1, forward to the backends of one pool with round robin, reuse backend connections, and apply rule 2 (headers).

<details>
<summary>Hint</summary>

Use `hyper` 1 with `hyper-util`'s client and its connection pool for the backend side. Let `test-backend` answer with a header `x-backend: <its name>` and echo the headers it received, so a test can check both the balancing and the header rules.

</details>

**Done when:** with three `test-backend`s, 30 requests through the proxy show 10 answers from each; a request with `Connection: close, x-secret` reaches the backend without `x-secret`, with `X-Forwarded-For: 127.0.0.1` and an `x-request-id`.

```bash
curl -s -D - http://127.0.0.1:8080/ -o /dev/null | grep -i x-backend
```

```text
x-backend: b1
```

#### M2 — Routing and HTTP/2

**Goal.** Several routes and pools (rule 1); HTTP/2 from clients next to HTTP/1.1; the `least_requests` algorithm.

<details>
<summary>Hint</summary>

Keep the router in `proxy-core` as a pure function from (host, path, configuration) to a route, and test it with a table of cases. For `least_requests`, the cheap and well-known variant is "power of two choices": pick two backends at random and take the one with fewer requests in flight.

</details>

**Done when:** `curl -H 'Host: api.example.test' http://127.0.0.1:8080/v1/users` reaches pool `api` and `/health` with the same host reaches `web`; `curl --http2-prior-knowledge http://127.0.0.1:8080/` succeeds and the access log shows `HTTP/2.0`; a path that no route matches gives `404 {"error":"no route"}`.

#### M3 — Health, timeouts and safe retries

**Goal.** Active health checks, outlier detection, route timeouts, the retry rules (rule 4) and the error codes (rule 3).

<details>
<summary>Hint</summary>

Count attempts in `test-backend` and expose them (`GET /attempts`), so that a test can prove "this `POST` reached a backend exactly once". Wrap the whole forward in `tokio::time::timeout` for `504`.

</details>

**Done when:** while a load of `GET`s runs, killing one of three backends causes no client error after the first health interval; a backend told to sleep 3 s with `timeout_ms = 1000` gives `504` after about 1 s; a failing `POST` is sent to a backend once, a failing `GET` with `retries = 1` twice, to two different backends; with all backends down the answer is `503`.

#### M4 — TLS with SNI and ALPN

**Goal.** TLS termination with `rustls`, one certificate per host name on the same address, HTTP/2 or HTTP/1.1 chosen by ALPN.

<details>
<summary>Hint</summary>

Generate test certificates with `rcgen` in the test itself (a small test certificate authority and one certificate per name). `rustls` lets you choose the certificate per connection from the SNI name with a certificate resolver.

</details>

**Done when:** `curl --resolve api.example.test:8443:127.0.0.1 --cacert ca.pem https://api.example.test:8443/` and the same for `www.example.test` show two different certificate subjects in `curl -v`; `curl -v` reports `ALPN: server accepted h2`; `proxy check` rejects a configuration whose certificate file does not exist or has already expired.

#### M5 — Hot reload, rate limits and the admin API

**Goal.** Rules 6 and 7 for reload, a token bucket per client address and route, and the admin API.

<details>
<summary>Hint</summary>

Put the whole validated configuration (routes, pools, limits) into one `Arc<Config>` and store it in an `ArcSwap`. Keep long-lived state that must survive a reload (health of backends, rate-limit buckets) outside the configuration, keyed by backend address and route name.

</details>

**Done when:** during `oha -q 500 -z 60s http://127.0.0.1:8080/`, a script that rewrites the configuration and sends `SIGHUP` 20 times leads to zero non-`2xx` answers in the `oha` summary; `POST /reload` with an invalid file returns `400` with the error and traffic continues with the old rules; a client above `rate_limit_per_second` receives `429` with `Retry-After`.

#### M6 — Performance and operations

**Goal.** Metrics per route and backend, an access log, graceful shutdown, a load test with a baseline, and a profile.

<details>
<summary>Hint</summary>

Measure in your Linux VM `dev` (Lesson 11.1) as well as on the Mac, with fixed rates (the open model of Lesson 19.2), for example 1,000, 5,000 and 20,000 requests per second. Record the hardware, the number of connections, response size and duration. Profile at the highest rate that still meets your latency goal.

</details>

**Done when:** `docs/results.md` has a table "direct to backend vs through proxy" with p50, p99 and p99.9 at three fixed rates, plus a flame graph and one paragraph that explains the largest cost in it; a test starts a 2-second request, sends `SIGTERM`, and checks that the request completes with `200` while a new connection is refused; `/metrics` shows request counts and latency histograms per route and per backend.

### Quality bar

| Area | What it means here |
|---|---|
| Unit and property tests | router table tests; property tests for balancing (round robin is fair over any number of requests; ejected backends never get traffic) |
| Integration tests | real `proxy` and `test-backend` processes on free ports, for every milestone |
| Failure tests | backend killed, slow, returning `500`, closing the connection mid-response; client sending headers slowly |
| Fuzzing | the configuration loader (`load` must never panic) and your header rewriting code |
| Benchmarks | proxy overhead at fixed rates with `oha`; `criterion` for the router and the balancer |
| Observability | access log with request ID (identifier), route, backend, status and duration; metrics per route and backend |
| Security | header read timeout (slowloris), limits on header size, body size and connections; hop-by-hop headers never forwarded; admin API only on `127.0.0.1` |

> 🔒 **Security:** A proxy is the first program an attacker talks to. Never trust `X-Forwarded-For` from a client as the client's address (anyone can send it); append to it, and use the real peer address for rate limits. Reject requests that `hyper` reports as malformed instead of trying to "repair" them.

### Stretch goals

- **Zero-downtime binary upgrade:** start the new version, hand over the listening sockets (or use `SO_REUSEPORT`), let the old one drain.
- **Consistent hashing with bounded loads** for the `consistent_hash` algorithm (Lesson 18.8).
- **mTLS (mutual TLS) to backends**, with client certificates.
- **HTTP/3** with `quinn` (Lesson 15.13) on the listener side.
- **A Kubernetes controller** that writes your configuration from Gateway API resources (controllers are Lesson 22.26; the Gateway API is the standard set of Kubernetes resources for routing traffic into a cluster, documented at gateway-api.sigs.k8s.io).

## Real systems to study

| System | What to look at |
|---|---|
| [Envoy](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/http/http_routing) | the architecture overview: routing, [outlier detection](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier), [load balancers](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers), retry budgets |
| [Pingora](https://github.com/cloudflare/pingora) | a proxy framework in Rust (Lesson 22.20); read the [blog post on why it was built](https://blog.cloudflare.com/how-we-built-pingora-the-proxy-that-connects-cloudflare-to-the-internet/) |
| [linkerd2-proxy](https://github.com/linkerd/linkerd2-proxy) | a service-mesh sidecar proxy in Rust, built on `hyper` and `tower` |
| [HAProxy](https://docs.haproxy.org/) | the configuration manual: health checks, timeouts and the many kinds of limits |

## Common mistakes

**1. Retrying a `POST`.** The backend may have processed it before the connection broke; a retry creates a second order. Rule 4 allows retries only for `GET`, `HEAD` and `OPTIONS`.

**2. Retries without a budget.** During an outage every request is tried twice or three times, and the remaining backends get two or three times the load — the outage spreads. Fix: the retry budget.

**3. Forwarding hop-by-hop headers.** A forwarded `Connection: close` or `Transfer-Encoding` confuses the backend connection and can open the door to request smuggling. Fix: rule 2, with a test.

**4. Holding a lock across `.await`.** A `std::sync::Mutex` around the backend list, held while the request is forwarded, serialises all traffic. Fix: read an `Arc` snapshot of the state and drop every guard before awaiting.

**5. All health checks at the same moment.** A hundred backends checked at exactly the same time every second create spikes. Fix: add jitter to the interval.

**6. Measuring without a baseline.** "p99 is 4 ms" says nothing about the proxy if the backend alone already has a p99 of 3.9 ms. Fix: always the direct-to-backend comparison.

## Exercises

Do these in `code/p23-proxy-config` before M0. They prepare the router and the validation of your capstone.

### Exercise 1 — Which pool? (easy)

With the `proxy.toml` of this lesson and rule 1, which pool (or which status code) does each request get? (a) `GET http://api.example.test/v1/users` (b) `GET http://api.example.test:8080/health` (c) `GET http://shop.example.test/v1/users` (d) `POST http://api.example.test/v1` (no slash at the end).

<details>
<summary>Hint</summary>

The host is compared without the port. The prefix `/v1/` has a slash at the end. The second route has no host, so it matches every host.

</details>

<details>
<summary>Answer</summary>

(a) `api`: the host matches and `/v1/users` starts with `/v1/`. (b) `web`: the host (without `:8080`) matches, but `/health` does not start with `/v1/`; the second route matches everything. (c) `web`: the first route requires the host `api.example.test`. (d) `web`: `/v1` does not start with `/v1/`. Case (d) is a classic surprise; your design document should say whether that is what you want, and a router test should pin it down.

</details>

### Exercise 2 — Write tests for the validation (medium)

Add a test module to `src/main.rs` with three tests: a route to an unknown pool is rejected with the exact message, a pool without backends is rejected, and a route without `timeout_ms` gets 5,000 ms. Build the TOML text inside the tests.

<details>
<summary>Hint</summary>

A raw string `r#"…"#` can contain quotes. Keep the listener and the route in one constant and add pools with a small helper function, so that each test shows only what is special about it. Remember that `pool` is a required field: a file without any `[[pool]]` fails while parsing, not in your validation.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    /// A listener and one route to the pool "web"; each test adds pools.
    const ROUTE_TO_WEB: &str = r#"
[[listener]]
address = "127.0.0.1:8080"

[[route]]
path_prefix = "/"
pool = "web"
"#;

    fn pool(name: &str, backends: &str) -> String {
        format!(
            "[[pool]]\nname = \"{name}\"\nalgorithm = \"round_robin\"\nbackends = [{backends}]\n"
        )
    }

    #[test]
    fn a_route_to_an_unknown_pool_is_rejected() {
        let text = format!("{ROUTE_TO_WEB}{}", pool("api", r#""127.0.0.1:9001""#));
        assert_eq!(
            load(&text).unwrap_err(),
            r#"route "/" uses unknown pool "web""#
        );
    }

    #[test]
    fn a_pool_without_backends_is_rejected() {
        let text = format!("{ROUTE_TO_WEB}{}", pool("web", ""));
        assert_eq!(load(&text).unwrap_err(), r#"pool "web" has no backends"#);
    }

    #[test]
    fn the_default_timeout_is_five_seconds() {
        let text = format!("{ROUTE_TO_WEB}{}", pool("web", r#""127.0.0.1:9101""#));
        let config = load(&text).unwrap();
        assert_eq!(config.route[0].timeout_ms, 5_000);
    }
}
```

```text
running 3 tests
...
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: each test builds the smallest file that shows one rule, and compares the whole error message, so a changed message is noticed (users and scripts read these messages).

</details>

### Exercise 3 — Duplicate pool names, test first (medium)

Two `[[pool]]` tables with the same `name` are accepted today, and nobody knows which one a route would use. Write the test `two_pools_with_the_same_name_are_rejected` (expected message: `pool "web" is defined twice`), watch it fail, then change `load`.

<details>
<summary>Hint</summary>

`HashSet::insert` returns `false` when the value was already there. Build the set of pool names with a loop instead of `collect`, and stop at the first duplicate.

</details>

<details>
<summary>Answer</summary>

The test, in the module of Exercise 2:

```rust
#[test]
fn two_pools_with_the_same_name_are_rejected() {
    let web = pool("web", r#""127.0.0.1:9101""#);
    let text = format!("{ROUTE_TO_WEB}{web}{web}");
    assert_eq!(load(&text).unwrap_err(), r#"pool "web" is defined twice"#);
}
```

Before the change, `load` returns `Ok`, and the test fails with:

```text
called `Result::unwrap_err()` on an `Ok` value: Config { listener: [Listener { address: 127.0.0.1:8080 }], route: [Route { host: None, path_prefix: "/", pool: "web", timeout_ms: 5000, retries: 0 }], pool: [Pool { name: "web", algorithm: RoundRobin, backends: [127.0.0.1:9101] }, Pool { name: "web", algorithm: RoundRobin, backends: [127.0.0.1:9101] }] }
```

In `load`, replace the line that builds `pools` with:

```rust
let mut pools = HashSet::new();
for pool in &config.pool {
    if !pools.insert(pool.name.as_str()) {
        return Err(format!("pool {:?} is defined twice", pool.name));
    }
}
```

Why it works: the loop builds the same set as before, but notices the moment a name is inserted the second time. All four tests pass.

</details>

### Exercise 4 — Retries and the budget (hard)

A pool has three backends and receives 1,000 `GET` requests per second; every route has `retries = 2`. One backend starts to fail every request. (a) Without a retry budget, how many requests per second do the two healthy backends receive together, before outlier detection ejects the bad one? (b) With a budget of 10%? (c) Why does the budget matter even more when *all* backends are overloaded?

<details>
<summary>Hint</summary>

About a third of the requests first go to the bad backend. Each of them is retried on another backend. Under overload, a retry is one more request to a server that is already too slow.

</details>

<details>
<summary>Answer</summary>

(a) About 667 requests per second go directly to the two good backends, and about 333 go to the bad one first; each of those fails and is retried, on a *different* backend (rule 4), which is a good one. So the good backends receive about 667 + 333 = 1,000 requests per second, and the bad one 333 — the total load on backends is 1,333 (one third more). If the retry could land on the bad backend again, some requests would use their second retry too. (b) With a 10% budget, at most 100 retries per second are allowed: the good backends receive about 767 per second, and about 233 requests per second fail with `502` until the bad backend is ejected. (c) When all backends are slow because of overload, retries add load exactly where there is no capacity, which makes them slower, which causes more timeouts and more retries — a *retry storm*. The budget caps this feedback loop. This is why Envoy and other proxies use retry budgets, and why your design document must state yours.

</details>

## Quiz

1. What are the downstream and the upstream of a proxy?

<details>
<summary>Answer</summary>

Downstream is the client side (who sends requests to the proxy); upstream is the backend side (the servers the proxy forwards to).

</details>

2. Why does the proxy remove hop-by-hop headers?

<details>
<summary>Answer</summary>

They describe one connection only (for example `Connection` or `Transfer-Encoding`). Forwarding them would describe the wrong connection, confuse the backend and can enable request smuggling.

</details>

3. Which requests may the proxy retry, and under which conditions?

<details>
<summary>Answer</summary>

Only `GET`, `HEAD` and `OPTIONS`; only if no byte of the response has reached the client; at most `retries` times; to a different backend; and only while the retry budget allows it.

</details>

4. What do SNI and ALPN do in M4?

<details>
<summary>Answer</summary>

SNI tells the proxy which host name the client wants, so it can choose the right certificate on a shared address. ALPN lets client and proxy agree on HTTP/2 or HTTP/1.1 during the TLS handshake.

</details>

5. Why does each request load the configuration from the `ArcSwap` once, at the start?

<details>
<summary>Answer</summary>

So that a reload never changes the rules in the middle of a request. The request keeps its `Arc` to the old configuration until it ends; new requests see the new one.

</details>

6. Why must performance results compare "through the proxy" with "direct to backend"?

<details>
<summary>Answer</summary>

Because the proxy's own cost is the difference. A number for the whole path mixes the backend's latency with the proxy's.

</details>

7. Why does `#[serde(deny_unknown_fields)]` matter for a proxy configuration?

<details>
<summary>Answer</summary>

Without it, a misspelled key is ignored and a default is used silently, for example a missing timeout. With it, `proxy check` and reloads reject the file and say which key is wrong.

</details>

## Summary

- Option 2 is an HTTP/1.1 and HTTP/2 reverse proxy with TLS (SNI, ALPN), routing, pools with balancing, health checks, safe retries, rate limits, hot reload and an admin API.
- The architecture separates listeners, limits, a pure router and balancer, upstream pools, health state and a configuration behind an `ArcSwap`.
- The contract fixes the command line, the configuration file, seven forwarding rules and the admin API; the configuration loader with strict parsing and validation is the first code you write.
- Milestones: M0 `proxy check`, M1 HTTP/1.1 forwarding, M2 routing and HTTP/2, M3 health, timeouts and retries, M4 TLS, M5 reload and rate limits, M6 performance and operations.
- Retries need rules and a budget; proxy performance is always reported as overhead against a direct baseline.
- Study Envoy, Pingora, linkerd2-proxy and HAProxy after you have designed each part yourself.

## Sources and further reading

- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110) — methods and idempotency, status codes, `Connection` and the rules for intermediaries such as proxies.
- [RFC 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112) — message framing, `Content-Length` and `Transfer-Encoding`, and why ambiguous framing must be rejected.
- [hyper](https://hyper.rs/) — the HTTP library under most Rust proxies; guides for clients and servers.
- [rustls documentation](https://docs.rs/rustls/latest/rustls/) — server configuration, certificate resolvers and ALPN.
- [arc-swap documentation](https://docs.rs/arc-swap/latest/arc_swap/) — the pattern "read the current configuration very often, replace it rarely".
- [serde attributes](https://serde.rs/attributes.html) and [toml documentation](https://docs.rs/toml/latest/toml/) — `deny_unknown_fields`, defaults and renaming used in the configuration code.

---

[🏠 Start](../START-HERE.md) · [← 23.3 Capstone option 1: a distributed key-value store](03-capstone-distributed-key-value-store.md) · [📋 Phase 23](README.md) · [23.5 Capstone option 3: a message broker →](05-capstone-message-broker.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
