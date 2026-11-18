<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 23 — Capstone projects](README.md) · [📋 Phase 23](README.md) · [23.2 Writing a design document →](02-writing-a-design-document.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/23-capstone-projects/01-how-capstone-projects-work.md) <!-- nav -->

# Lesson 23.1 — How capstone projects work

> **Phase 23 — Capstone projects** · Lesson 1 of 10 · ⏱ about 135 minutes

## What you will learn

- What a capstone project is, and how it is different from the projects you built so far.
- The rules of this phase: two capstones, 4–7 weeks each, and how to choose them.
- The life of a capstone: design document, milestones with "Done when" checks, the quality bar, results and presentation.
- How to set up a capstone repository as a Cargo workspace with a *walking skeleton* and acceptance tests.
- How to plan your time, keep a work log, and what to do when you are stuck or late.

## Before you start

- You finished [Phase 22 — Specializations](../22-specializations/README.md) and its checkpoint, [Lesson 22.30 — Checkpoint: your specialization](../22-specializations/30-checkpoint-your-specialization.md).
- You know Cargo workspaces from [Lesson 08.12 — Project layout and workspaces](../08-software-design/12-project-layout-and-workspaces.md) and end-to-end tests of programs from [Lesson 09.2 — End-to-end tests for command-line programs](../09-testing-debugging-benchmarking/02-testing-command-line-programs.md).
- You built milestone-based projects before, for example [Lesson 17.14 — Project: your own key-value store](../17-storage-and-databases/14-project-key-value-store.md) and [Lesson 15.15 — Project: a TCP load balancer](../15-networking/15-project-tcp-load-balancer.md).

## New words

| Word | Meaning |
|---|---|
| **capstone project** | A large final project that combines many skills from a whole course. In this phase you build two. |
| **portfolio** | A small collection of your best public work that shows employers what you can do. |
| **scope** | The list of what a project will do. Everything outside the scope is not built. |
| **scope creep** | When a project slowly grows new features that nobody planned, so it never finishes. |
| **non-goal** | Something you decide, in writing, *not* to build. It protects the scope. |
| **walking skeleton** | The smallest version of the whole system that builds, runs and is tested end to end. It does almost nothing, but every part is connected. |
| **milestone** | A planned point in a project where a clear piece of work is finished and checked. Here: M0, M1, M2, … |
| **acceptance test** | A test that checks one promise from the contract from the outside, the way a user would. A milestone is done when its acceptance tests pass. |
| **quality bar** | The minimum quality every capstone must reach: tests, lints, benchmarks, documentation, CI and observability. |
| **definition of done** | A short list of conditions that must all be true before you call a piece of work finished. |
| **time box** | A fixed amount of time for a task. When the time is over, you stop and decide what to do next. |
| **buffer** | Time in a plan that is reserved for surprises. |
| **work log** | A file where you write, after every session, what you did, how long it took and what is next. |
| **annotated tag** | A Git label with a message, attached to one commit, for example `m1`. It marks a finished milestone. |
| **CI** (Continuous Integration) | A server that runs your checks (format, lints, tests) automatically on every push. You built CI pipelines in Lesson 09.12. |
| **MVP** (Minimum Viable Product) | The smallest version of a product that is already useful to a real user. |
| **TCP** (Transmission Control Protocol) | The internet protocol that gives two programs a reliable, ordered stream of bytes. |
| **HTTP** (HyperText Transfer Protocol) | The request-and-response protocol of the web; HTTPS is HTTP over an encrypted connection. |
| **AI** (Artificial Intelligence) | Here: an AI assistant, a program that answers questions in natural language. Use it as a reviewer, not as the author. |
| **CLI** (Command-Line Interface) | A way to use a program by typing commands and options in a terminal. |

## 1. What a capstone is

All projects so far were *exercises*. Each one practised a few ideas from one phase, the lesson gave you the contract, and the milestones were small. A capstone is different. It is a **complete system** that you design, build, test, measure and explain yourself. It uses many phases at once.

