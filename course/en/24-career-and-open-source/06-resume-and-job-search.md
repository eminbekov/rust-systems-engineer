<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 24.5 Working with maintainers and communities](05-working-with-maintainers-and-communities.md) · [📋 Phase 24](README.md) · [24.7 Coding and system design interviews →](07-coding-and-system-design-interviews.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/24-career-and-open-source/06-resume-and-job-search.md) <!-- nav -->

# Lesson 24.6 — Resume and job search

> **Phase 24 — Career and open source** · Lesson 6 of 9 · ⏱ about 105 minutes

## What you will learn

- What a resume is for, who reads it and how, and which sections a systems engineer's resume needs.
- How to write bullet points that show what you did and what came out of it, honestly.
- How to tailor a resume to one job description, and check it with a small Rust program and its tests.
- How to organise a job search: where to look, how to track applications, how to follow up and how to handle
  rejection.

## Before you start

- You finished [Lesson 24.5 — Working with maintainers and communities](05-working-with-maintainers-and-communities.md).
- You marked job descriptions with H, L and G in [Lesson 24.1](01-roles-for-systems-engineers.md) and chose your
  portfolio projects in [Lesson 24.2](02-building-your-portfolio.md). This lesson uses both.

## New words

| Word | Meaning |
|---|---|
| **resume** | A short document (one or two pages) that summarises your skills, projects and experience for a job application. |
| **CV** (curriculum vitae) | Latin for "course of life". In many countries it means the same as a resume; in some it means a longer, complete record. |
| **bullet point** | One line in a list, starting with a dash or a dot. A resume is mostly bullet points. |
| **tailoring** | Adjusting your resume to one job description: which projects come first, which words you use. |
| **ATS** (Applicant Tracking System) | Software that companies use to collect and search job applications. People still decide, but they often search the stored resumes by words. |
| **PDF** (Portable Document Format) | A file format that looks the same on every computer. Most applications ask for a resume as a PDF. |
| **cover letter** | A short letter sent with the resume that says why you apply for this job. |
| **referral** | A recommendation of you for a job by someone who works at the company. |
| **recruiter** | A person whose job is to find and contact candidates, inside a company or for an agency. |
| **percentile** | A value below which a given share of measurements falls; p99 latency is the time that 99% of requests stay under (Lesson 19.1). |
| **UTF-8** (8-bit Unicode Transformation Format) | The standard way to store text as bytes (Lesson 12.5): English letters take one byte, most other letters two or more. |
| **TCP** (Transmission Control Protocol) | The reliable network protocol of Phase 15; it appears in the example resume. |

## 1. What a resume is for

A resume has one job: to make a busy person think "this one is worth a conversation". It does not have to tell
your whole story; the interviews do that.

Who reads it? Often a recruiter first, then an engineer from the team. Each reads quickly, first scanning the
top half of the first page. Many companies store applications in an **ATS**, and people search those systems by
words such as "Rust", "Linux" or "Kubernetes". Two practical rules follow:

1. **The most important facts go to the top**: your role and main skills in one line, then your strongest
   project or experience.
2. **Use a simple layout**: one column, normal headings ("Projects", "Experience", "Skills"), real text (not text
   inside images), saved as a **PDF**. Fancy designs often break when software reads them.

One page is enough for most people early in their career. Two pages are fine when you have years of relevant
experience. Nobody reads three.

## 2. The sections of a systems engineer's resume

| Section | Content | Tips |
|---|---|---|
| Header | Name, email, link to your portfolio or code hosting profile, city or time zone if relevant. | No photo, no date of birth, no full address unless your country expects it. |
| Summary | One or two lines: the role you aim for and your core skills. | "Systems engineer: Rust, Linux, networking and storage." |
| Skills | Grouped: languages, systems topics, tools. | Only what you can discuss in an interview. |
| Projects | Your two or three portfolio projects (Lesson 24.2), each with two or three bullets and a link. | Put them **above** experience if they are more relevant than your past jobs. |
| Experience | Jobs, newest first: title, organisation, dates, two to four bullets each. | Earlier non-technical jobs can stay, shortly; they show reliability. |
| Open source | Merged contributions with links (Lessons 24.4 and 24.5). | Even one merged pull request is worth a line. |
| Education | Degrees, courses, certifications. | This course can be listed as self-study with its main projects. |

