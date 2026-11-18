<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 24 — Career and open source](README.md) · [📋 Phase 24](README.md) · [24.2 Building your portfolio →](02-building-your-portfolio.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/24-career-and-open-source/01-roles-for-systems-engineers.md) <!-- nav -->

# Lesson 24.1 — Roles for systems engineers

> **Phase 24 — Career and open source** · Lesson 1 of 9 · ⏱ about 120 minutes

## What you will learn

- What people mean by "systems engineer", and why the job title tells you less than the daily work.
- Eight common roles (systems software, infrastructure, SRE, database, networking, embedded, kernel and
  platform engineer): what each one does every day and which phases of this course prepare you for it.
- How to read a job description honestly: required lines, nice-to-have lines, and what you can show today.
- How to rate your own skills and compute a "role fit" with a small Rust program and its tests.

## Before you start

- You finished [Phase 23 — Capstone projects](../23-capstone-projects/README.md), or you are working on it.
  This phase runs alongside the other phases, so you can also start it earlier and come back.
- You chose a specialization in [Lesson 22.1 — Choosing a specialization](../22-specializations/01-choosing-a-specialization.md).
  The idea of a *T-shaped engineer* from that lesson is used here.
- You keep a learning journal in `~/projects/rust/notes/` ([Lesson 00.8](../00-getting-started/08-study-routine-notes-and-checkpoint.md)).

## New words

| Word | Meaning |
|---|---|
| **role** | The kind of work a person does in a team, for example "keeps the servers running" or "writes the storage engine". |
| **job title** | The name written on a job offer, for example "Infrastructure Engineer". The same title can mean different work in different companies. |
| **job description** | The text of a job advertisement: what the team does, what you will do, and what they expect you to know. |
| **SRE** (Site Reliability Engineering / Site Reliability Engineer) | A way to run production systems with software engineering: measure reliability, automate manual work, handle incidents. The person who does it is also called an SRE. |
| **on-call** | A duty period when you must answer alerts about production problems, also at night or at the weekend (Lesson 21.12). |
| **platform** (internal) | The set of shared tools and services that a company builds for its own developers: deployment, clusters, logs, metrics. |
| **firmware** | The program that runs on a small device, often stored in the chip itself (Lesson 22.10). |
| **must-have / nice-to-have** | Lines in a job description that the team really needs / lines that would be a bonus. |
| **evidence** | Something another person can look at that proves a skill: a repository, a merged pull request, a write-up, a talk. |
| **role fit** | In this lesson: how well your current skills cover the skills a role needs, as a percentage. |
| **OS** (Operating System) | The program that manages the hardware and runs other programs, for example Linux (Phase 13). |
| **CI/CD** (Continuous Integration / Continuous Delivery) | Automatic building, testing and delivery of every change (Lessons 09.12 and 21.7). |
| **SLO** (Service Level Objective) | A reliability target, for example "99.9% of requests succeed in a month" (Lesson 21.11). |
| **API** (Application Programming Interface) | The functions and types that a program or library offers to other code. |
| **TCP/IP** (Transmission Control Protocol / Internet Protocol) | The two main protocols of the internet: IP moves packets between machines, TCP turns them into a reliable stream (Phase 15). |
| **TLS** (Transport Layer Security) | The protocol that encrypts network connections, for example for web addresses that start with `https://` (Lesson 10.11). |
| **I/O** (Input/Output) | Reading and writing data: files, network sockets, devices. |
| **eBPF** (extended Berkeley Packet Filter) | Small, checked programs that run inside the Linux kernel (Lesson 22.6). |

## 1. What "systems engineer" means

A **systems engineer** builds and runs the software that other software stands on: operating systems,
runtimes, databases, networks, proxies, clusters and the tools around them. Users rarely see this
software, but every website, phone app and payment depends on it.

This is a broad area. Two people with the same title "Systems Engineer" can have very different days.
One writes a storage engine in Rust and thinks about `fsync` all week. The other keeps two thousand Linux
servers healthy and writes automation. Both are systems work, but the skills are different.

So the first rule of this phase is simple: **read the daily work, not the title.** A job description
usually has a section like "What you will do". That section tells you the role. The title only tells
you how one company names it.