Think of the difference between cooking one dish from a recipe and running a small restaurant for a week. The dishes are the same skills, but now you plan the menu, buy the food, keep the kitchen clean and serve real guests.

| | Projects so far | Capstone |
|---|---|---|
| Who writes the contract | the lesson | the lesson gives a reference contract; you adapt it in your design document |
| Size | one to three sittings | 4–7 weeks of about 10 hours |
| Phases used | mostly one | five or more |
| Tests | unit and end-to-end tests | every level: unit, property, end-to-end, failure, load, fuzz |
| Documents | a short README | README, design document, decision records, results with numbers |
| Result | practice | a portfolio project that you show and explain to other engineers |

A capstone shows that you can do the full job of a systems engineer: understand a problem, choose a design and defend it, build it in steps, prove that it works, measure it, and explain it clearly. This is exactly what an interviewer for a systems role wants to see. One deep, finished, well-explained project says more than many small, unfinished ones.

## 2. The rules of this phase

1. **Build two capstones.** Not one, not four. Two finished capstones are much better than four half-finished ones.
2. **Choose two different kinds.** One capstone should go deep into the specialization you chose in Phase 22. The other one should cover a different area, so that your portfolio shows range.
3. **Each capstone takes 4–7 weeks** at about 10 hours per week (40–70 hours). The whole phase takes 10–14 weeks. Build them one after the other, not at the same time.
4. **Design before code.** Write a design document first ([Lesson 23.2](02-writing-a-design-document.md)). Coding starts when the document has goals, non-goals, an architecture, failure modes and a test plan.
5. **Work milestone by milestone.** A milestone is done only when all its "Done when" checks pass **in CI** and the work is committed and tagged. Do not start the next milestone before that.
6. **Meet the quality bar** (Section 4). It is not optional, and it is not something you add in the last week.
7. **The core idea must be your own code.** Use crates for the plumbing (`tokio`, `serde`, `clap`, `hyper`, `tracing`). The heart of the project must be yours: in the key-value store, the replication and the storage; in the proxy, the routing, balancing and reload logic. If you use a crate for a core part (for example `openraft`), say so clearly in the README and build something else of equal depth yourself.
8. **Read real systems, do not copy them.** Studying how etcd or Envoy solve a problem is encouraged. Copying their code is not. When an idea comes from a real system, name it in your design document.
9. **Finish by presenting and reviewing.** A capstone is complete after [Lesson 23.9 — Presenting your work](09-presenting-your-work.md) and the review in [Lesson 23.10](10-checkpoint-capstone-review.md).

The five options are described in Lessons 23.3–23.7:

| Option | You build | Builds mostly on | Fits the track | Hardest part |
|---|---|---|---|---|
| 1 — [Distributed key-value store](03-capstone-distributed-key-value-store.md) | a replicated store, like a tiny etcd | Phases 17, 18, 16 | Databases | staying correct when nodes crash |
| 2 — [HTTP reverse proxy](04-capstone-http-reverse-proxy.md) | a proxy and load balancer, like a tiny Envoy | Phases 15, 14, 19 | Networking | tail latency, reload without dropping requests |
| 3 — [Message broker](05-capstone-message-broker.md) | a persistent log with consumer groups, like a tiny Kafka | Phases 17, 15, 18 | Databases or Networking | offsets, replication, durability |
| 4 — [Observability pipeline](06-capstone-observability-pipeline.md) | an agent that collects, transforms and ships logs and metrics, like a tiny Vector | Phases 16, 20, 21 | eBPF or Cloud-native | backpressure, no silent data loss |
| 5 — [Your own idea](07-capstone-your-own-idea.md) | a kernel, a query engine, an operator, … | your track | any | choosing a scope you can finish |

> 💡 **Tip:** Read all five option lessons quickly before you choose. Then choose with two questions: "Which one continues my specialization?" and "Which one would I most like to explain in an interview?"

## 3. The life of a capstone

