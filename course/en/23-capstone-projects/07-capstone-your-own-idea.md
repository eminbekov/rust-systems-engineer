<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 23.6 Capstone option 4: an observability pipeline](06-capstone-observability-pipeline.md) · [📋 Phase 23](README.md) · [23.8 Testing, benchmarking and hardening your capstone →](08-testing-benchmarking-and-hardening-capstones.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/23-capstone-projects/07-capstone-your-own-idea.md) <!-- nav -->

# Lesson 23.7 — Capstone option 5: your own idea

> **Phase 23 — Capstone projects** · Lesson 7 of 10 · ⏱ about 50 hours (25 sittings)

## What you will learn

- Which ideas make good capstones, with seven questions you can check any idea against.
- A menu of ideas that continue each Phase 22 track, and what each one builds on.
- How to scope your own idea: a one-sentence pitch, a minimum viable product, non-goals, and a *spike* for the riskiest part.
- How to write the contract and milestones yourself, in the same form as options 1–4, with a one-page proposal.
- How to turn "Done when" checks that need outside tools into a small, checked shell script.

## Before you start

- You finished [Lesson 23.1](01-how-capstone-projects-work.md) and [Lesson 23.2](02-writing-a-design-document.md), and you read at least one of the option lessons [23.3](03-capstone-distributed-key-value-store.md)–[23.6](06-capstone-observability-pipeline.md), so you know what a full capstone contract looks like.
- You finished your track in [Phase 22 — Specializations](../22-specializations/README.md); the last lesson of each track (for example [Lesson 22.5](../22-specializations/05-os-track-project-and-next-steps.md) or [Lesson 22.29](../22-specializations/29-cloud-native-track-project-and-next-steps.md)) lists next steps that make good capstones.
- You wrote bash scripts in [Lesson 11.9 — Robust bash scripts](../11-linux-shell-and-containers/09-robust-bash-scripts.md) and checked them with ShellCheck.
- Keep `code/p23-skeleton` from Lesson 23.1; the script in Section 4 runs against it.

## New words

| Word | Meaning |
|---|---|
| **pitch** | One sentence that says what you build and for whom. If you cannot write it, the idea is not clear yet. |
| **MVP** (Minimum Viable Product) | The smallest version that is already useful to someone. In a capstone it is milestone M1. |
| **spike** | A short, time-boxed experiment that tests the riskiest unknown of a project before you commit to it. Its code is usually thrown away. |
| **risk** | Something that could stop the project or make it much bigger than planned, such as "I do not know whether QEMU can emulate this device". |
| **cut list** | The ordered list of features you will drop first if you run late. Written at the start, when it is easy to be honest. |
| **proposal** | A one-page document that describes an idea before the design document: pitch, contract sketch, milestones, risks, non-goals. |
| **appetite** | How much time you are willing to spend on something, decided *before* you design it. The design must fit the appetite, not the other way round. |
| **TCP** (Transmission Control Protocol) | The internet protocol that gives two programs a reliable, ordered stream of bytes. |
| **USB** (Universal Serial Bus) | The common cable and protocol for connecting devices such as development boards to a computer. |
| **SQL** (Structured Query Language) | The language for querying relational databases (Lesson 16.5). |
| **TSDB** (Time-Series Database) | A database for values with timestamps, such as metrics. |
| **PID / UTS namespace** | Linux namespaces (Lesson 13.15) that give a container its own process numbers (PID, process identifier) and its own host name (UTS, UNIX Time-Sharing). |
| **VM** (Virtual Machine) | A whole computer simulated by software; the course's Linux VM is called `dev` (Lesson 11.1). |
| **ADR** (Architecture Decision Record) | A short file that records one important decision: context, decision, consequences (Lesson 23.2). |
| **CPU** (Central Processing Unit) | The processor; "CPU use" is how much of its time a program takes. |
| **UI** (User Interface) | The screens and controls through which a person uses a program. |
| **UTF-8** (Unicode Transformation Format, 8-bit) | The usual encoding of text as bytes; Rust's `String` and `&str` are always valid UTF-8. |
| **DNSSEC** (DNS Security Extensions) | Digital signatures for DNS answers, so that a resolver can check that an answer is genuine. |
| **HTTP** (HyperText Transfer Protocol) | The request-and-response protocol of the web; HTTPS is HTTP over an encrypted connection. |

