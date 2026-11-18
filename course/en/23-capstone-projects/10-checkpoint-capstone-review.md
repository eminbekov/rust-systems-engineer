<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 23.9 Presenting your work](09-presenting-your-work.md) · [📋 Phase 23](README.md) · [Phase 24 — Career and open source →](../24-career-and-open-source/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/23-capstone-projects/10-checkpoint-capstone-review.md) <!-- nav -->

# Lesson 23.10 — Checkpoint: capstone review

> **Phase 23 — Capstone projects** · Lesson 10 of 10 · ⏱ about 90 minutes

## What you will learn

- How to review a finished capstone the way an experienced engineer would, with a rubric of eight criteria.
- How to check, mechanically, that a stranger can build and test your project and that every milestone tag still builds.
- Whether you can answer the questions that this phase prepared you for, from the rules of a capstone to presenting it.
- What to fix before you call the phase done, and where to go next.

## Before you start

- You have finished two capstones (Lessons [23.3](03-capstone-distributed-key-value-store.md)–[23.7](07-capstone-your-own-idea.md)), or one finished and the second at least at M3. Each has its own Git repository with milestone tags, a README, `docs/design.md`, `docs/adr/` and `docs/results.md`.
- Docker Desktop is running (for Task 1), and you know `docker run` with a mounted folder from [Lesson 11.12](../11-linux-shell-and-containers/12-containers-and-docker-basics.md).
- The review itself fits one sitting; the fixes it finds (Task 4) may need a few more.

## New words

| Word | Meaning |
|---|---|
| **rubric** | A table of criteria with a description of each level (here 0, 1 or 2 points), so that a review is the same whoever does it. |
| **fresh-clone test** | Cloning the repository into an empty folder and building and testing it in a clean environment, to prove that nothing depends on your own machine. |
| **self-review** | Reviewing your own work with a fixed checklist and some distance — after a night of sleep, as if someone else had written it. |
| **peer review** | A review by another person, ideally another learner or an engineer who did not build the project. |
| **ADR** (Architecture Decision Record) | A short file that records one important decision: context, decision, consequences (Lesson 23.2). |
| **CI** (Continuous Integration) | A server that runs your checks automatically on every push (Lesson 09.12). |
| **QPS** (Queries Per Second) | How many requests a system receives or serves per second. |
| **p99** | The 99th percentile of latency: 99% of requests are faster than this value (Lesson 19.1). |
| **MVP** (Minimum Viable Product) | The smallest version that is already useful; milestone M1 of a capstone (Lesson 23.7). |
| **HTTP** (HyperText Transfer Protocol) | The request-and-response protocol of the web; HTTPS is HTTP over an encrypted connection. |

## 1. The review rubric

Score each capstone with this rubric. Each row gives 0, 1 or 2 points; a capstone is complete with **at least 12 of 16 points and no row at 0**.

| Criterion | 0 points | 1 point | 2 points |
|---|---|---|---|
| Problem and scope | unclear what it does | pitch and non-goals exist | pitch, non-goals and a cut list; scope matches the result |
| Design document | missing or a code tour | goals, design and failure modes | also estimates, alternatives, test plan, open questions; kept up to date with ADRs |
| Contract and tests | few tests, no contract | acceptance tests for each milestone | contract tests or model-based tests, property tests, and a test matrix without "missing" rows of high risk |
| Failure handling | failures not tested | some failure tests | every row of the failure-mode table has a test (crash, timeout, bad input, overload) |
| Performance evidence | no numbers | numbers without methodology | methodology, baseline, percentiles at fixed rates, a profile with an explanation |
| Observability and operations | none | logs | structured logs, metrics, health checks, graceful shutdown, a way to run it (Compose, kind) |
| Code quality | CI red or missing | CI green: fmt, clippy, tests | also fuzzing, supply-chain checks, no `unwrap` on outside input, `// SAFETY:` on every `unsafe` |
| Documentation and presentation | README missing | README with quick start | README by the structure of Lesson 23.9, honest claims, a write-up or a recorded demo |

> 💡 **Tip:** Score yourself first, write the scores into `docs/review.md`, and then ask someone else to score the same project without seeing yours. The rows where the two scores differ are the ones to look at.

## Self-test quiz

**The rules and the plan (Lesson 23.1)**

1. How many capstones does this phase ask for, of which kinds, and how long should each take?

<details>
<summary>Answer</summary>

Two: one deep in your Phase 22 specialization and one in a different area; each 4–7 weeks at about 10 hours per week.

</details>

2. What is milestone M0 in every capstone, and what must it contain?

