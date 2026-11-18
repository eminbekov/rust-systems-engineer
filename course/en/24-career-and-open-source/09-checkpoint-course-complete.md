<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 24.8 Keep learning: books, papers and habits](08-keep-learning.md) · [📋 Phase 24](README.md) · 🏁 End of the course · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/24-career-and-open-source/09-checkpoint-course-complete.md) <!-- nav -->

# Lesson 24.9 — Checkpoint: course complete

> **Phase 24 — Career and open source** · Lesson 9 of 9 · ⏱ about 120 minutes

## What you will learn

- Whether you can use the ideas of this phase: roles, portfolio, reading code, contributing, working with
  maintainers, resumes, interviews and long-term learning.
- How well the central ideas of the whole course are still in your head, with questions from the earlier stages.
- How to turn an honest self-assessment into a list of lessons to review, with a small Rust program you write
  yourself.
- How to make a twelve-month plan that keeps building, contributing and learning after the course ends.

## Before you start

- You finished Lessons 24.1 to 24.8 of this phase and, ideally, [Phase 23 — Capstone projects](../23-capstone-projects/README.md).
- Your practice package `code/p24-career` contains `roles`, `readme-check`, `loc`, `bug-report`, `resume-check`,
  `intervals`, `estimate` and `review`.
- Plan two sittings: the quiz and Task 1 in the first, Tasks 2 to 4 in the second.

## New words

| Word | Meaning |
|---|---|
| **self-assessment** | Rating your own skills honestly, with evidence, to decide what to learn next. |
| **mock interview** | A practice interview with a friend, a colleague or a study partner, under real time limits. |
| **learning plan** | A written plan for the next months: what you will build, contribute, read and write, and when you review it. |
| **VM** (Virtual Machine) | A whole computer simulated by software, such as your Linux VM `dev` (Lesson 11.1). |
| **DCO** (Developer Certificate of Origin) | The statement you agree to with `git commit -s` (Lesson 24.4). |
| **MRE** (Minimal Reproducible Example) | The smallest complete program that still shows a problem (Lesson 24.5). |
| **CI** (Continuous Integration) | Automatic builds and tests for every change (Lesson 09.12). |
| **API** (Application Programming Interface) | The requests or functions that a service or library offers to others. |
| **p99** | The 99th percentile of latency: the time that 99% of requests stay under (Lesson 19.1). |
| **STAR** (Situation, Task, Action, Result) | The structure for interview stories from Lesson 24.7. |
| **WAL** (write-ahead log) | A log where a database records each change before applying it, so it can recover after a crash (Lesson 17.4). |
| **retrospective** | A short look back at a finished period of work: what went well, what did not, what to change. |

## The course at a glance

| Stage | Phases | What you can do now |
|---|---|---|
| 1. Foundations | 00–06 | Use the terminal and Git; write, test and document Rust with ownership, traits, generics, iterators, errors, threads and `async`. |
| 2. Engineering fundamentals | 07–11 | Choose data structures and algorithms by cost; design clean code; test it in depth; think about security; work in Linux and containers. |
| 3. How computers really work | 12–14 | Explain bits, caches, machine code and memory allocation; use system calls, processes, files, virtual memory; write correct concurrent and async code. |
| 4. Build real systems | 15–18 | Build network servers and protocols, backend services with databases, storage engines, and replicated systems with consensus. |
| 5. Mastery and career | 19–24 | Profile and speed up programs; make them reliable; run them on Kubernetes; go deep in one specialization; finish a capstone; build a career on it. |

## Self-test quiz

Answer each question in your own words first, out loud or on paper. Then open the answer. Questions 1–11 are about this
phase, questions 12–16 about the whole course.

1. A job ad is called "Rust Engineer". What do you read first to know what the job is, and why?

<details>
<summary>Answer</summary>

The "What you will do" section, because it describes the daily work. The title says only how one company names the
role; the same title means different work in different places (Lesson 24.1).

</details>

2. What should a reader find on the first screen of a portfolio project's README?

<details>
<summary>Answer</summary>

What the project is and why it exists, its status, and ideally a small diagram and the command to run it. Many readers
decide on the first screen whether to continue (Lesson 24.2).