## 1. What makes a good capstone idea

Options 1–4 are good capstones because they share a few properties. Your own idea should have them too. Check it with seven questions:

1. **Is it a system?** It manages state, concurrency, input/output or hardware, not only screens and forms. A to-do web application is an application; a storage engine under it is a system.
2. **Does it have failure modes you can test?** Crashes, timeouts, bad input, a full disk, a lost device. If nothing can go wrong, there is little to show.
3. **Can it be measured?** Throughput, latency, memory, start-up time, binary size, power. Numbers turn "it works" into "it works this well".
4. **Is the core your code?** The interesting part must not be a crate you call. A database whose query engine is a library is a wrapper.
5. **Is an MVP reachable in about two weeks**, and the whole capstone in 4–7 weeks of ~10 hours? If M1 needs a month, the idea is too big.
6. **Can it be demonstrated in two minutes?** A short demo (Lesson 23.9) needs something visible: a command, a dashboard, a kernel prompt, a blinking light.
7. **Is there a real system to compare with?** Comparing your design with a real one is a large part of what you learn and of what you can tell an interviewer.

An idea that fails one question can often be fixed: a web application becomes a capstone when you build its storage, its job queue or its sync protocol yourself; an idea without measurements gets one when you decide what "fast" means for it.

## 2. A menu of ideas

Each Phase 22 track ends with next steps. These ideas continue them; each one passes the seven questions.

| Idea | Track | Builds on | Compare with |
|---|---|---|---|
| A toy kernel with user programs, system calls and a shell | OS (operating system) | Lessons 22.2–22.5, Phase 13 | xv6, Writing an OS in Rust |
| An OCI (Open Container Initiative) container runtime that Docker can call | OS or cloud native | Lessons 13.15–13.17 | runc, youki, crun |
| An eBPF node agent: run-queue latency, TCP retransmits, OOM (out-of-memory) kills as metrics | eBPF | Lessons 22.6–22.9 | bcc tools, Parca |
| Sensor firmware with Embassy plus a host-side collector over USB or serial | Embedded | Lessons 22.10–22.13 | Embassy examples, Zephyr samples |
| A SQL query engine over your own storage engine | Databases | Lessons 22.14–22.17, Phase 17 | SQLite, DataFusion |
| A small language that compiles to WebAssembly, with a test suite of programs | Compilers | Lessons 22.22–22.25 | Crafting Interpreters (Lox), wasmtime |
| A Kubernetes operator that runs your key-value store (option 1) with backups and upgrades | Cloud native | Lessons 22.26–22.29, Phase 21 | etcd operators, CloudNativePG |
| An authoritative and caching DNS (Domain Name System) server | Networking | Lessons 15.10–15.11 | Hickory DNS, CoreDNS |
| A distributed job scheduler with retries, leases and at-least-once execution | Distributed systems | Lesson 14.15, Phase 18 | Nomad, Temporal |
| A time-series database for metrics with compression and a small query language | Databases | Phase 17, Lesson 21.9 | Prometheus TSDB, VictoriaMetrics |

> 💡 **Tip:** The best own-idea capstones combine your track with a problem you have actually seen: a tool you wished existed, a slow step in a build, a system at work whose inside you never understood. Curiosity carries you through week five.

## 3. Scoping your idea

Most own-idea capstones fail in the same way: they are too big. Scoping makes the idea fit the **appetite** — 4–7 weeks — before you design anything.

**Step 1: the pitch.** One sentence: "`minirt` is an OCI container runtime that Docker can use to start Alpine containers with namespaces, cgroups and seccomp." If the sentence needs "and also", the idea is two ideas.