Every capstone goes through the same stages. The lessons of this phase follow them:

```text
 week 0          weeks 1-5 (about)                 week 6          week 7
+---------+    +----+----+----+----+----+         +-----------+   +------------+
| choose  |    | M0 | M1 | M2 | M3 | M4 | ...      | harden:   |   | present:   |
| design  | -> |    |    |    |    |    |   ---->  | tests,    |-> | README,    | -> review
| doc     |    +----+----+----+----+----+         | benchmarks|   | results,   |   (23.10)
| (23.2)  |    each: build -> Done when -> tag     | (23.8)    |   | talk (23.9)|
+---------+                                        +-----------+   +------------+
```

1. **Choose and design** (about 10% of the time). Pick the option, write `docs/design.md`, and get feedback on it.
2. **Build in milestones** (about 55%). M0 is the walking skeleton (Section 5). Each later milestone adds one capability and ends with acceptance tests.
3. **Harden** (about 15%). Fill gaps in testing, run fuzzers and failure tests, benchmark and profile ([Lesson 23.8](08-testing-benchmarking-and-hardening-capstones.md)). Some hardening happens in every milestone; this stage finishes it.
4. **Present** (about 10%). Write the README and `docs/results.md`, record a demo, prepare a short talk ([Lesson 23.9](09-presenting-your-work.md)).
5. **Buffer** (about 10%). Something always takes longer than planned. The buffer is for that, not for new features.

## 4. The quality bar

Every capstone must meet this bar before you call it done. Lesson 23.8 shows how to reach each point; here is the list so that you plan for it from the start.

| Area | Minimum |
|---|---|
| Build hygiene | `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo test` pass in CI on every push |
| Tests | unit tests for the core, at least one property-based or model-based test, end-to-end tests for every milestone, at least one failure test (a crash, a timeout or a lost connection) |
| Fuzzing | one fuzz target for every parser of outside input (network frames, config files, queries) |
| Performance | a repeatable benchmark with the hardware and workload written down, and p50/p99 latency (the median and the 99th percentile, Lesson 19.1) or throughput numbers in `docs/results.md` |
| Observability | structured logs with `tracing`; for services also metrics and a health endpoint |
| Robustness | timeouts on every network call, limits on every queue and buffer, graceful shutdown |
| Security | no `unwrap` on outside input, `cargo deny check` or `cargo audit` in CI, every `unsafe` block has a `// SAFETY:` comment |
| Documentation | `README.md`, `docs/design.md`, decision records in `docs/adr/`, `docs/results.md` |

The **definition of done** for a whole capstone is short: every milestone's "Done when" passes in CI, the quality bar is met, the documents are written, and you have presented the project.

> 📏 **Convention:** Write the quality bar into your README as a checklist and tick items only when CI proves them. A claim without a test or a number is a wish.

## 5. The repository and a walking skeleton

Each capstone lives in its own folder, `code/p23-<name>`, for example `code/p23-kv` or `code/p23-proxy`. Unlike earlier projects, each capstone gets **its own Git repository**, because you will publish it alone in Lesson 23.9. The examples of this lesson use a practice skeleton, `code/p23-skeleton`, that you can copy for both capstones.

The layout follows the hexagonal idea of [Lesson 08.14](../08-software-design/14-layers-and-hexagonal-architecture.md): logic in a core crate without network or files, and thin binaries around it.

```text
code/p23-skeleton/
├── Cargo.toml            ← the workspace (no [package])
├── README.md             ← written in Lesson 23.9, started now
├── docs/
│   ├── design.md         ← Lesson 23.2
│   ├── adr/              ← decision records
│   ├── log.md            ← your work log
│   └── results.md        ← benchmarks, Lesson 23.8
└── crates/
    ├── skel-core/        ← logic only: easy to unit-test
    └── skel/             ← the binary: arguments, network, files
        └── tests/acceptance.rs
```