A second rule: titles change faster than skills. Ten years ago some of today's roles had other names,
and in ten years they will have new names again. The skills you learned in this course (how memory,
processes, networks, storage and distributed systems work) stay useful under any title.

## 2. Eight roles

The table below describes eight common roles. Real teams mix them; a small company may have one person
who does three of them.

| Role | What you do on a normal day | Core skills | Course phases |
|---|---|---|---|
| **Systems software engineer** | Write low-level programs used by other teams: runtimes, agents, command-line tools, libraries, file sync engines. | Rust, memory, concurrency, OS interfaces, API design, performance. | 03–06, 12–14, 19 |
| **Infrastructure engineer** | Build and run the base layer: servers, networks, storage, clusters; automate how they are created and repaired. | Linux, networking, containers, automation, infrastructure as code. | 11, 15, 21 |
| **SRE** | Keep production reliable: define SLOs, build monitoring, handle incidents, remove repeated manual work. | Linux debugging, observability, automation, incident handling, networking. | 11, 16, 20, 21 |
| **Database / storage engineer** | Build storage engines, query engines, replication and backup. | Data structures on disk, durability, transactions, concurrency, consensus. | 07, 17, 18, 22 (databases track) |
| **Networking engineer (software)** | Build the data path: proxies, load balancers, protocol libraries, packet processing. | TCP/IP, TLS, async I/O, performance, Linux networking. | 13–15, 19, 22 (networking or eBPF track) |
| **Embedded / firmware engineer** | Program microcontrollers in devices; work with little memory and often no OS. | `no_std` Rust, hardware registers, interrupts, timing, power. | 12, 22 (embedded track) |
| **Kernel engineer** | Work inside the OS: drivers, schedulers, file systems, memory management. | Low-level code, `unsafe`, hardware, concurrency, reading huge codebases. | 12, 13, 22 (OS track) |
| **Platform engineer** | Build the internal platform for developers: deployment pipelines, cluster tooling, templates, self-service tools. | Containers, Kubernetes, CI/CD, automation, observability, API design. | 09, 11, 16, 21, 22 (cloud-native track) |

**Where Rust appears.** Rust is used in all of these areas, but not equally. You find it most in new
systems software, databases, proxies and networking libraries, command-line tools, embedded firmware, and
more and more in the Linux kernel (drivers written in Rust). Infrastructure, SRE and platform work often
uses several languages; Rust appears there in agents, operators and tools. For you this means: Rust skill
is a strong plus everywhere, and the *systems knowledge* from this course is needed everywhere.

The roles form layers. Lower layers are closer to the hardware:

```text
   users and product teams
 ┌──────────────────────────────────────────────────────────┐
 │  platform engineer · SRE      (how software is run)      │
 ├──────────────────────────────────────────────────────────┤
 │  infrastructure engineer      (servers, clusters, nets)  │
 ├──────────────────────────────────────────────────────────┤
 │  systems software · database · networking engineer       │
 │                               (the programs underneath)  │
 ├──────────────────────────────────────────────────────────┤
 │  kernel engineer · embedded engineer   (next to metal)   │
 └──────────────────────────────────────────────────────────┘
   hardware
```

An engineer who knows one layer well and understands the layers next to it is very valuable. When a
request is slow, the SRE who can read a flame graph (Lesson 19.3) and the database engineer who knows
how the page cache works (Lesson 17.9) find the problem much faster than people who only know their
own layer.

> 💡 **Tip:** You do not need to choose a role for life. Many engineers move between neighbouring roles:
> infrastructure to SRE, systems software to database, networking to kernel. Choose the role that fits
> your **next** two years.

## 3. Reading a job description

A job description is written by a busy team, often from an old template. It usually has three kinds of
lines:

1. **What you will do.** This is the role. Read it first.
2. **Must-have lines** ("You have", "Requirements"). Teams often list more than they really need.
3. **Nice-to-have lines** ("Bonus", "Nice to have"). Nobody has all of them.

Here is an example. It is **invented for this lesson**; it does not describe any real company.