**Step 2: the MVP.** What is the smallest version that someone could use? For `minirt`: "`minirt run` starts a shell in a bundle made from `docker export alpine`, in new PID, mount and UTS namespaces." That is M1, and it must be reachable in about two weeks.

**Step 3: non-goals and the cut list.** Write at least three non-goals ("no networking between containers", "no rootless mode", "no Windows") and a cut list in order ("if late, drop: 1. seccomp profiles, 2. lifecycle hooks, 3. the comparison with youki"). Writing them now, while you are optimistic, is easier than deciding them in week six.

**Step 4: the spike.** Find the riskiest unknown — the thing that, if it does not work, makes the plan collapse. Spend a fixed time box (one or two evenings) on a throwaway experiment that answers only that question. For `minirt`, it might be "can my program call `clone` with new namespaces from Rust and `pivot_root` into a directory, in the VM `dev`?" A spike that fails is a success: it saved you weeks.

**Step 5: the proposal.** Put it all on one page in `docs/proposal.md`:

```markdown
# Capstone proposal: minirt

Pitch: minirt is an OCI container runtime that Docker can use to start
Alpine containers with namespaces, cgroups v2 and seccomp.

Why it matters: Docker and Kubernetes do not start containers themselves;
they call a runtime. Building one shows Linux internals end to end.

Builds on: Lessons 13.15-13.17 (namespaces, cgroups, mini runtime), 10.12 (unsafe review).

Contract sketch:
  minirt create <id> --bundle <dir>   minirt start <id>   minirt state <id>
  minirt kill <id> [SIGNAL]           minirt delete <id>  minirt run <id>
  State JSON and status values follow the OCI runtime specification.

Core that is mine: namespace setup, mounts, pivot_root, cgroups, seccomp,
the create/start split. Crates for plumbing: oci-spec, nix, clap, serde_json.

Milestones:
  M0 skeleton + spike result     Done when: CI green, spike notes in docs/
  M1 run a bundle (MVP)           Done when: `ps` inside shows PID 1
  M2 create/start/state/delete    Done when: state goes created -> running -> stopped
  M3 cgroups and seccomp          Done when: a fork bomb stops at the PID limit
  M4 Docker uses minirt           Done when: docker run --runtime minirt alpine echo hi
  M5 harden and measure           Done when: start time vs runc in docs/results.md

Biggest risk: the order of setup steps after clone. Spike: 2 evenings.
Non-goals: no networking, no rootless mode, no checkpoint/restore.
Cut list: 1. seccomp, 2. lifecycle hooks, 3. youki comparison.
Compare with: runc, youki, crun.
```

This is an example of *scoping*, not a design: the design document (Lesson 23.2) comes next and answers how each milestone works and fails.

## The project

### What you will build

Your own capstone in `code/p23-<name>`, with the same structure as options 1–4: a workspace with a pure core crate, thin binaries, tests, `docs/proposal.md`, `docs/design.md`, `docs/adr/` and later `docs/results.md`.

### Contract

You write the contract yourself. It must contain the same parts as the contracts of options 1–4:

| Part | What to write |
|---|---|
| Interface | the exact command line, API (Application Programming Interface) or protocol, with examples of input and output |
| Rules | numbered rules for behaviour, errors, limits and guarantees, each one testable |
| Non-goals | at least three, plus the cut list |
| Executable contract | a trait, a reference model or a test vector, as in options 1–4, written in M0 |
| Milestones | M0–M5 (or M6), each with "Done when" checks that a stranger could run |
| Quality bar | how each row of the Lesson 23.1 quality bar applies to your idea |

> 📏 **Convention:** Write "Done when" checks as commands with expected output, exactly like the option lessons do. "Works well" is not a check; `minirt state demo` printing `"status": "running"` is.

### Step-by-step plan

These milestones are about the *process*, because the content is yours.