> 🔒 **Security:** A resume is often forwarded and stored for years. Leave out anything you would not want in a
> public place: your full address, identity document numbers, a photo of documents. Never mention confidential
> details of an employer, such as internal system names or customer names.

## 3. Bullet points: what you did and what came out of it

A weak bullet describes a duty. A strong bullet describes an **action** and its **result**. A simple formula:

> **verb** (what you did) + **what** (the thing) + **how** (the key technique) + **result** (a number or a fact)

| Weak | Strong |
|---|---|
| Responsible for a key-value store. | Built a key-value store in Rust with a write-ahead log and crash recovery; 52 tests, including one that kills the process during writes. |
| Worked on performance. | Cut the p99 latency of a request handler from 12 ms to 4 ms by removing allocations in the hot path, measured with criterion and flame graphs. |
| Helped with open source. | Fixed an empty-key bug in an open-source parsing crate; the change and its test were merged after review. |

Some rules:

- **Start with a strong verb**: built, wrote, designed, measured, reduced, automated, fixed, migrated.
- **Numbers make claims concrete**: sizes (lines, tests, requests per second), results (latency, memory),
  counts (services, users, pull requests). Use real numbers that you measured and can explain.
- **Name the technique**: "with a write-ahead log", "with epoll", "with property-based tests". It shows depth and
  matches the words people search for.
- **No "I"**: resume bullets leave out the subject. "Built …", not "I built …".
- **Be honest.** Every line is a possible interview question. If you cannot explain a claim for five minutes,
  remove it. Course projects are real projects; call them projects, not jobs.

## 4. Check bullets with a program

These rules are mechanical enough for a program to catch the worst cases. Create
`code/p24-career/src/bin/resume-check.rs`:

```rust
// Checks the bullet points of a resume against a few simple writing rules.
use std::process::ExitCode;

const WEAK_STARTS: &[&str] = &[
    "responsible for",
    "worked on",
    "helped",
    "participated in",
    "was involved",
];

/// Returns the problems found in one bullet; an empty list means it looks fine.
fn problems(bullet: &str) -> Vec<&'static str> {
    let mut found = Vec::new();
    let lower = bullet.to_lowercase();
    if WEAK_STARTS.iter().any(|weak| lower.starts_with(weak)) {
        found.push("starts with a weak phrase; start with what you did");
    }
    if !bullet.chars().any(|c| c.is_ascii_digit()) {
        found.push("no number; add a size, a count or a measured result");
    }
    if bullet.chars().count() > 160 {
        found.push("longer than 160 characters; shorten or split it");
    }
    found
}

fn main() -> ExitCode {
    let Some(path) = std::env::args().nth(1) else {
        eprintln!("usage: resume-check <resume.md>");
        return ExitCode::from(2);
    };
    let text = match std::fs::read_to_string(&path) {
        Ok(text) => text,
        Err(err) => {
            eprintln!("cannot read {path}: {err}");
            return ExitCode::from(2);
        }
    };

    let (mut total, mut weak) = (0, 0);
    for (index, line) in text.lines().enumerate() {
        let Some(bullet) = line.trim().strip_prefix("- ") else {
            continue;
        };
        total += 1;
        let found = problems(bullet);
        if !found.is_empty() {
            weak += 1;
            println!("line {}: {bullet}", index + 1);
            for problem in found {
                println!("    {problem}");
            }
        }
    }
    println!("{weak} of {total} bullets need work");
    if weak == 0 {
        ExitCode::SUCCESS
    } else {
        ExitCode::FAILURE
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_good_bullet_has_no_problems() {
        let bullet =
            "Built a key-value store in Rust with a write-ahead log; 40 tests, crash-tested";
        assert!(problems(bullet).is_empty());
    }

    #[test]
    fn weak_start_is_found_in_any_case() {
        assert_eq!(problems("Responsible for 3 services").len(), 1);
    }

    #[test]
    fn missing_number_is_found() {
        assert_eq!(
            problems("Wrote a shell in Rust"),
            vec!["no number; add a size, a count or a measured result"]
        );
    }

    #[test]
    fn long_bullets_are_found() {
        let long = format!("Built 1 {}", "very ".repeat(40));
        assert!(problems(&long).contains(&"longer than 160 characters; shorten or split it"));
    }
}
```