A **walking skeleton** is milestone M0 of every capstone. It does almost nothing, but the whole path exists: a workspace, a core crate, a binary that uses it, one acceptance test that runs the real binary, and CI. Later milestones put flesh on bones that are already connected. Create it:

```bash
cd ~/projects/rust
mkdir -p code/p23-skeleton/crates
cd code/p23-skeleton
cargo new --lib crates/skel-core
cargo new crates/skel
mkdir -p crates/skel/tests docs
```

The root `Cargo.toml` (create it):

```toml
[workspace]
resolver = "3"
members = ["crates/*"]

[workspace.package]
version = "0.1.0"
edition = "2024"

[workspace.dependencies]
skel-core = { path = "crates/skel-core" }
```

`crates/skel-core/Cargo.toml` and `crates/skel/Cargo.toml` (replace the generated ones):

```toml
[package]
name = "skel-core"
version.workspace = true
edition.workspace = true

[dependencies]
```

```toml
[package]
name = "skel"
version.workspace = true
edition.workspace = true

[dependencies]
skel-core.workspace = true
```

`crates/skel-core/src/lib.rs`:

```rust
//! The core of the capstone: logic only, no network and no files.

/// The name and version that `skel --version` prints.
pub fn version_line() -> String {
    format!("skel {}", env!("CARGO_PKG_VERSION"))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn version_line_starts_with_the_name() {
        assert!(version_line().starts_with("skel "));
    }
}
```

`crates/skel/src/main.rs`:

```rust
use std::process::ExitCode;

fn main() -> ExitCode {
    let args: Vec<String> = std::env::args().skip(1).collect();
    match args.first().map(String::as_str) {
        Some("--version") => {
            println!("{}", skel_core::version_line());
            ExitCode::SUCCESS
        }
        _ => {
            eprintln!("usage: skel --version");
            ExitCode::from(2)
        }
    }
}
```

**What happens:**

1. The workspace has two members. `skel-core` is a library; `skel` is a binary that depends on it through `skel-core.workspace = true`.
2. `env!("CARGO_PKG_VERSION")` is filled in by Cargo at compile time with the version from `Cargo.toml`, so the binary always prints the real version.
3. `main` returns `ExitCode`: `SUCCESS` (0) for `--version`, and 2 for anything else. Exit code 2 is the usual code for "wrong usage".
4. The usage message goes to standard error with `eprintln!`, so that a script that reads standard output does not see it.

## 6. "Done when" as tests

Every milestone in Lessons 23.3–23.7 ends with **"Done when:"** checks. Turn each check into an acceptance test **before** you start the milestone. The test fails at first (red); the milestone is done when it passes (green). This is test-driven development from [Lesson 09.6](../09-testing-debugging-benchmarking/06-tdd-in-practice.md), applied to a whole milestone.

`crates/skel/tests/acceptance.rs`:

```rust
//! Acceptance tests: one test per "Done when" line of the milestones.
use std::process::Command;

/// Runs the `skel` binary that Cargo built for this test.
fn skel(args: &[&str]) -> (i32, String) {
    let out = Command::new(env!("CARGO_BIN_EXE_skel"))
        .args(args)
        .output()
        .expect("the skel binary should start");
    let code = out.status.code().unwrap_or(-1);
    (code, String::from_utf8_lossy(&out.stdout).into_owned())
}

#[test]
fn m0_prints_its_version() {
    let (code, stdout) = skel(&["--version"]);
    assert_eq!(code, 0);
    assert_eq!(stdout, "skel 0.1.0\n");
}

#[test]
fn m0_unknown_arguments_exit_with_code_2() {
    let (code, _) = skel(&["--frobnicate"]);
    assert_eq!(code, 2);
}

#[test]
#[ignore = "M1 is not started yet"]
fn m1_a_value_survives_a_restart() {
    todo!("write this test at the start of M1");
}
```

**What happens:**