```text
Infrastructure Engineer (invented example)

We run a fleet of Linux servers and a Kubernetes platform for our product teams.

You will:
- Automate how we create, update and repair servers.
- Build internal tools in Rust and maintain our deployment pipelines.
- Take part in an on-call rotation with the rest of the team.

You have:
- Solid Linux knowledge: processes, file systems, networking basics.
- Experience with containers and Kubernetes.
- Programming experience in at least one language; Rust is a plus.
- You can explain technical decisions in writing.

Nice to have:
- Experience with Prometheus or another metrics system.
- Contributions to open-source projects.
```

**The marking method.** Copy every "You have" and "Nice to have" line into your journal and mark it:

- **H** (have): you can show evidence today. Write the evidence next to it.
- **L** (learning): you are working on it, and you know which lesson or project closes it.
- **G** (gap): not covered yet. Write one concrete next step.

A learner who finished this course could mark the ad like this:

```text
H  Solid Linux knowledge        -> Phase 11 and 13 projects: my shell (13.8), mini container runtime (13.17)
H  Containers and Kubernetes    -> Project 21.13: my service on a kind cluster
H  Programming, Rust a plus     -> code/ repository, capstone project
L  Explain decisions in writing -> design document of my capstone (23.2); I will publish it
L  Prometheus                   -> Lesson 21.9 done; no public example yet
G  Open-source contributions    -> Lesson 24.4: first contribution this month
```

Two honest observations come out of this. First, the "on-call" line in "You will" is important: ask in
the interview how often it happens and how it is supported. Second, if you have most of the
must-have lines, **apply**. Job descriptions are wish lists; teams often hire people who match much of the list
and can learn the rest.

> ⚠️ **Common mistake:** Waiting until you match 100% of a job description. Almost nobody does. Mark the
> lines, close the gaps you can close in a few weeks, and apply when most must-have lines are "H" or "L".

## 4. Rate yourself: a role-fit program

Feelings about skills are unreliable. Writing numbers down, even rough ones, makes them easier to compare
and to update every few months. We use a scale from 0 to 3:

| Score | Meaning |
|---|---|
| 0 | I have never used it. |
| 1 | I followed a lesson and can do it with notes. |
| 2 | I built something with it alone, and it has tests. |
| 3 | I can explain it to someone else and debug it when it breaks. |

The program below compares your ratings with the areas each role needs most and prints a fit
percentage. It is not science; it is a mirror that helps you think.

All code of this phase lives in one practice package, `code/p24-career`. Each lesson adds one small
program to `src/bin/`, and you run it with `cargo run --bin <name>`. Create the package and remove the
default `main.rs`:

```bash
cd ~/projects/rust
cargo new code/p24-career
cd code/p24-career
rm src/main.rs
mkdir src/bin
```

Create `src/bin/roles.rs`:

```rust
// Rate yourself from 0 to 3 in each skill area and see which roles fit you today.

/// A role and the skill areas it needs most.
type Role = (&'static str, &'static [&'static str]);

#[rustfmt::skip] // keep one role per line
const ROLES: &[Role] = &[
    ("systems software", &["rust", "linux", "concurrency", "performance"]),
    ("infrastructure", &["linux", "networking", "containers", "automation"]),
    ("SRE", &["linux", "observability", "automation", "networking"]),
    ("database", &["rust", "storage", "concurrency", "distributed"]),
    ("networking", &["rust", "networking", "linux", "performance"]),
    ("embedded", &["rust", "hardware", "low-level"]),
    ("kernel", &["low-level", "linux", "hardware", "concurrency"]),
    ("platform", &["containers", "automation", "observability", "rust"]),
];

/// Returns how well your ratings cover the areas a role needs, from 0 to 100 percent.
/// An area you did not rate counts as 0; a score above 3 counts as 3.
fn fit(needs: &[&str], ratings: &[(&str, u32)]) -> u32 {
    let mut got = 0;
    for need in needs {
        for (area, score) in ratings {
            if area == need {
                got += (*score).min(3);
            }
        }
    }
    let max = 3 * needs.len() as u32;
    got * 100 / max
}

fn main() {
    let my_ratings = [
        ("rust", 2),
        ("linux", 2),
        ("networking", 1),
        ("storage", 1),
        ("concurrency", 2),
        ("performance", 1),
        ("containers", 2),
        ("automation", 1),
        ("observability", 1),
        ("distributed", 1),
        ("hardware", 0),
        ("low-level", 1),
    ];

    let mut results: Vec<(u32, &str)> = ROLES
        .iter()
        .map(|(name, needs)| (fit(needs, &my_ratings), *name))
        .collect();
    // Best fit first; equal fits in alphabetical order.
    results.sort_by(|a, b| b.0.cmp(&a.0).then(a.1.cmp(b.1)));

    for (percent, name) in results {
        let bar = "#".repeat((percent / 10) as usize);
        println!("{name:<17} {percent:>3}% {bar}");
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn full_marks_give_100_percent() {
        assert_eq!(fit(&["a", "b"], &[("a", 3), ("b", 3)]), 100);
    }

    #[test]
    fn missing_areas_count_as_zero() {
        assert_eq!(fit(&["a", "b"], &[("a", 3)]), 50);
        assert_eq!(fit(&["a", "b"], &[]), 0);
    }

    #[test]
    fn scores_above_three_are_capped() {
        assert_eq!(fit(&["a", "b"], &[("a", 9), ("b", 0)]), 50);
    }
}
```