**What happens:**

1. `problems` checks one bullet against three rules and collects a message for each broken rule. Returning a
   list (instead of printing) keeps the function easy to test.
2. The weak-start check lowercases the text first, so "Responsible for" and "responsible for" are both found.
3. `c.is_ascii_digit()` is true for `0` to `9`; `any` stops at the first digit. `chars().count()` counts
   characters, not bytes, so a bullet with non-English letters is measured correctly (Lesson 04.8).
4. In `main`, only lines that start with `- ` (after trimming) are bullets; `let ... else { continue; }` skips the
   others. Line numbers start at 1 for people, so the program prints `index + 1`.
5. The exit code is 1 when some bullets need work, as in `readme-check` (Lesson 24.2).

Write a draft resume in Markdown, `demo/resume.md`. This one belongs to the invented learner Ada:

```markdown
# Ada Learner

Systems engineer (Rust, Linux). ada@example.com · example.com/ada

## Projects

- Built minikv, a key-value store in Rust with a write-ahead log and crash recovery; 52 tests, including a test that kills the process during writes
- Worked on a TCP load balancer
- Wrote a Unix shell with pipes, redirection and job control in about 1,500 lines of Rust

## Open source

- Helped with documentation in a Rust crate
```

```bash
cargo run -q --bin resume-check -- demo/resume.md; echo "exit code: $?"
cargo test --bin resume-check
```

```text
line 8: Worked on a TCP load balancer
    starts with a weak phrase; start with what you did
    no number; add a size, a count or a measured result
line 13: Helped with documentation in a Rust crate
    starts with a weak phrase; start with what you did
    no number; add a size, a count or a measured result
2 of 4 bullets need work
exit code: 1
```