1. `env!("CARGO_BIN_EXE_skel")` is the path of the `skel` binary. Cargo builds the binaries of a package before its integration tests and gives each test this variable, so the test runs the real program.
2. The helper `skel` runs the program with arguments and returns the exit code and the standard output.
3. The test names start with the milestone (`m0_`, `m1_`). The test list then reads like a progress report.
4. `#[ignore = "…"]` keeps a test that belongs to a later milestone in the file without running it. The reason is printed in the test output.

Run everything from `code/p23-skeleton`:

```bash
cargo test
```

```text
     Running tests/acceptance.rs (target/debug/build/skel/64448e282f08d568/out/acceptance-64448e282f08d568)

running 3 tests
test m1_a_value_survives_a_restart ... ignored, M1 is not started yet
test m0_unknown_arguments_exit_with_code_2 ... ok
test m0_prints_its_version ... ok

test result: ok. 2 passed; 0 failed; 1 ignored; 0 measured; 0 filtered out; finished in 0.45s

     Running unittests src/lib.rs (target/debug/build/skel-core/b3fe702f9bd0ce5c/out/skel_core-b3fe702f9bd0ce5c)

running 1 test
test tests::version_line_starts_with_the_name ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(The output is shortened: Cargo also runs the empty test harness of `src/main.rs` and the doc tests.) To see which acceptance tests are still waiting, list only the ignored ones:

```bash
cargo test -q -p skel --test acceptance -- --list --ignored
```

```text
m1_a_value_survives_a_restart: test
```

When you start M1, you remove the `#[ignore]` line, replace `todo!` with the real test, watch it fail, and then build the feature.

> 🧪 **Testing:** Put the acceptance tests in CI from M0 on, with the workflow from [Lesson 09.12](../09-testing-debugging-benchmarking/12-continuous-integration.md). A milestone counts as done only when CI is green, not when it works on your laptop.

**One Git step: a repository per capstone and a tag per milestone.** The course folder is already a Git repository. A repository inside another one is called *embedded*, and the outer repository should ignore it. First tell the course repository to ignore all capstone folders, then start the capstone's own repository and tag M0:

```bash
cd ~/projects/rust
echo 'code/p23-*/' >> .gitignore
git add .gitignore && git commit -m "Ignore capstone repositories (they have their own Git)"
cd code/p23-skeleton
git init
printf '/target\n' > .gitignore
git add .
git commit -m "M0: walking skeleton"
git tag -a m0 -m "M0 done: the walking skeleton builds and its tests pass"
git log --oneline --decorate
```

```text
b172a40 (HEAD -> main, tag: m0) M0: walking skeleton
```

`git tag -a` creates an annotated tag with a message. At the end of every milestone you create the next one (`m1`, `m2`, …). Later anyone can check out exactly the state of each milestone with `git switch --detach m2`.

## 7. Planning time and staying on track

A plan turns "4–7 weeks" into numbers you can check every week. For a capstone of 60 hours (6 weeks × 10 hours):

| Part | Share | Hours |
|---|---|---|
| Choose and design | 10% | 6 |
| Milestones M0–M5 | 55% | 33 |
| Hardening | 15% | 9 |
| Presenting | 10% | 6 |
| Buffer | 10% | 6 |
| **Total** | 100% | **60** |

Give each milestone a time box, for example "M2: 7 hours". Keep a **work log** in `docs/log.md` and write three lines after every session:

```markdown
## 2026-12-03 (2 h)
- Done: M2 leader election works with three nodes on localhost.
- Blocked: the election test is flaky, about 1 in 20 runs fails.
- Next: make the election timeout come from a seeded random generator.
```

The log helps in three ways: you see early when a milestone takes too long, you remember why you made a decision, and in Lesson 23.9 it gives you the story for your blog post (the hardest bug is usually in the log).

**When a milestone takes twice its time box**, stop and choose one of these, in this order:

1. Make the milestone smaller. Move the less important part to the stretch goals.
2. Use part of the buffer.
3. Move a later, optional milestone to the stretch goals.