<details>
<summary>Answer</summary>

The walking skeleton: a workspace with a core crate and a binary, an acceptance test that runs the real binary, the later milestones' tests as `#[ignore]` stubs, and CI.

</details>

3. A milestone takes twice its time box. What do you do, in which order, and what do you never do?

<details>
<summary>Answer</summary>

Make the milestone smaller (move a part to the stretch goals), use some of the buffer, or move a later optional milestone to the stretch goals. Never cut the quality bar.

</details>

**Design documents (Lesson 23.2)**

4. What makes a requirement checkable? Give an example.

<details>
<summary>Answer</summary>

A measure, a number with a unit, a condition and a test: "p99 latency of `GET` below 5 ms at 10,000 QPS on one node, measured with a 60-second load test".

</details>

5. Which five columns does a failure-mode table have?

<details>
<summary>Answer</summary>

Component, failure, how we notice, what the user sees, what we do.

</details>

6. Why are ADRs never edited, and what happens when a decision changes?

<details>
<summary>Answer</summary>

They record what was decided and why at that moment; editing them would erase the history. A new ADR supersedes the old one, and the old one is marked as superseded.

</details>

**The options (Lessons 23.3–23.7)**

7. In the key-value store, why must the deduplication table be part of the replicated state machine and of snapshots?

<details>
<summary>Answer</summary>

Because after a leader change or a restart from a snapshot, the node must still recognise retried requests. A table only in the leader's memory, or missing from the snapshot, lets a retry be applied twice.

</details>

8. In the proxy, which requests may be retried, and why is a retry budget needed?

<details>
<summary>Answer</summary>

Only `GET`, `HEAD` and `OPTIONS`, before any response byte reached the client, to a different backend. The budget caps retries at a share of all requests, so that during an outage retries cannot multiply the load (a retry storm).

</details>

9. In the broker, what is the high watermark, and why may consumers read only below it?

<details>
<summary>Answer</summary>

The first offset that not every in-sync replica has yet; every record below it is on all in-sync replicas. Records at or above it can disappear in a failover; a consumer that read them would have seen data that was later lost.

</details>

10. In the observability pipeline, when may a file checkpoint move past a line?

<details>
<summary>Answer</summary>

Only when every sink that receives events from that line has acknowledged them (end-to-end acknowledgement). Then a crash causes duplicates, never loss.

</details>

11. Which seven questions decide whether your own idea is a good capstone?

<details>
<summary>Answer</summary>

Is it a system? Does it have testable failure modes? Can it be measured? Is the core your code? Is an MVP reachable in about two weeks (and the whole in 4–7 weeks)? Can it be demonstrated in two minutes? Is there a real system to compare with?

</details>

**Hardening (Lesson 23.8)**

12. A round-trip property test reports `minimal failing input: len = 65536`. What kind of bug do you suspect first, and what do you do with the regression file?

<details>
<summary>Answer</summary>

A 16-bit length that is silently truncated (an `as u16` cast), because 65,536 is the first value that does not fit. Fix it with `try_from` and commit `proptest-regressions/`, so the case is tried first in every later run.

</details>

13. What belongs in the fast lane of CI, and what in the slow lane?

<details>
<summary>Answer</summary>

Fast lane, on every push: format, Clippy, unit and integration tests, quick acceptance checks. Slow lane, after it and longer at night: simulation seeds, fuzzing, benchmark builds.

</details>

14. Name five things a trustworthy benchmark write-up contains.

<details>
<summary>Answer</summary>

Any five of: hardware and software versions, release build, the exact workload with a fixed rate, warm-up, at least three runs with median and spread, a baseline, percentiles (not averages), a profile with an explanation, and an honest list of limits.

</details>

**Presenting (Lesson 23.9)**

15. What must the first screen of your README make clear, and how do you keep its code examples true?

<details>
<summary>Answer</summary>

What the project is, what it guarantees, that it is a learning project, and one diagram. Examples stay true when the README is included as crate documentation (`#![doc = include_str!("../README.md")]`), so its `rust` blocks run as doc tests; every other block gets a language label.

</details>

16. Which three kinds of questions come after a capstone talk, and where do the answers come from?

<details>
<summary>Answer</summary>

"What happens when X fails?" (failure-mode table), "Why not Y?" (ADRs and alternatives), "How do you know?" (test matrix and results).

</details>

## Practical tasks

### Task 1 — The fresh-clone test

Prove that your capstone builds and passes its tests on a machine that is not yours. Clone it into an empty temporary folder and run the tests in a clean container from the official `rust:1.100` image. Try it first with `code/p23-skeleton` from Lesson 23.1.