#### M0 — Proposal, spike and skeleton

**Goal.** `docs/proposal.md` (Section 3), the spike for the biggest risk with its result written down, and the walking skeleton from Lesson 23.1 with CI.

**Done when:** the proposal passes the seven questions of Section 1 (write the answer to each in one line under the proposal); the spike notes say what you tried, what happened and whether the plan changes; `cargo test` is green in CI.

#### M1 — The MVP

**Goal.** The smallest useful version, end to end, with acceptance tests.

**Done when:** the M1 checks of your proposal pass in CI, and you can show the MVP in a two-minute demo to someone (or record it for yourself).

#### M2–M4 — Your capabilities

**Goal.** One capability per milestone, in the order of your proposal. After each milestone, look at the cut list and the time spent, and decide honestly whether to continue as planned.

<details>
<summary>Hint</summary>

If a milestone takes twice its time box, use the rules from Lesson 23.1 (make it smaller, use the buffer, move a later milestone to the stretch goals). For your own idea there is one more rule: never change the *pitch* in the middle of the project. If the idea itself must change, write an ADR that explains why.

</details>

**Done when:** each milestone's checks pass in CI and the milestone is tagged.

#### M5 — Harden and measure

**Goal.** The quality bar: failure tests, fuzzing of every parser, benchmarks with numbers, observability where it applies (Lesson 23.8).

**Done when:** `docs/results.md` contains numbers and a comparison with the real system you named; the quality-bar table in your README has every row ticked or explained.

### Quality bar

The rows of the Lesson 23.1 quality bar apply to every idea, but they look different per track:

| Track | Tests that matter most | What to measure |
|---|---|---|
| OS | tests in QEMU with scripted serial input; a crash in a user program must not crash the kernel | boot time, context-switch cost |
| eBPF | the agent survives a missing kernel feature; per-CPU data summed correctly | overhead in CPU per event |
| Embedded | host-side tests of all logic (`no_std` core compiled for the Mac); simulated sensors | power or latency per reading |
| Databases | model-based tests against SQLite or a simple model; fuzzing of the SQL parser | queries per second, plan quality |
| Compilers | a suite of test programs with expected output (snapshot tests); fuzzing of the parser | compile time, run time of benchmarks |
| Cloud native | reconcile tests with a fake API server; tests in `kind` | time to converge after a change |

### Stretch goals

Your proposal's cut list, reversed: what you dropped first is what you add first if time is left. Add one stretch goal that connects your idea to another capstone, for example "the operator manages the key-value store of option 1" or "the eBPF agent sends its metrics through the pipeline of option 4".

## 4. Turning outside checks into a script

Some "Done when" checks need tools outside Cargo: QEMU, Docker, `kind`, a serial port. A Rust acceptance test can start them, but often a small shell script is clearer. Here is a pattern that works for any capstone. Try it on the skeleton: in `code/p23-skeleton`, create `scripts/done-when.sh`:

```bash
#!/usr/bin/env bash
# Done-when checks for one milestone. Run it from the workspace root.
set -euo pipefail

bin=target/debug/skel
cargo build --quiet

failures=0

# check NAME EXPECTED COMMAND...: run COMMAND, compare stdout+stderr with EXPECTED.
check() {
    local name=$1 expected=$2
    shift 2
    local actual
    actual=$("$@" 2>&1) || true
    if [[ "$actual" == "$expected" ]]; then
        echo "PASS $name"
    else
        echo "FAIL $name"
        echo "  expected: $expected"
        echo "  actual:   $actual"
        failures=$((failures + 1))
    fi
}

check "prints its version" "skel 0.1.0" "$bin" --version
check "explains its usage" "usage: skel --version" "$bin" --frobnicate

if ((failures > 0)); then
    echo "$failures check(s) failed"
    exit 1
fi
echo "all checks passed"
```

**What happens:**