</details>

3. Why must every benchmark number in a README come with the machine, the Rust version, the build profile and the
command?

<details>
<summary>Answer</summary>

Without them the number cannot be compared or reproduced; the same code can be several times faster or slower on
another machine or in a debug build (Lesson 24.2).

</details>

4. Name the six steps for reading a large codebase.

<details>
<summary>Answer</summary>

Ask a question; read the documents; draw a map; find the entry points; trace one path; run it and change it
(Lesson 24.3).

</details>

5. Why do you build and test an unfamiliar project only in a VM (virtual machine) or a container?

<details>
<summary>Answer</summary>

Build scripts, procedural macros and tests are code from strangers that runs with your user's rights during
`cargo build` and `cargo test` (Lesson 24.3).

</details>

6. In the fork workflow, which remote do you push to, and what does `git commit -s` add?

<details>
<summary>Answer</summary>

You push to `origin`, your fork; you only fetch from `upstream`. `-s` adds a `Signed-off-by: Name <email>` line, the
sign-off that projects using the DCO (Developer Certificate of Origin) require (Lesson 24.4).

</details>

7. What are the four parts of a good bug report, and what is an MRE (minimal reproducible example)?

<details>
<summary>Answer</summary>

What I did, what I expected, what happened (copied exactly), and the environment. An MRE is the smallest complete
program that still shows the problem (Lesson 24.5).

</details>

8. A maintainer asks for a change that you think is wrong. What do you do?

<details>
<summary>Answer</summary>

Explain your reasons once, briefly and politely. If they still prefer their way, follow it, because they will maintain
the code, or close the pull request politely (Lesson 24.5).

</details>

9. Rewrite "Responsible for the cache" as a strong resume bullet (invent honest-looking details).

<details>
<summary>Answer</summary>

For example: "Added a Redis cache in front of the link lookup; cut database reads by 90% in a load test of 1,000
requests per second." Verb + what + how + result, with a measured number (Lesson 24.6).

</details>

10. List the seven steps of the system design method. Why do estimates come before the diagram?

<details>
<summary>Answer</summary>

Requirements, estimates, API, data model, high-level design, deep dives, trade-offs and wrap-up. Estimates decide what
the design needs: one database or many, a cache or not. Without them people add parts "just in case" (Lesson 24.7).

</details>

11. What do you do in each of the three passes of reading a research paper?

<details>
<summary>Answer</summary>

First: title, abstract, introduction, headings, conclusion, figures, to get the idea. Second: the whole paper without
the hardest details, with a short summary. Third: every step and assumption in depth, ideally re-implementing the core
idea (Lesson 24.8).

</details>

12. In two sentences: what does Rust's ownership system guarantee, and how?

<details>
<summary>Answer</summary>

Every value has exactly one owner, and it is freed when the owner goes out of scope, so memory is freed exactly once.
References are checked at compile time: many shared references or one mutable reference, never both, and never
longer than the value lives (Phase 03).

</details>

13. What does `fsync` guarantee, and why does a database write to a write-ahead log first?

<details>
<summary>Answer</summary>

`fsync` returns only when the file's data has reached durable storage, so it survives a crash or power loss. A
database appends each change to a log and syncs it before it reports success; after a crash it replays the log, so
no acknowledged change is lost (Lessons 17.2 and 17.4).

</details>

14. Why should retries use exponential backoff with jitter, and what must be true of the operation you retry?

<details>
<summary>Answer</summary>

Growing, randomised delays stop many clients from retrying at the same moment and overloading a struggling service
again. The operation must be idempotent (doing it twice has the same effect as once), or retries can duplicate work
(Lesson 18.3).

</details>

15. A service has a good average latency but a bad p99. Name two possible causes and one tool to find the cause.

<details>
<summary>Answer</summary>

For example long pauses while very large structures are allocated or freed, lock contention, a slow dependency for some requests, or a
cache miss path. A profiler with flame graphs, or percentiles per request type in the metrics, shows where the time goes
(Lessons 19.1 and 19.3).

</details>

16. What is a data race, and how does Rust prevent it at compile time?

<details>
<summary>Answer</summary>