```text
running 4 tests
test tests::a_good_bullet_has_no_problems ... ok
test tests::long_bullets_are_found ... ok
test tests::weak_start_is_found_in_any_case ... ok
test tests::missing_number_is_found ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Writing the resume in Markdown has a second advantage: it lives in Git, so you can keep one version per job
application on a branch and see what you changed. Convert it to PDF with your editor or a document tool when you
send it.

> 🧪 **Testing:** The checker is a heuristic (Lesson 24.3): a bullet with a number can still be weak, and a
> bullet without one can be fine. The tests pin down what the rules *do*, so that you can change a rule on purpose
> and see which tests react.

## 5. Tailoring to one job description

A resume that tries to fit every job fits none. For each application that you really want, spend twenty minutes:

1. Take the job description and your H/L/G marks from Lesson 24.1.
2. **Reorder**: the project that matches the "What you will do" section best goes first.
3. **Use their words, honestly**: if they write "observability" and you built metrics and traces, write
   "observability (metrics, traces)", not only "Prometheus".
4. **Cut what does not help** this application, so the relevant lines stand out.
5. Write a **short cover letter** if they ask for one: three paragraphs. Why this team; which two of your projects
   match their work and how; that you would like to talk.

> ⚠️ **Common mistake:** Adding skills from the job description that you do not have. It may pass a search, but
> the first technical question ends it, and it costs trust. Write "learning: X" in your notes instead, and close the
> gap.

## 6. Organising the search

**Where to look.**

- Career pages of companies whose software you use or admire.
- Job boards of communities, newsletters and meetups. The Rust community has job threads and job sections in
  some newsletters; other communities (Linux, Kubernetes, databases) have their own.
- **People.** Maintainers and contributors you met in open source, people you met at meetups, former colleagues.
  A referral means someone vouches for you; ask for it only from people who know your work.

**Track every application.** A small table in your notes is enough:

```text
| Date       | Company   | Role                  | Source        | Status          | Next step           |
|------------|-----------|-----------------------|---------------|-----------------|---------------------|
| 2026-11-20 | Example A | Infrastructure Eng.   | career page   | applied         | follow up 12-04     |
| 2026-11-22 | Example B | Systems Engineer      | referral      | interview 12-01 | prepare design talk |
```

**Follow up once.** If you hear nothing after about two weeks, one short, polite email to the recruiter is fine.

**Expect rejections.** Most applications end without an offer, for everyone, often for reasons you never learn: the
role was filled internally, the budget changed, another candidate had one more year of a specific skill. Count
applications and conversations, not rejections. After every interview, write down which questions were hard and
study those (Lesson 24.7).

**Keep a pace you can sustain.** A few tailored applications per week are better than many generic ones. Keep
learning and contributing in parallel: they make each next application stronger.

## Common mistakes

**A list of duties instead of results.** "Responsible for deployments" says what was expected of you, not what you
achieved.

**Unclear dates or gaps hidden badly.** It is fine to have a gap; a line like "2025–2026: full-time study of systems
programming (self-study, projects below)" is honest and useful.

**Spelling mistakes in technology names.** Write "PostgreSQL", "Kubernetes", "Rust", exactly as the projects do.
Careless names suggest careless code.

**A design that software cannot read.** Two columns, icons for skills, text in images: keep it simple.

**Forgetting the links.** A reader who likes a project wants to click on it. Check every link in the PDF.

## Try it yourself

1. Write your resume in `notes/resume.md` with the sections of Section 2, then run `resume-check` on it until it
   reports no problems you agree with.
2. Choose one real job description and make a tailored copy: `git switch -c resume/<company>` in your notes
   repository is a simple way to keep versions.
3. Ask one person who works as an engineer to read your resume for 60 seconds and tell you what they remember.
4. Start the application table from Section 6 in your notes.

## Exercises

### Exercise 1 — Rewrite the weak bullets (easy)

Rewrite the two bullets that `resume-check` reported in `demo/resume.md`, so that the checker reports
`0 of 4 bullets need work`. Use only facts that a learner who finished this course could honestly claim.

<details>
<summary>Hint</summary>

Use the formula: verb + what + how + result. The load balancer is Lesson 15.15; the documentation fix could be
your first contribution from Lesson 24.4.

</details>

<details>
<summary>Answer</summary>

For example:

```markdown
- Wrote a TCP load balancer in Rust with health checks and round-robin over 3 backends; tested failover when 1 backend stops
- Fixed 2 wrong examples in the documentation of an open-source Rust crate (merged pull request)
```

```text
0 of 4 bullets need work
```

Why it works: each bullet now starts with a verb, names the technique and has a concrete number. Both are things you
can explain in detail in an interview.

</details>

### Exercise 2 — No "I" (medium)

Add a rule: a bullet that starts with "I " or contains " I " gets the message `written with "I"; start with the verb
instead`. Write the test first and make sure that "Built 2 tools in Rust" still has no problems.

<details>
<summary>Hint</summary>

You already have `lower`, the lowercased bullet. Check `lower.starts_with("i ")` and `lower.contains(" i ")`.

</details>

<details>
<summary>Answer</summary>

In `problems`, before the length rule:

```rust
    if lower.starts_with("i ") || lower.contains(" i ") {
        found.push("written with \"I\"; start with the verb instead");
    }