1. `set -euo pipefail` (Lesson 11.9) stops the script on the first unexpected error, on an unset variable, and on a failure inside a pipe.
2. `check` takes a name, the expected output and the command. `shift 2` removes the first two arguments, so `"$@"` is exactly the command with its arguments, correctly quoted.
3. `actual=$("$@" 2>&1) || true` captures standard output and standard error together; `|| true` keeps `set -e` from stopping the script when the command exits with an error code on purpose (as `--frobnicate` does).
4. Every check prints `PASS` or `FAIL`, and the script continues, so you see all failures at once. The exit code at the end is 1 if any check failed, so CI can use the script as one step.

```bash
chmod +x scripts/done-when.sh
./scripts/done-when.sh
```

```text
PASS prints its version
PASS explains its usage
all checks passed
```

(If you did Exercise 4 of Lesson 23.1, your usage line is `usage: skel --version | --help`; use that text as the expected value of the second check.)

Check the script itself with ShellCheck (no output means no findings):

```bash
docker run --rm -v "$PWD":/mnt koalaman/shellcheck:stable scripts/done-when.sh
```

And see a failure: change the expected version to `skel 0.2.0` and run it again:

```text
FAIL prints its version
  expected: skel 0.2.0
  actual:   skel 0.1.0
PASS explains its usage
1 check(s) failed
```

Change it back. In a real capstone, the commands in the checks are your "Done when" lines: `minirt state demo`, `qemu-system-aarch64 … < script.txt`, `kubectl get pods`, and the script becomes a CI job.

> 🧪 **Testing:** A script check is an acceptance test like any other: write it before the milestone, see it fail, then make it pass. Keep the slow ones (QEMU, `kind`) in a separate CI job so that the fast tests still give feedback in a minute.

## Common mistakes

**1. An idea that is really three ideas.** "A database with its own query language, replication and a web UI." Fix: the pitch test — one sentence, no "and also"; the rest goes into non-goals or a later project.

**2. Skipping the spike.** Two weeks into the project you learn that the hardware cannot do what you need. Fix: the spike in M0, time-boxed, on the riskiest unknown only.

**3. A core that is a library call.** The query engine is a crate, the kernel is a template, the operator is generated code with three lines changed. Fix: question 4 of Section 1; name what is yours in the proposal.

**4. Milestones without checks.** "M3: improve performance." Fix: "M3 Done when: `bench` reports at least 50,000 queries per second on the example data set, measured as described in `docs/results.md`."

**5. Unquoted `$@` in check scripts.** Arguments with spaces are split, and the check runs a different command. ShellCheck finds it:

```text
In scripts/bad.sh line 15:
    actual=$($@ 2>&1) || true
             ^-- SC2068 (error): Double quote array expansions to avoid re-splitting elements.
```

Fix: always `"$@"`.

## Exercises

### Exercise 1 — Apply the seven questions (easy)

Check three ideas against the seven questions of Section 1 and say which ones are good capstones, and how to fix the others: (a) a to-do list web application with user accounts; (b) a faster `grep` that uses SIMD (Single Instruction, Multiple Data: one instruction works on many values at once, Lesson 19.7) and memory-mapped files; (c) "an operating system".

<details>
<summary>Hint</summary>

Look especially at questions 1 (system?), 3 (measurable?) and 5 (MVP in two weeks?).

</details>

<details>
<summary>Answer</summary>

(a) Not as stated: it is an application, and its interesting parts (storage, sessions) would be libraries. Fix: build the part under it yourself, for example a sync protocol for offline clients with conflict resolution, and measure it. (b) Yes: it is a system tool, it has failure modes (huge files, invalid UTF-8, files changing while read), it is very measurable (compare with existing tools on the same files), and an MVP (fixed string search with `mmap`) fits in two weeks. (c) Not as stated: it is far too big (question 5). Fix: "a toy kernel for QEMU arm64 that runs two user programs with system calls and a serial shell" — the idea from the menu, with an MVP of "boot and print to the serial console".

</details>

### Exercise 2 — Pitch, MVP and non-goals (medium)