Never cut the quality bar to save time. A smaller project with tests and numbers is a good capstone; a bigger one without them is not.

**When you are stuck** (there are no answers to open in a capstone):

1. Reread the contract and your design document. Often the problem is a vague requirement.
2. Make the problem smaller: write the smallest failing test that shows it.
3. Reread the lesson where the idea was taught (each option lesson lists them).
4. Look at how a real system solves it (each option lesson lists real systems to study).
5. Ask: a forum, a community chat, an experienced engineer, or an AI assistant as a reviewer. Ask for a hint, not for code.

> ⚠️ **Common mistake:** Starting the second capstone before the first is finished, "to keep things interesting". Two half-finished projects show less than one finished one. Finish, tag, present, then start the next.

## Common mistakes

**1. Adding a capstone repository to the course repository.** If you forget the ignore line and run `git add code` in `~/projects/rust`, Git warns:

```text
warning: adding embedded git repository: code/p23-demo
hint: You've added another git repository inside your current repository.
hint: Clones of the outer repository will not contain the contents of
hint: the embedded repository and will not know how to obtain it.
```

The outer repository then stores only a pointer, not your files. Fix: `git rm --cached code/p23-demo`, add `code/p23-*/` to `.gitignore`, and commit.

**2. Coding before designing.** Two weeks of code, then you discover that the data format cannot support replication. A few hours of design (Lesson 23.2) find such problems while they are still cheap to fix.

**3. A giant first milestone.** "M1: the whole store with replication." It never ends and you never see a green test. Fix: M1 must be small enough to finish in one or two weeks of evenings; the walking skeleton M0 in one evening.

**4. Writing acceptance tests at the end.** Then they test what you built, not what you promised. Fix: write the milestone's tests first, as `#[ignore]` stubs at the start of the project and as real tests at the start of the milestone.

**5. Gold-plating.** Polishing M1 for weeks (a beautiful CLI, colours, many options) while the hard part waits. Fix: the time box. When it is over, move on.

**6. Hiding a dependency.** Using a crate for the core idea and not saying so. Reviewers notice. Fix: rule 7 in Section 2 — name it in the README and build something else of equal depth.

## Try it yourself

1. Create `code/p23-skeleton` exactly as in Sections 5 and 6, and run `cargo test`. You should see two passing acceptance tests and one ignored.
2. Break the contract on purpose: in `skel-core`, change the format to `"skel v{}"`. Run `cargo test` and read the failure:

   ```text
   assertion `left == right` failed
     left: "skel v0.1.0\n"
    right: "skel 0.1.0\n"
   ```

   The acceptance test caught a change that the unit test (`starts_with("skel ")`) let through. Undo the change.
3. Run the quality gate: `cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test`.
4. Make the Git step from Section 6 and check with `git log --oneline --decorate` that the tag `m0` points to your commit.
5. Create `docs/log.md` and write your first entry for today.

## Exercises

### Exercise 1 — Choose your two capstones (easy)

Read the table in Section 2 (and skim Lessons 23.3–23.7). Write `docs/choice.md` in your notes with: the two options, one sentence each on why, which one is in your specialization, and the order in which you will build them.

<details>
<summary>Hint</summary>

Start from your Phase 22 track. Databases → option 1 or 3. Networking → option 2. eBPF or cloud-native → option 4 or your own idea. OS (operating system), embedded or compilers → usually option 5 (your own idea), plus one of 1–4 as the second capstone.

</details>

<details>
<summary>Answer</summary>

An example (yours will differ):

```markdown
# Capstone choice
1. Option 2, HTTP reverse proxy (specialization: networking track).
   Why: it continues Phase 15 and 22, and almost every web service runs behind proxies.
2. Option 1, distributed key-value store (different area: storage and consensus).
   Why: it shows that I understand replication and failure.
Order: proxy first (closer to what I know), then the store.
```

A good choice note names the track connection, the "different area", and the order.

</details>

### Exercise 2 — Write a test: usage goes to standard error (medium)