```

and in `mod tests`:

```rust
    #[test]
    fn first_person_is_found() {
        let expected = vec!["written with \"I\"; start with the verb instead"];
        assert_eq!(problems("I built 2 tools"), expected);
        assert_eq!(problems("Built 2 tools that I use daily"), expected);
        assert!(problems("Built 2 tools in Rust").is_empty());
    }
```

```text
test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Why it works: lowercasing makes `I` into `i`, and the spaces around `i` make sure that words such as "in" or "is" do
not match. The third assertion protects against exactly that mistake.

</details>

### Exercise 3 — Summary lines for two roles (easy, no code)

Write two different summary lines for your resume: one for an infrastructure engineer role and one for a database
engineer role. Each must be true for you.

<details>
<summary>Hint</summary>

Name the role first, then the three skills that matter most for it, then your strongest evidence.

</details>

<details>
<summary>Answer</summary>

Examples (yours must match your real work):

```text
Infrastructure engineer: Linux, containers and Kubernetes; ran my own Rust service on a local cluster with metrics and alerts.
Database engineer: storage engines in Rust (write-ahead log, LSM tree, crash recovery) and replication with Raft.
```

The same person can write both, because both are true; tailoring chooses what to show first, it does not invent.
(LSM tree means log-structured merge tree, the storage structure of Lesson 17.7; on a real resume, use the short
form only when the readers of that job know it.)

</details>

## Quiz

1. Why should the most important facts be at the top of the first page?

<details>
<summary>Answer</summary>

Readers scan quickly and often decide after the top half of the first page whether to read further.

</details>

2. What is an ATS, and what does it mean for your layout?

<details>
<summary>Answer</summary>

An Applicant Tracking System stores and searches applications. Use a simple one-column layout with normal headings
and real text, so that the software reads your resume correctly and searches find your skills.

</details>

3. What makes a bullet point strong?

<details>
<summary>Answer</summary>

It starts with what you did, names the thing and the technique, and gives a concrete, true result, often a number.

</details>

4. Is it acceptable to list skills from the job description that you do not have yet?

<details>
<summary>Answer</summary>

No. It will come out in the interview and costs trust. Show what you have, and work on the gap.

</details>

5. Why does `resume-check` use `chars().count()` and not `len()` for the length rule?

<details>
<summary>Answer</summary>

`len()` counts bytes. Letters outside English take two or more bytes in UTF-8, so `len()` would call a bullet with
such letters too long too early; `chars().count()` counts characters.

</details>

## Summary

- A resume has one job: to get you a conversation. Put the most important facts at the top; keep the layout simple.
- Sections: header, summary, skills, projects, experience, open source, education. Projects can come before
  experience.
- Strong bullets: verb + what + how + result, with true numbers, without "I".
- Tailor each important application to the job description, honestly; never claim skills you do not have.
- Search through career pages, communities and people; track applications; follow up once; expect rejections and
  learn from each interview.

## Sources and further reading

- [Harvard Mignone Center for Career Success — Create a Strong Resume](https://careerservices.fas.harvard.edu/resources/create-a-strong-resume/) —
  a free guide with examples of strong bullet points.
- [Massachusetts Institute of Technology, Career Advising — Resumes](https://capd.mit.edu/resources/resumes/) — resume guides and examples for technical
  roles.
- [Tech Interview Handbook — Resume](https://www.techinterviewhandbook.org/resume/) — a free, community-written guide
  for software engineering resumes.
- [The Tech Resume Inside Out](https://thetechresume.com/) — a book about resumes for software engineers, written from
  the hiring side.
- [std::primitive::str](https://doc.rust-lang.org/std/primitive.str.html) — `strip_prefix`, `to_lowercase`, `chars`
  and the other string methods used here.

---

[🏠 Start](../START-HERE.md) · [← 24.5 Working with maintainers and communities](05-working-with-maintainers-and-communities.md) · [📋 Phase 24](README.md) · [24.7 Coding and system design interviews →](07-coding-and-system-design-interviews.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