Choose one idea from the menu in Section 2 (or your own). Write its pitch in one sentence, its MVP in one sentence, three non-goals, and a cut list of three items.

<details>
<summary>Hint</summary>

The MVP must be something a person can *use* or *see*, reachable in two weeks. Non-goals are things someone could reasonably expect from the pitch.

</details>

<details>
<summary>Answer</summary>

An example for the DNS server:

- Pitch: "`minidns` is an authoritative and caching DNS server for a home network, configured from one file."
- MVP: "`dig @127.0.0.1 -p 5353 router.home A` returns the address from the zone file."
- Non-goals: no DNSSEC signing; no zone transfers; no DNS over HTTPS.
- Cut list: 1. negative caching, 2. TCP fallback for large answers, 3. the comparison with Hickory DNS.

</details>

### Exercise 3 — Write a test: check the exit code (medium)

`check` compares only output. Add a function `check_exit NAME CODE COMMAND...` that runs the command, ignores its output, and compares its **exit code** with `CODE`. Use it to check that `skel --frobnicate` exits with 2. Make sure the script still passes ShellCheck, and that a wrong expected code is reported.

<details>
<summary>Hint</summary>

With `set -e`, a failing command stops the script, so catch the code in the same line: `"$@" >/dev/null 2>&1 || actual=$?`. Start with `actual=0` for commands that succeed.

</details>

<details>
<summary>Answer</summary>

Add the function after `check`:

```bash
# check_exit NAME CODE COMMAND...: run COMMAND, compare its exit code with CODE.
check_exit() {
    local name=$1 expected=$2
    shift 2
    local actual=0
    "$@" >/dev/null 2>&1 || actual=$?
    if [[ "$actual" == "$expected" ]]; then
        echo "PASS $name"
    else
        echo "FAIL $name (exit code $actual, expected $expected)"
        failures=$((failures + 1))
    fi
}
```

and the check after the other two:

```bash
check_exit "wrong usage exits with 2" 2 "$bin" --frobnicate
```

```text
PASS prints its version
PASS explains its usage
PASS wrong usage exits with 2
all checks passed
```

With a wrong expectation (`1` instead of `2`), the script reports it and exits with 1:

```text
FAIL wrong usage exits with 2 (exit code 2, expected 1)
1 check(s) failed
```

Why it works: `|| actual=$?` runs only when the command fails, and `$?` is then its exit code; `set -e` does not stop the script, because the failure is part of an `||` list. ShellCheck reports nothing for the new function.

</details>

### Exercise 4 — Scope a big idea (hard)

A friend wants to build "a distributed SQL database" as a capstone in 6 weeks at 10 hours per week. Turn it into a proposal that fits: pitch, MVP, milestones M0–M5 with one "Done when" each, the biggest risk and its spike, and non-goals.

<details>
<summary>Hint</summary>

Pick one hard, interesting part and make everything else small or reused. The course already gave your friend a storage engine (Project 17.14), a Raft store (Project 18.14 or option 1) and a SQL parser (Lesson 22.15).

</details>

<details>
<summary>Answer</summary>

One good scoping (there are others):

- Pitch: "`rsql` runs single-table SQL queries over a key-value store that is replicated with Raft."
- MVP (M1): `SELECT` and `INSERT` on one table, one node, data stored in the `p17-kv` engine.
- M0: skeleton, SQL parser reused from Lesson 22.15 with its tests; spike on encoding rows and a primary-key index as keys. Done when: `rsql check "SELECT * FROM t"` prints the parsed plan.
- M1 Done when: `INSERT` then `SELECT` returns the rows after a restart.
- M2 secondary indexes. Done when: a query with `WHERE email = …` reads one index key instead of scanning (counted in a test).
- M3 replicated storage (the Raft store of option 1 or Project 18.14). Done when: killing the leader during inserts loses no acknowledged row.
- M4 single-key transactions with compare-and-swap. Done when: two concurrent `UPDATE … SET n = n + 1` never lose an update (property test).
- M5 measure and harden. Done when: `docs/results.md` compares point queries with SQLite on the same data, and the SQL parser has run under the fuzzer for one hour.
- Biggest risk: the row and index encoding (order-preserving keys). Spike: two evenings.
- Non-goals: no joins, no multi-key transactions, no SQL standard compliance, no query optimizer beyond "use the index if there is one".