Add an acceptance test `m0_no_arguments_print_usage_to_stderr` to `crates/skel/tests/acceptance.rs`. It runs `skel` without arguments and checks three things: the exit code is 2, nothing is printed to standard output, and standard error contains `usage: skel`.

<details>
<summary>Hint</summary>

The helper `skel` returns only standard output. Call `Command::new(env!("CARGO_BIN_EXE_skel")).output()` directly: the result has the fields `status`, `stdout` and `stderr` (both `Vec<u8>`).

</details>

<details>
<summary>Answer</summary>

```rust
#[test]
fn m0_no_arguments_print_usage_to_stderr() {
    let out = Command::new(env!("CARGO_BIN_EXE_skel"))
        .output()
        .expect("the skel binary should start");
    assert_eq!(out.status.code(), Some(2));
    assert!(out.stdout.is_empty(), "nothing should go to stdout");
    let stderr = String::from_utf8_lossy(&out.stderr);
    assert!(stderr.contains("usage: skel"), "stderr was: {stderr}");
}
```

Why it works: `status.code()` is `Some(2)` because `main` returned `ExitCode::from(2)`. The usage text was printed with `eprintln!`, so it is in `stderr`, and `stdout` is empty. The messages after the conditions make a failure easy to read.

</details>

### Exercise 3 — A time budget (medium)

You can work 8 hours per week and want to finish your first capstone in 6 weeks. Split the hours with the shares from Section 7 (round to whole hours). Then: your M1 had a time box of 5 hours and took 10. What do you do?

<details>
<summary>Hint</summary>

6 × 8 = 48 hours. Use 10%, 55%, 15%, 10%, 10%. For the second question, look at the ordered list "When a milestone takes twice its time box".

</details>

<details>
<summary>Answer</summary>

48 hours: design 5, milestones 26, hardening 7, presenting 5, buffer 5 (5 + 26 + 7 + 5 + 5 = 48).

M1 took 5 extra hours. First, check whether M1 can be made smaller (move a part to the stretch goals). If the 5 hours are already spent, take them from the buffer, which is now 0, and decide now which optional milestone becomes a stretch goal if anything else is late. Do not take time from hardening or presenting.

</details>

### Exercise 4 — A `--help` flag, test first (hard)

Add `skel --help`: it prints the usage line to **standard output** and exits with 0. Work test-first: write the acceptance test `m0_help_prints_usage_and_succeeds`, see it fail, then change `main.rs`. Keep one constant for the usage text so that both places print the same line.

<details>
<summary>Hint</summary>

The failing test shows `left: 2, right: 0`: today `--help` falls into the `_` branch. Add a `Some("--help")` branch before `_`, and a `const USAGE: &str = "…";` at the top of `main.rs`.

</details>

<details>
<summary>Answer</summary>

The test (in `acceptance.rs`):

```rust
#[test]
fn m0_help_prints_usage_and_succeeds() {
    let (code, stdout) = skel(&["--help"]);
    assert_eq!(code, 0);
    assert!(stdout.starts_with("usage: skel"), "stdout was: {stdout}");
}
```

Before the change it fails with:

```text
thread 'm0_help_prints_usage_and_succeeds' (6457531) panicked at crates/skel/tests/acceptance.rs:47:5:
assertion `left == right` failed
  left: 2
 right: 0
```

The new `crates/skel/src/main.rs`:

```rust
use std::process::ExitCode;

const USAGE: &str = "usage: skel --version | --help";

fn main() -> ExitCode {
    let args: Vec<String> = std::env::args().skip(1).collect();
    match args.first().map(String::as_str) {
        Some("--version") => {
            println!("{}", skel_core::version_line());
            ExitCode::SUCCESS
        }
        Some("--help") => {
            println!("{USAGE}");
            ExitCode::SUCCESS
        }
        _ => {
            eprintln!("{USAGE}");
            ExitCode::from(2)
        }
    }
}
```