<details>
<summary>Hint</summary>

`mktemp -d` creates an empty temporary folder. `git clone <path-to-repo> <folder>` takes only what is committed — which is exactly the point. Mount the clone into the container with `-v "$PWD":/work -w /work`, and give the container a name that starts with `course-`.

</details>

<details>
<summary>Answer</summary>

```bash
cd "$(mktemp -d)"
git clone --quiet ~/projects/rust/code/p23-skeleton fresh
cd fresh
docker run --rm --name course-fresh-clone -v "$PWD":/work -w /work rust:1.100 cargo test --quiet
```

```text
running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s


running 5 tests
i....
test result: ok. 4 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.00s


running 1 test
.
test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s


running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(This output is from the skeleton with the `--help` test of Lesson 23.1; your counts will differ.) Typical failures of this test are the most valuable findings of the whole review: a file that is in `.gitignore` but needed, a test that depends on a path in your home folder, a tool that is installed only on your Mac, or a `Cargo.lock` that was never committed. `--rm` removes the container when it finishes; delete the temporary folder afterwards.

</details>

### Task 2 — Every milestone still builds

The milestone tags are a promise that a reader can check out any milestone and see it work. Write a loop that checks out each tag `m0`, `m1`, … in turn, runs the tests, prints `ok` or `FAILED` per tag, and returns to `main`.

<details>
<summary>Hint</summary>

`git tag --list 'm*'` lists the milestone tags. `git switch --detach <tag>` checks out a tag without creating a branch. The working tree must be clean before you start (`git status --short` prints nothing), or `git switch` refuses.

</details>

<details>
<summary>Answer</summary>

```bash
for tag in $(git tag --list 'm*'); do
    git switch --quiet --detach "$tag"
    if cargo test --quiet >/dev/null 2>&1; then echo "$tag ok"; else echo "$tag FAILED"; fi
done
git switch --quiet main
```

For a repository with two milestone tags (the skeleton of Lesson 23.1 has only `m0`, so it prints one line):

```text
m0 ok
m1 ok
```

Why it matters: a tag whose tests fail means that a later change was "fixed" by editing history in your head, not in Git, or that the tests depend on something outside the repository. Note that `git tag --list` sorts by name, so `m10` would come before `m2`; with more than ten milestones, use `git tag --list 'm*' --sort=creatordate`.

</details>

### Task 3 — Turn a missing row into a test

Take one row marked **missing** in your test matrix (Lesson 23.8) and write the test. If your matrix has none left, use this example from the broker: "a client sends a 4 GiB length field" — write `frame_len(header: [u8; 4]) -> Result<usize, String>` that refuses lengths above `MAX_FRAME` (1 MiB) **before** anything is allocated, test first.

<details>
<summary>Hint</summary>

Test the boundary on both sides: the largest allowed value must pass, the value `0xffff_ffff` must fail with a clear message. `u32::from_be_bytes` turns the four header bytes into the length.

</details>

<details>
<summary>Answer</summary>

```rust
/// The largest frame the broker accepts, in bytes.
pub const MAX_FRAME: u32 = 1024 * 1024;

/// Reads the 4-byte big-endian length at the start of a frame and checks it
/// against `MAX_FRAME` before anything is allocated.
pub fn frame_len(header: [u8; 4]) -> Result<usize, String> {
    let len = u32::from_be_bytes(header);
    if len > MAX_FRAME {
        return Err(format!("frame of {len} bytes is larger than {MAX_FRAME}"));
    }
    Ok(len as usize)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_4_gib_length_is_refused() {
        assert_eq!(
            frame_len([0xff; 4]),
            Err("frame of 4294967295 bytes is larger than 1048576".to_string())
        );
    }

    #[test]
    fn the_maximum_itself_is_allowed() {
        assert_eq!(frame_len(MAX_FRAME.to_be_bytes()), Ok(1_048_576));
    }
}
```

```text
running 2 tests
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the function only reads four bytes and compares a number, so a hostile length costs nothing. The two tests pin both sides of the boundary, which is where off-by-one mistakes (`>=` instead of `>`) hide. Mark the row as done in the matrix and link the test.

</details>

### Task 4 — Score and fix

Score both capstones with the rubric of Section 1 in `docs/review.md`: one line per criterion with the points and one sentence of evidence (a link to a test, a file or a number). For every row below 2 points, write the smallest change that would raise it, and do the two most valuable changes now.

<details>
<summary>Hint</summary>