The scope works because only two parts are new and hard (encoding with indexes, and transactions on top of Raft); the rest reuses earlier projects, which the proposal states openly.

</details>

## Quiz

1. Why is "a to-do web application" not a capstone as it stands, and how could it become one?

<details>
<summary>Answer</summary>

It is an application whose interesting parts would be libraries, with few testable failure modes. It becomes a capstone if you build a system part yourself — storage, a sync protocol, a job queue — and measure it.

</details>

2. What is a spike, and when is a failed spike a good result?

<details>
<summary>Answer</summary>

A short, time-boxed, throwaway experiment that tests the riskiest unknown. A failed spike is a good result because it shows early, in an evening or two, that the plan must change — instead of after weeks of work.

</details>

3. Why write the cut list at the start of the project?

<details>
<summary>Answer</summary>

At the start you can decide calmly what matters least. Late in the project, under time pressure, it is much harder to drop things honestly, and people cut tests instead.

</details>

4. What does "appetite" mean, and how does it change design?

<details>
<summary>Answer</summary>

The time you are willing to spend, decided before the design (here 4–7 weeks). The design must fit the appetite: you choose a smaller scope instead of letting the design decide how long the project takes.

</details>

5. In `check`, why is the command written as `"$@"` and why is `|| true` needed?

<details>
<summary>Answer</summary>

`"$@"` passes every argument exactly as given, including arguments with spaces. `|| true` stops `set -e` from ending the script when the checked command exits with a non-zero code on purpose.

</details>

6. Which parts must your own contract contain?

<details>
<summary>Answer</summary>

The interface with examples, numbered testable rules, non-goals and a cut list, an executable contract (trait, reference model or test vector), milestones with "Done when" checks, and how the quality bar applies.

</details>

## Summary

- A good capstone idea is a system, has testable failure modes, can be measured, has a core that is your code, has an MVP within about two weeks, can be demonstrated, and has a real system to compare with.
- The idea menu continues each Phase 22 track: a toy kernel, an OCI runtime, an eBPF agent, sensor firmware, a query engine, a WebAssembly compiler, an operator, a DNS server, a job scheduler, a time-series database.
- Scope with a pitch, an MVP, non-goals, a cut list and a spike for the biggest risk, and put it on one page in `docs/proposal.md`.
- Write the contract in the same form as options 1–4, including an executable part written in M0.
- "Done when" checks that need outside tools fit a small `set -euo pipefail` script with `check` functions, checked by ShellCheck and run in CI.

## Sources and further reading

- [Shape Up — Set Boundaries](https://basecamp.com/shapeup/1.2-chapter-03) — the idea of an *appetite*: fix the time, then shape the scope to fit it.
- [Extreme Programming — Create spike solutions](http://www.extremeprogramming.org/rules/spike.html) — the original description of a spike.
- [ShellCheck](https://www.shellcheck.net/) and its [repository](https://github.com/koalaman/shellcheck) — every warning code (such as SC2068) has a page that explains it.
- [OCI runtime specification](https://github.com/opencontainers/runtime-spec) — the contract behind the `minirt` example proposal.
- [Writing an OS in Rust](https://os.phil-opp.com/) — the classic starting point for the kernel idea in the menu.

---

[🏠 Start](../START-HERE.md) · [← 23.6 Capstone option 4: an observability pipeline](06-capstone-observability-pipeline.md) · [📋 Phase 23](README.md) · [23.8 Testing, benchmarking and hardening your capstone →](08-testing-benchmarking-and-hardening-capstones.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
