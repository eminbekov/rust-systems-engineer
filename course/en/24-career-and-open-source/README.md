<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 23.10 Checkpoint: capstone review](../23-capstone-projects/10-checkpoint-capstone-review.md) · [24.1 Roles for systems engineers →](01-roles-for-systems-engineers.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/24-career-and-open-source/README.md) <!-- nav -->

# Phase 24 — Career and open source

> **Stage 5 — Mastery and career** · ⏱ ongoing, alongside the other phases

## What this phase is about

This course taught you how computers, operating systems, networks, storage and distributed systems work, and how to
build them in Rust. This last phase is about turning that knowledge into work that other people can see and use: a job,
merged contributions to open-source projects, and a habit of learning that continues after the course.

"Systems engineer" is not one job. The phase starts with eight common roles, from infrastructure and SRE (Site
Reliability Engineering) to database, networking, embedded and kernel work, and shows how to read a job description
honestly. Then you make your work visible: a portfolio of two or three strong projects with clear READMEs, write-ups
and talks.

The middle of the phase is open source. You learn to read a large real codebase (ripgrep) with a method instead of
from top to bottom, practise the whole fork workflow in a safe local laboratory, and learn how to work with maintainers:
precise bug reports, good questions, calm answers to review comments.

The end of the phase prepares you for interviews and for the years after the course: resumes, coding and system design
interviews with a method and a worked example, books and research papers to read next, and habits that keep you
learning. Every lesson adds one small, tested Rust tool to your practice package. The phase runs alongside the others:
start it whenever you like and come back to it every few months.

## What you will be able to do

- [ ] Describe eight systems engineering roles, mark a job description with what you have, are learning and lack, and
  choose a direction for the next two years.
- [ ] Present two or three portfolio projects with READMEs that answer a reader's questions, honest benchmark numbers
  and a clean public history.
- [ ] Read a large unfamiliar Rust codebase with a question, a map and `git grep`, `git log` and `git blame`.
- [ ] Make a first open-source contribution with the fork workflow: issue, branch, test first, signed-off commit,
  rebase and a clear pull request.
- [ ] Write bug reports with minimal reproducible examples, answer review comments well and work in open-source
  communities.
- [ ] Write a resume with strong, honest bullet points and organise a job search.
- [ ] Solve coding problems out loud and run a system design interview in seven steps, with estimates.
- [ ] Keep learning with a reading list, the three-pass method for papers, spaced repetition and a twelve-month plan.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 24.1 | [Roles for systems engineers](01-roles-for-systems-engineers.md) | 120 min |
| 24.2 | [Building your portfolio](02-building-your-portfolio.md) | 120 min |
| 24.3 | [Reading large codebases](03-reading-large-codebases.md) | 135 min |
| 24.4 | [Your first open-source contribution](04-your-first-open-source-contribution.md) | 2.5 h |
| 24.5 | [Working with maintainers and communities](05-working-with-maintainers-and-communities.md) | 120 min |
| 24.6 | [Resume and job search](06-resume-and-job-search.md) | 105 min |
| 24.7 | [Coding and system design interviews](07-coding-and-system-design-interviews.md) | 135 min |
| 24.8 | [Keep learning: books, papers and habits](08-keep-learning.md) | 120 min |
| 24.9 | [Checkpoint: course complete](09-checkpoint-course-complete.md) | 120 min |

## Projects in this phase

- **`code/p24-career`** — one practice package that grows through the phase, one program per lesson in `src/bin/`,
  each with tests: `roles` (role fit), `readme-check`, `loc` (a size map of a codebase), `bug-report`, `resume-check`,
  `intervals` and `estimate` (interview practice), `review` (spaced repetition) and `selfcheck` (final self-assessment).
- **The fork laboratory** (Lesson 24.4, `~/projects/git-playground/oss-lab`) — the full contribution workflow between an
  "upstream" project, your "fork" and your clone, all on your own computer.
- **Your real contributions and portfolio** — your public profile, your project READMEs, your first merged pull request
  and your twelve-month plan (Lessons 24.2, 24.4 and 24.9).

## How to study this phase

1. **Do the code and the real-world task of every lesson.** The Rust tools are small; the important part is using them
   on your own READMEs, resume, flashcards and skill ratings.
2. **Never practise on real projects with fake work.** Use the local laboratory for experiments. On real projects, only
   open issues and pull requests that you mean and can explain in your own words.
3. **Run unfamiliar code only in your VM (virtual machine) `dev` or a container.** Reading another project is safe;
   building and testing it runs other people's code.
4. **Spread the phase out.** Read Lessons 24.1 to 24.3 early, do 24.4 and 24.5 when you have a project you use, and
   24.6 and 24.7 a few weeks before you apply. Repeat the checkpoint every three months.
5. **Keep everything in your notes.** Ratings, job descriptions, contribution logs, interview questions you could not
   answer: these notes are your plan for the next months.

---

[🏠 Start](../START-HERE.md) · [← 23.10 Checkpoint: capstone review](../23-capstone-projects/10-checkpoint-capstone-review.md) · [24.1 Roles for systems engineers →](01-roles-for-systems-engineers.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