**What happens:**

1. `type Role = (...)` gives a short name to a tuple: the role's name and a slice of the areas it needs.
   Everything is `&'static str` because the text is written in the program itself (Lesson 05.3).
2. `#[rustfmt::skip]` tells `cargo fmt` not to reformat the next item. Without it, `rustfmt` would split
   every role over four lines; one role per line is easier to read here. Use this attribute rarely.
3. `fit` adds up your scores for the areas the role needs. `(*score).min(3)` caps a score at 3, so a
   typo like `9` cannot give more than full marks. An area you did not rate adds nothing.
4. `got * 100 / max` turns the sum into a percentage. Integer division rounds down: 7 of 12 is 58%.
5. In `main`, `map` builds a `(percent, name)` pair for each role, and `sort_by` sorts by percent,
   largest first (`b.0.cmp(&a.0)`), then by name (`then(...)`). Uppercase letters sort before lowercase,
   which is why `SRE` comes before `kernel`.
6. `{name:<17}` pads the name to 17 characters on the left side, `{percent:>3}` right-aligns the
   number, and the bar has one `#` per 10%.

Run it:

```bash
cargo run --bin roles
```

```text
systems software   58% #####
database           50% #####
infrastructure     50% #####
networking         50% #####
platform           50% #####
SRE                41% ####
kernel             41% ####
embedded           33% ###
```

Then run the tests:

```bash
cargo test --bin roles
```

```text
running 3 tests
test tests::full_marks_give_100_percent ... ok
test tests::missing_areas_count_as_zero ... ok
test tests::scores_above_three_are_capped ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Testing:** The tests use tiny invented roles (`["a", "b"]`) instead of the real table. Then each
> test checks one rule (full marks, missing areas, the cap) and does not break when you change the table.

How to read the result: the example learner fits several roles at about the same level, with "systems
software" a little ahead. That is normal after a broad course. The low lines are not bad news either; they
show what a move into that role would need. In the exercises you add a list of **gaps** for each role.

> 📏 **Convention:** Rate yourself again every three months and keep the old ratings in your journal. The
> change over time is more useful than any single number.

## 5. Choosing a direction for now

Combine three things:

1. **Your role fit** from the program, corrected by your honest judgement.
2. **Your specialization** from Lesson 22.1. A databases track points to the database role, the
   networking or eBPF track to networking, the cloud-native track to platform or infrastructure.
3. **What you enjoy.** Look at your journal: which lessons made you forget the time?

Write two sentences in `notes/career.md`: "For the next two years I aim at the **X** role, because …
My biggest gaps for it are **A** and **B**." The next lessons turn those sentences into a portfolio, open
source work and interview practice.

## Common mistakes

**Comparing references of different depth.** Inside closures over slices you often get `&&str` and
`&&&str`. If you write `area == need` where the two sides have different numbers of `&`, the compiler
refuses:

```text
error[E0277]: can't compare `str` with `&str`
  --> src/bin/roles.rs:41:40
   |