Why it works: asking for help is not an error, so the usage goes to standard output with code 0; wrong usage goes to standard error with code 2. All four acceptance tests now pass (`4 passed; 0 failed; 1 ignored`), including the one from Exercise 2, because the usage still starts with `usage: skel`.

</details>

## Quiz

1. How is a capstone different from the earlier projects of the course?

<details>
<summary>Answer</summary>

It is a complete system that you design yourself, it uses many phases at once, it takes weeks instead of sittings, and it must meet a full quality bar (tests at every level, benchmarks, documents, CI). It is a portfolio project that you present to others.

</details>

2. Why does this phase ask for exactly two capstones of different kinds?

<details>
<summary>Answer</summary>

Two finished projects are worth more than many unfinished ones. One goes deep into your specialization, the other shows range in a different area.

</details>

3. What is a walking skeleton, and why is it milestone M0?

<details>
<summary>Answer</summary>

The smallest version of the whole system that builds, runs and is tested end to end, with CI. It connects every part first, so that later milestones only add behaviour to a path that already works and is already tested.

</details>

4. When is a milestone done?

<details>
<summary>Answer</summary>

When all its "Done when" checks pass as acceptance tests in CI, and the work is committed and tagged (for example `m2`). "It works on my laptop" is not enough.

</details>

5. What does `env!("CARGO_BIN_EXE_skel")` give an integration test?

<details>
<summary>Answer</summary>

The path of the `skel` binary that Cargo built for the test run. The test can start the real program with `std::process::Command`.

</details>

6. Your milestone takes twice its time box. Which is the wrong reaction: making the milestone smaller, using the buffer, or skipping the tests to save time?

<details>
<summary>Answer</summary>

Skipping the tests. The quality bar is never cut; you cut scope instead (smaller milestone, buffer, optional milestone to stretch goals).

</details>

7. Why does each capstone get its own Git repository, and what must the course repository do?

<details>
<summary>Answer</summary>

Because you will publish each capstone alone. The course repository must ignore the folders (`code/p23-*/` in `.gitignore`); otherwise `git add` creates an embedded repository that stores only a pointer, not the files.

</details>

## Summary

- A capstone is a complete system that you design, build in milestones, test, measure, document and present. You build **two**, of different kinds, 4–7 weeks each.
- Design comes first (Lesson 23.2). The core idea must be your own code; crates are fine for plumbing, and every borrowed idea is named.
- Every capstone starts with a **walking skeleton** (M0): workspace, core crate, binary, an acceptance test that runs the real binary, and CI.
- Each "Done when" becomes an acceptance test before the milestone starts; `#[ignore = "…"]` keeps future tests visible; a milestone ends with green CI and a Git tag.
- The **quality bar** (tests at every level, fuzzing, benchmarks, observability, robustness, security, documents) is planned from the start and never cut.
- Plan with time boxes and a buffer, keep a work log, and when you are late, cut scope, not quality.

## Sources and further reading

- [The Cargo Book — Workspaces](https://doc.rust-lang.org/cargo/reference/workspaces.html) — every workspace key used in the skeleton.
- [The Cargo Book — Environment variables](https://doc.rust-lang.org/cargo/reference/environment-variables.html) — `CARGO_PKG_VERSION`, `CARGO_BIN_EXE_<name>` and the others Cargo sets.
- [The Rust Programming Language — 11.3 Test Organization](https://doc.rust-lang.org/book/ch11-03-test-organization.html) — unit tests and integration tests in `tests/`.
- [The Rust Reference — Testing attributes](https://doc.rust-lang.org/reference/attributes/testing.html) — `#[test]` and `#[ignore]` with a reason.
- [Pro Git — 2.6 Tagging](https://git-scm.com/book/en/v2/Git-Basics-Tagging) — lightweight and annotated tags.

---

[🏠 Start](../START-HERE.md) · [← Phase 23 — Capstone projects](README.md) · [📋 Phase 23](README.md) · [23.2 Writing a design document →](02-writing-a-design-document.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