"Evidence" means something a reader can open: `tests/simulation.rs`, `docs/results.md#…`, a CI run. If you cannot point to anything, the honest score for that row is lower than you think.

</details>

<details>
<summary>Answer</summary>

An example for a proxy capstone (yours will differ):

```markdown
# Review of p23-proxy (2026-12-20)

| Criterion | Points | Evidence |
|---|---|---|
| Problem and scope | 2 | README first paragraph; non-goals and cut list in docs/proposal.md |
| Design document | 2 | docs/design.md (status: accepted, updated 12-18), 7 ADRs |
| Contract and tests | 2 | 64 tests; router property tests; test matrix has no high-risk gaps |
| Failure handling | 1 | slow, dead and 500 backends tested; slow client NOT tested |
| Performance evidence | 2 | docs/results.md: baseline vs proxy at 3 rates, flame graph |
| Observability and operations | 1 | metrics and access log; no graceful-shutdown test |
| Code quality | 2 | CI green: fmt, clippy, tests, cargo deny, fuzz (config loader) |
| Documentation and presentation | 1 | README complete; no write-up or demo yet |
| **Total** | **13 / 16** | complete (no row at 0) |

Next changes:
1. Add the slowloris test (header timeout) -> Failure handling 2.
2. Add the SIGTERM drain test from M6 -> Observability and operations 2.
3. Record the two-minute demo with asciinema -> Documentation 2.
```

The two most valuable changes are usually the missing failure tests: they protect users, and they are exactly what interviewers ask about.

</details>

## If you got something wrong

| Topic | Review |
|---|---|
| The rules, the walking skeleton, milestones, time planning | [Lesson 23.1 — How capstone projects work](01-how-capstone-projects-work.md) |
| Requirements, estimates, failure modes, ADRs | [Lesson 23.2 — Writing a design document](02-writing-a-design-document.md) |
| Replication, idempotency, read index, snapshots | [Lesson 23.3 — Capstone option 1: a distributed key-value store](03-capstone-distributed-key-value-store.md) |
| Routing, retries, health checks, hot reload | [Lesson 23.4 — Capstone option 2: a high-performance HTTP reverse proxy](04-capstone-http-reverse-proxy.md) |
| Offsets, consumer groups, the high watermark | [Lesson 23.5 — Capstone option 3: a message broker](05-capstone-message-broker.md) |
| Sources and sinks, checkpoints, buffers, backpressure | [Lesson 23.6 — Capstone option 4: an observability pipeline](06-capstone-observability-pipeline.md) |
| Choosing and scoping an idea, proposals, check scripts | [Lesson 23.7 — Capstone option 5: your own idea](07-capstone-your-own-idea.md) |
| Test matrix, property tests, CI lanes, benchmark methodology | [Lesson 23.8 — Testing, benchmarking and hardening your capstone](08-testing-benchmarking-and-hardening-capstones.md) |
| README, doc tests, write-up, demo, talk, publishing | [Lesson 23.9 — Presenting your work](09-presenting-your-work.md) |

## What's next

With two reviewed capstones you have the centre of a portfolio. [Phase 24 — Career and open source](../24-career-and-open-source/README.md) builds on it: [Lesson 24.1 — Roles for systems engineers](../24-career-and-open-source/01-roles-for-systems-engineers.md) shows which roles your capstones fit, the following lessons turn them into a portfolio and a résumé, and you make your first contribution to a real open-source project — often one of the systems you studied for your capstone. Keep the capstones alive: fix issues that readers report, and add a stretch goal now and then. A project that is still maintained a year later says even more than one that was finished once.

## Sources and further reading

- [Google Engineering Practices — What to look for in a code review](https://google.github.io/eng-practices/review/reviewer/looking-for.html) — a reviewer's checklist that complements the rubric.
- [The Cargo Book — `cargo test`](https://doc.rust-lang.org/cargo/commands/cargo-test.html) — options such as `--quiet`, `--doc` and `--include-ignored` used in the tasks.
- [The Cargo Book — Cargo.toml vs Cargo.lock](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html) — why a binary project commits its `Cargo.lock`, which the fresh-clone test depends on.
- [Docker — `docker container run`](https://docs.docker.com/reference/cli/docker/container/run/) and the [official `rust` image](https://hub.docker.com/_/rust) — mounts, working directories and image tags for Task 1.

---

[🏠 Start](../START-HERE.md) · [← 23.9 Presenting your work](09-presenting-your-work.md) · [📋 Phase 23](README.md) · [Phase 24 — Career and open source →](../24-career-and-open-source/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