Two threads access the same memory at the same time, at least one writes, and there is no synchronisation. Rust's
borrowing rules and the `Send` and `Sync` traits allow shared mutation across threads only through types such as
`Mutex` or atomics, so safe code cannot compile with a data race (Lessons 05.10 and 14.1).

</details>

## Practical tasks

### Task 1 — The final self-assessment program

Write `code/p24-career/src/bin/selfcheck.rs`. It reads your skill ratings from a file (default `skills.txt`), with the
areas and the 0–3 scale of Lesson 24.1 (the `my_ratings` list of `roles.rs`, now in a file): one `area = score` per
line, and prints what to review next.

**Contract:**

- Blank lines and lines starting with `#` are skipped. A bad line stops the program with exit code 2 and the message
  ``line N: expected `area = 0..3`, got `<line>` ``.
- The first output line: `<n> areas rated, average <x.x> of 3`.
- Then `review next:` and one line per area rated below 2, weakest first (equal scores in alphabetical order), with the
  place in the course to review: `  <area padded to 14> <score>  -> <where>`.
- Then one line for every course area that is not in the file at all: `  <area> -  -> not rated yet (<where>)`.
- The course areas and where to review them:

| Area | Where to review |
|---|---|
| rust | Phases 03-06 |
| linux | Phases 11 and 13 |
| networking | Phase 15 |
| storage | Phase 17 |
| concurrency | Phase 14 |
| performance | Phase 19 |
| containers | Lessons 11.12-11.14, Phase 21 |
| automation | Lessons 11.8-11.9, 21.7-21.8 |
| observability | Lessons 16.12, 21.9-21.10 |
| distributed | Phase 18 |
| hardware | Phase 12, Phase 22 |
| low-level | Phase 12, Lessons 06.6 and 06.10 |

- Tests for: parsing (good and bad lines), the order of the weakest areas, and the detection of unrated areas.

**Done when** this `skills.txt`:

```text
# Final self-assessment, 0 (never used) to 3 (can teach it)
rust = 3
linux = 2
networking = 2
storage = 2
concurrency = 2
performance = 1
containers = 2
automation = 1
observability = 1
distributed = 2
hardware = 0
```

gives exactly:

```bash
cargo run -q --bin selfcheck
```

```text
11 areas rated, average 1.6 of 3
review next:
  hardware       0  -> Phase 12, Phase 22
  automation     1  -> Lessons 11.8-11.9, 21.7-21.8
  observability  1  -> Lessons 16.12, 21.9-21.10
  performance    1  -> Phase 19
  low-level      -  -> not rated yet (Phase 12, Lessons 06.6 and 06.10)
```

and a file with the line `linux three` as line 2 gives ``line 2: expected `area = 0..3`, got `linux three` `` and exit code 2.
Then run it on **your own** honest ratings.

<details>
<summary>Hint</summary>

Keep the table as a `const REVIEW: &[(&str, &str)]`. Write four small functions: `parse_ratings` (`split_once('=')` and `trim`, as in `kvparse` of Lesson 24.4),
`weakest` (filter below 2, then `sort_by` score and name), `unrated` (areas of `REVIEW` that no rating mentions) and
`where_to_review` (look up an area in `REVIEW`). `main` only reads the file and prints.

</details>

<details>
<summary>Answer</summary>

```rust
// Final self-assessment: read your skill ratings and say what to review next.

/// Each skill area and where the course teaches it.
const REVIEW: &[(&str, &str)] = &[
    ("rust", "Phases 03-06"),
    ("linux", "Phases 11 and 13"),
    ("networking", "Phase 15"),
    ("storage", "Phase 17"),
    ("concurrency", "Phase 14"),
    ("performance", "Phase 19"),
    ("containers", "Lessons 11.12-11.14, Phase 21"),
    ("automation", "Lessons 11.8-11.9, 21.7-21.8"),
    ("observability", "Lessons 16.12, 21.9-21.10"),
    ("distributed", "Phase 18"),
    ("hardware", "Phase 12, Phase 22"),
    ("low-level", "Phase 12, Lessons 06.6 and 06.10"),
];

/// Parses lines like `linux = 2`. Blank lines and lines starting with `#` are skipped.
fn parse_ratings(text: &str) -> Result<Vec<(&str, u32)>, String> {
    let mut ratings = Vec::new();
    for (index, line) in text.lines().enumerate() {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }
        let bad = || format!("line {}: expected `area = 0..3`, got `{line}`", index + 1);
        let (area, score) = line.split_once('=').ok_or_else(bad)?;
        let score: u32 = score.trim().parse().map_err(|_| bad())?;
        if score > 3 {
            return Err(bad());
        }
        ratings.push((area.trim(), score));
    }
    Ok(ratings)
}