41 |                 .find(|(area, _)| area == need)
   |                                        ^^ no implementation for `str == &str`
   |
   = help: the trait `PartialEq<&str>` is not implemented for `str`
help: consider dereferencing here
   |
41 |                 .find(|(area, _)| area == *need)
   |                                           +
```

The fix is to remove one level with `*`, as the help line says. (You will meet this code in Exercise 2.)

**Choosing by title or by hype.** A title like "Rust Engineer" says nothing about the work. Read the
"What you will do" section.

**Treating the fit percentage as truth.** It only shows what you typed. Use it to start a conversation
with yourself, not to make the decision for you.

**Ignoring the operations part of a role.** On-call duty, incident reviews and maintenance are part of
infrastructure, SRE and platform roles. Decide honestly whether you want that.

## Try it yourself

1. Open `src/bin/roles.rs` and change `my_ratings` to your own honest scores.
2. Run `cargo run --bin roles` and write the top three roles in `notes/career.md`.
3. Find two real job descriptions for your top role (company career pages or community job boards) and
   mark every requirement line with H, L or G, as in Section 3.
4. Commit the new program:

```bash
cd ~/projects/rust
git add code/p24-career notes/career.md
git commit -m "Add p24-career with a role-fit program"
```

## Exercises

### Exercise 1 — A ninth role (easy)

Add a **performance engineer** role that needs `performance`, `low-level`, `linux` and `concurrency`.
Before you run the program, compute its fit for the example ratings by hand. Then run it and check.

<details>
<summary>Hint</summary>

Add one line to `ROLES`. The scores of the four areas are 1, 1, 2 and 2; the maximum is 3 × 4.

</details>

<details>
<summary>Answer</summary>

```rust
    ("performance", &["performance", "low-level", "linux", "concurrency"]),
```

By hand: (1 + 1 + 2 + 2) × 100 / 12 = 600 / 12 = 50%. The output gains one line, between
`networking` and `platform` (equal 50%, alphabetical order):

```text
performance        50% #####
```

Why it works: `fit` looks only at the areas listed for the role, so adding a role changes nothing else.

</details>

### Exercise 2 — Show the gaps (medium)

Write a function `gaps(needs, ratings)` that returns the areas of a role that you rated **below 2**
(including areas you did not rate at all), in the order of the role's list. Print them after each bar.
Write a test first.

<details>
<summary>Hint</summary>

Use `needs.iter().filter(...)`. Inside the filter, look for the area in `ratings` with `find` and use
`map_or(0, ...)` for "not rated". The returned slices point into `needs`, so the signature needs a
lifetime: `fn gaps<'a>(needs: &[&'a str], ratings: &[(&str, u32)]) -> Vec<&'a str>`.

</details>

<details>
<summary>Answer</summary>

```rust
/// Returns the areas of a role that you rated below 2, in the role's order.
fn gaps<'a>(needs: &[&'a str], ratings: &[(&str, u32)]) -> Vec<&'a str> {
    needs
        .iter()
        .filter(|need| {
            let score = ratings
                .iter()
                .find(|(area, _)| area == *need)
                .map_or(0, |(_, score)| *score);
            score < 2
        })
        .copied()
        .collect()
}

#[test]
fn gaps_lists_low_and_missing_areas() {
    let ratings = [("a", 3), ("b", 1)];
    assert_eq!(gaps(&["a", "b", "c"], &ratings), vec!["b", "c"]);
    assert!(gaps(&["a"], &ratings).is_empty());
}
```

The test goes inside `mod tests`. In `main`, collect a third value and print it:

```rust
    let mut results: Vec<(u32, &str, Vec<&str>)> = ROLES
        .iter()
        .map(|(name, needs)| (fit(needs, &my_ratings), *name, gaps(needs, &my_ratings)))
        .collect();
    results.sort_by(|a, b| b.0.cmp(&a.0).then(a.1.cmp(b.1)));

    for (percent, name, missing) in results {
        let bar = "#".repeat((percent / 10) as usize);
        println!(
            "{name:<17} {percent:>3}% {bar:<10} gaps: {}",
            missing.join(", ")
        );
    }
```

With the example ratings and the role from Exercise 1:

```text
systems software   58% #####      gaps: performance
database           50% #####      gaps: storage, distributed
infrastructure     50% #####      gaps: networking, automation
networking         50% #####      gaps: networking, performance
performance        50% #####      gaps: performance, low-level
platform           50% #####      gaps: automation, observability
SRE                41% ####       gaps: observability, automation, networking
kernel             41% ####       gaps: low-level, hardware
embedded           33% ###        gaps: hardware, low-level
```

Why it works: `filter` keeps an area when its score is below 2, and `copied` turns `&&str` into `&str`.
`{bar:<10}` pads the bar so that the "gaps" column lines up.

</details>

### Exercise 3 — Mark a job description (easy, no code)

Mark the invented job description from Section 3 with **your own** H, L and G, with evidence or a next
step for each line. Then answer: would you apply today?

<details>
<summary>Hint</summary>

Be strict with H: write H only if you can paste a link or a file path as evidence.

</details>

<details>
<summary>Answer</summary>

There is no single right answer. A good answer has six marked lines, evidence for every H, a lesson or
project for every L, and one concrete step for every G (for example "G Open source → Lesson 24.4,
first pull request by the end of the month"). If four of the six lines are H or L, the answer to
"would you apply?" is yes.

</details>

## Quiz

1. Why is the job title a weak signal?

<details>
<summary>Answer</summary>

Companies use the same title for different work, and titles change over time. The "What you will do"
section describes the real role.

</details>

2. What is the difference between an SRE and a platform engineer, in one sentence each?

<details>
<summary>Answer</summary>

An SRE keeps production reliable: SLOs, monitoring, incidents, automation of manual work. A platform
engineer builds the internal tools and services that developers use to build, deploy and run their
software.

</details>

3. Which roles usually need `no_std` Rust?

<details>
<summary>Answer</summary>

Embedded / firmware engineers (no operating system on the device) and kernel engineers (the kernel
cannot use the normal standard library).

</details>

4. You match four of six must-have lines. What do you do?

<details>
<summary>Answer</summary>

Apply. Requirement lists are wish lists. In parallel, plan how to close the two gaps and be ready to
say so in the interview.

</details>

5. In `fit`, why is the score capped with `min(3)`?

<details>
<summary>Answer</summary>

So that a typo or an over-optimistic number (like 9) cannot push the result over 100% or hide a gap in
another area.

</details>

6. What is "evidence" for a skill, and why does it matter?

<details>
<summary>Answer</summary>

Something another person can look at: a repository, a merged pull request, a write-up, a talk. People who
hire cannot see inside your head; evidence lets them check a claim quickly.

</details>

## Summary

- "Systems engineer" covers many roles; read the daily work in a job description, not the title.
- Eight common roles: systems software, infrastructure, SRE, database, networking, embedded, kernel and
  platform engineer. They form layers from the hardware up to the developers.
- Mark each requirement line as H (have, with evidence), L (learning) or G (gap, with a next step), and
  apply when most must-have lines are H or L.
- Rate your skills from 0 to 3 and compare them with what roles need; repeat every three months.
- Choose a direction for the next two years by combining role fit, your specialization and what you enjoy.

## Sources and further reading

- [Site Reliability Engineering — Introduction](https://sre.google/sre-book/introduction/) — the chapter
  that defines the SRE role, free online.
- [Cloud Native Computing Foundation — Platforms White Paper](https://tag-app-delivery.cncf.io/whitepapers/platforms/) —
  what an internal platform is and what platform teams do.
- [Rust for Linux](https://rust-for-linux.com/) — the project that brings Rust into the Linux kernel.
- [The Embedded Rust Book](https://docs.rust-embedded.org/book/) — the starting point for embedded Rust work.
- [Rust — Networking](https://www.rust-lang.org/what/networking) and [Rust — Embedded](https://www.rust-lang.org/what/embedded) —
  the official overview pages of two Rust domains.
- [The Rust Reference — Attributes](https://doc.rust-lang.org/reference/attributes.html) — tool attributes
  such as `#[rustfmt::skip]`.

---

[🏠 Start](../START-HERE.md) · [← Phase 24 — Career and open source](README.md) · [📋 Phase 24](README.md) · [24.2 Building your portfolio →](02-building-your-portfolio.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