/// Areas rated below 2, weakest first (equal scores in alphabetical order).
fn weakest<'a>(ratings: &[(&'a str, u32)]) -> Vec<(&'a str, u32)> {
    let mut low: Vec<(&str, u32)> = ratings.iter().copied().filter(|(_, s)| *s < 2).collect();
    low.sort_by(|a, b| a.1.cmp(&b.1).then(a.0.cmp(b.0)));
    low
}

/// Areas of the course that have no rating at all.
fn unrated(ratings: &[(&str, u32)]) -> Vec<&'static str> {
    REVIEW
        .iter()
        .map(|(area, _)| *area)
        .filter(|area| !ratings.iter().any(|(rated, _)| rated == area))
        .collect()
}

fn where_to_review(area: &str) -> &'static str {
    REVIEW
        .iter()
        .find(|(name, _)| *name == area)
        .map_or("(not a course area)", |(_, lessons)| lessons)
}

fn main() {
    let path = std::env::args()
        .nth(1)
        .unwrap_or_else(|| "skills.txt".to_string());
    let text = std::fs::read_to_string(&path).unwrap_or_else(|err| {
        eprintln!("cannot read {path}: {err}");
        std::process::exit(2);
    });
    let ratings = parse_ratings(&text).unwrap_or_else(|message| {
        eprintln!("{message}");
        std::process::exit(2);
    });

    let total: u32 = ratings.iter().map(|(_, score)| score).sum();
    println!(
        "{} areas rated, average {:.1} of 3",
        ratings.len(),
        total as f64 / ratings.len().max(1) as f64
    );
    println!("review next:");
    for (area, score) in weakest(&ratings) {
        println!("  {area:<14} {score}  -> {}", where_to_review(area));
    }
    for area in unrated(&ratings) {
        println!(
            "  {area:<14} -  -> not rated yet ({})",
            where_to_review(area)
        );
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parses_good_lines_and_reports_bad_ones() {
        assert_eq!(
            parse_ratings("# me\nrust = 2\n\n linux=3 \n"),
            Ok(vec![("rust", 2), ("linux", 3)])
        );
        assert_eq!(
            parse_ratings("rust = 2\nlinux = 7").unwrap_err(),
            "line 2: expected `area = 0..3`, got `linux = 7`"
        );
        assert!(parse_ratings("rust: 2").is_err());
    }

    #[test]
    fn weakest_first_then_by_name() {
        let ratings = [("storage", 1), ("rust", 3), ("hardware", 0), ("linux", 1)];
        assert_eq!(
            weakest(&ratings),
            vec![("hardware", 0), ("linux", 1), ("storage", 1)]
        );
    }

    #[test]
    fn finds_areas_without_a_rating() {
        let all: Vec<(&str, u32)> = REVIEW.iter().map(|(a, _)| (*a, 2)).collect();
        assert!(unrated(&all).is_empty());
        assert_eq!(unrated(&all[1..]), vec!["rust"]);
    }
}
```

```text
running 3 tests
test tests::finds_areas_without_a_rating ... ok
test tests::weakest_first_then_by_name ... ok
test tests::parses_good_lines_and_reports_bad_ones ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: the closure `bad` builds one message for every kind of bad line and `?` returns it early. `weakest` borrows
the area names from the file's text, which the lifetime `'a` expresses; `unrated` returns names from the `'static` table.
`ratings.len().max(1)` avoids a division by zero for an empty file. Every function except `main` is pure, so the tests need
no files.

</details>

### Task 2 — Audit your portfolio (no new code)

Make your public work ready for a stranger's eyes. For each portfolio project, run `readme-check` (Lesson 24.2) on its
README; run `resume-check` (Lesson 24.6) on your resume; then check by hand the points that no program can check.

<details>
<summary>Hint</summary>

Work from the outside in: profile page, then each project's first screen, then its commands from a fresh clone, then the
resume. Use a checklist and tick each item only with evidence.

</details>

<details>
<summary>Answer</summary>

The audit is done when all of these are true:

```text
[ ] readme-check prints "all 6 sections present" for every portfolio project
[ ] resume-check prints "0 of N bullets need work" (or every remaining warning is a conscious choice)
[ ] a fresh clone of each project passes: cargo fmt --check, cargo clippy -- -D warnings, cargo test
[ ] every CI badge is green, every link in the README and the resume opens
[ ] git log -p shows no secrets, no employer material, no personal data of others
[ ] the profile page names the role you aim for and links to the projects and merged pull requests
[ ] a friend can say in two minutes what each project does
```

</details>

### Task 3 — A mock interview

Ask a friend, a colleague or a study partner to interview you for 90 minutes: 45 minutes of coding (a problem from
Lesson 07.18 or 24.7 that you have **not** solved yet) and 45 minutes of system design. Choose the design question from:
a rate limiter for an API, a metrics agent that runs on every server, or a job queue with retries. Use the methods of
Lesson 24.7. Afterwards, both of you fill in the rubric.

<details>
<summary>Hint</summary>

Give your interviewer the seven steps of the design method and the rubric before you start, so they know what to watch.
Record the session if both agree, and watch it once.

</details>

<details>
<summary>Answer</summary>

A rubric with four levels per row (0 = missing, 1 = weak, 2 = good, 3 = strong):

| Row | Look for |
|---|---|
| Understanding | Asked about input, output, limits and edge cases before coding; wrote down requirements before designing. |
| Communication | Thought out loud; explained choices; asked when unsure. |
| Correctness | Code works on the examples; tests for edge cases; design handles the stated load. |
| Numbers | Stated complexity; made estimates; let numbers decide the design. |
| Failures | Named what breaks and what happens then: crashes, slow dependencies, duplicates, overload. |
| Trade-offs | Said what each choice wins and loses, and what to do next. |

For the rate limiter, for example, a strong answer clarifies the limit (requests per client per second), estimates the
number of clients, chooses an algorithm (token bucket), decides where the counters live (in each server or in a shared
store such as Redis) and discusses what happens when that store is down (fail open or fail closed). Write the two rows
with the lowest scores in your journal: they are your study plan for the next weeks.

</details>

### Task 4 — Your twelve-month plan

Write `notes/plan-12-months.md`: for each month, one thing to **build**, one to **contribute**, one to **learn** (a book
part or a paper) and one to **write**. Add the dates when you will review the plan.

<details>
<summary>Hint</summary>

Start from three inputs: your weakest areas from Task 1, your role from Lesson 24.1, and your rhythm from Lesson 24.8.
Plan about ten hours per week. If a month becomes too full, cut "write" to a short note, never cut "build".

</details>

<details>
<summary>Answer</summary>

An example skeleton (fill it with your own topics):

```text
| Month | Build                              | Contribute                           | Learn                         | Write                      |
|-------|------------------------------------|--------------------------------------|-------------------------------|----------------------------|
| 1     | finish capstone README, benchmarks | first docs or test pull request      | weakest area: 1 book chapter  | capstone write-up          |
| 2     | tool for my weakest area           | second pull request, same project    | paper, first and second pass  | note on the review I got   |
| 3     | improve capstone after feedback    | triage: reproduce 3 bug reports      | book chapters 2-3, with code  | resume update (24.6)       |
| 4-6   | second deep project in my track    | a code pull request with a test      | one paper per month           | one write-up per month     |
| 7-9   | ...                                | ...                                  | ...                           | ...                        |
| 10-12 | ...                                | ...                                  | ...                           | lightning talk             |

Reviews: every 3 months, re-run selfcheck with new ratings and compare.
```

A good plan is small enough to survive two bad weeks, names concrete projects and sources, and has review dates in
the calendar.

</details>

## If you got something wrong

| Topic | Review this lesson |
|---|---|
| Roles, job descriptions, H/L/G marking | [Lesson 24.1 — Roles for systems engineers](01-roles-for-systems-engineers.md) |
| Portfolio, READMEs, benchmarks, licences | [Lesson 24.2 — Building your portfolio](02-building-your-portfolio.md) |
| Reading code, `git grep`, `git blame`, partial clones | [Lesson 24.3 — Reading large codebases](03-reading-large-codebases.md) |
| Forks, `upstream`, sign-off, rebase, pull requests | [Lesson 24.4 — Your first open-source contribution](04-your-first-open-source-contribution.md) |
| Bug reports, MREs, review comments, communities | [Lesson 24.5 — Working with maintainers and communities](05-working-with-maintainers-and-communities.md) |
| Resume sections and bullets, job search | [Lesson 24.6 — Resume and job search](06-resume-and-job-search.md) |
| Coding and system design interviews, STAR | [Lesson 24.7 — Coding and system design interviews](07-coding-and-system-design-interviews.md) |
| Books, papers, habits, spaced repetition | [Lesson 24.8 — Keep learning: books, papers and habits](08-keep-learning.md) |
| Ownership | [Lesson 03.2 — The ownership rules](../03-ownership-and-borrowing/02-ownership-rules.md) |
| Threads and data races | [Lesson 05.10 — Threads, channels and shared state](../05-rust-intermediate/10-threads-channels-and-shared-state.md) and [Lesson 14.1 — Why concurrency is hard](../14-concurrency-and-async/01-why-concurrency-is-hard.md) |
| `fsync` and the write-ahead log | [Lesson 17.2 — Durability: fsync and crash safety](../17-storage-and-databases/02-durability-and-fsync.md) and [Lesson 17.4 — The write-ahead log (WAL)](../17-storage-and-databases/04-write-ahead-log.md) |
| Retries, backoff, idempotency | [Lesson 18.3 — Retries, backoff and idempotency](../18-distributed-systems/03-retries-backoff-and-idempotency.md) |
| Percentiles and profiling | [Lesson 19.1 — Latency, throughput and percentiles](../19-performance-and-profiling/01-latency-throughput-and-percentiles.md) and [Lesson 19.3 — Profilers and flame graphs](../19-performance-and-profiling/03-profilers-and-flame-graphs.md) |

## What's next

This is the last lesson of the course. The course ends; the work it prepared you for does not. Phase 24 is meant to run
alongside everything else, so come back to it every three months: rate yourself again (Task 1), update your portfolio
(Task 2), practise an interview (Task 3) and adjust the plan (Task 4).

Before you close the course:

1. **Write a retrospective** in your learning journal: the three hardest topics, the three projects you are proudest of,
   and the one habit you will keep. Add the date.
2. **Schedule your reviews:** put the quiz topics you missed on your calendar for tomorrow, in a week and in a month, and
   add them to your `cards.txt` (Lesson 24.8).
3. **Keep the course as a reference.** The [Glossary](../GLOSSARY.md) explains every term and abbreviation, and the
   [start page](../START-HERE.md) links to every phase. Re-reading a lesson after a year often shows you how far you came.
4. **Choose your next step** from your plan: the next pull request, the next chapter, the next project. Start it this week.

You began with the terminal and your first `cargo new`. You now understand how programs, operating systems, networks,
storage and distributed systems work, and you can build them in Rust. Keep building.

## Sources and further reading

- [This Week in Rust](https://this-week-in-rust.org/) — weekly news, the "Call for Participation" for contributors, and
  events.
- [The Rust Blog](https://blog.rust-lang.org/) — release notes of every Rust version.
- [Open Source Guides](https://opensource.guide/) — guides for contributors and maintainers.
- [Tech Interview Handbook](https://www.techinterviewhandbook.org/) — free interview preparation material.
- [Papers We Love](https://github.com/papers-we-love/papers-we-love) — a community collection of papers for your
  reading plan.

---

[🏠 Start](../START-HERE.md) · [← 24.8 Keep learning: books, papers and habits](08-keep-learning.md) · [📋 Phase 24](README.md) · 🏁 End of the course · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
