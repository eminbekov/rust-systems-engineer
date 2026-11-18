<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 00.8 Study routine, notes in Markdown and checkpoint](../00-getting-started/08-study-routine-notes-and-checkpoint.md) · [01.1 What version control is →](01-what-is-version-control.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/01-git-and-version-control/README.md) <!-- nav -->

# Phase 01 — Git and version control

> **Stage 1 — Foundations: tools, Git and the Rust language** · ⏱ about 2–3 weeks at ~10 hours per week

## What this phase is about

Every program changes over time. You add a feature, you break something, you want yesterday's version back, or a teammate changes the same file as you. **Version control** is the tool that records every change, so that you can see what happened, undo mistakes and work together without overwriting each other. **Git** is the version control system that almost everybody uses, and the Rust project itself, Cargo and most Rust libraries live in Git repositories.

In this phase you learn Git from zero, on your Mac, in the terminal. You start with the idea of a history of snapshots (commits), then learn to look at it, to undo mistakes safely, and to work in parallel with branches. You learn what to do when two changes collide (a merge conflict). Then you connect to the outside world: GitHub and GitLab, SSH (Secure Shell) keys, and the way real teams work with pull requests, code review and automatic checks.

Why now, before the Rust language? Because from the first Rust lesson on you will write code that you want to keep safe. A good habit built today, "small commits on a branch, never lose work", will save you many times. At the end of the phase you build a real learning repository for the whole course, with a Rust project, CI (Continuous Integration, automatic checks on every push) and a tagged release.

You do not need any previous Git or programming knowledge. Everything is explained inside the lessons, and every command was run to get the output you see.

## What you will be able to do

- [ ] Explain what version control is and why Git, GitHub and GitLab are three different things.
- [ ] Install and configure Git, and set your identity, safe defaults and a global ignore file.
- [ ] Create a repository, stage and commit changes with good messages, and ignore files with `.gitignore` (including for a Cargo project).
- [ ] Read the history and the changes with `git log`, `git diff`, `git show`, `git blame` and `git bisect`.
- [ ] Undo mistakes safely with `restore`, `amend`, `reset`, `revert`, `stash` and `reflog`.
- [ ] Use branches, merge them, and resolve a merge conflict by hand.
- [ ] Work with a remote: clone, fetch, pull and push over SSH, and keep secrets out of the history.
- [ ] Follow the team workflow: pull request, code review, rebase, Conventional Commits, SemVer tags and a CI check for a Rust project.

## Lessons

| # | Lesson | Time |
|---|---|---|
| 01.1 | [What version control is](01-what-is-version-control.md) | 105 min |
| 01.2 | [Installing and configuring Git](02-install-and-configure-git.md) | 120 min |
| 01.3 | [Your first repository and commits](03-your-first-repository-and-commits.md) | 3 h |
| 01.4 | [Viewing history and changes](04-viewing-history-and-changes.md) | 3 h |
| 01.5 | [Undoing changes safely](05-undoing-changes-safely.md) | 2.5 h |
| 01.6 | [Branches and merging](06-branches-and-merging.md) | 2.5 h |
| 01.7 | [Merge conflicts](07-merge-conflicts.md) | 120 min |
| 01.8 | [Remote repositories: GitHub and GitLab](08-remotes-github-and-gitlab.md) | 2.5 h |
| 01.9 | [Teamwork: pull requests, code review and conventions](09-teamwork-pull-requests-and-conventions.md) | 3.5 h |
| 01.10 | [Project: your learning repository, and checkpoint](10-project-your-learning-repo-and-checkpoint.md) | 3 h |

## Projects in this phase

- **Practice repositories** (Lessons 3–9): small throw-away folders in `~/projects/git-playground` where you repeat every command.
- **`code/p01-git-practice`** (Lesson 10): your first Rust library crate, inside the learning repository `~/projects/rust`, with tests, a resolved conflict, a pull request with CI and a tagged release `v0.1.0`.
- **`notes/git-cheatsheet.md`** (Lesson 10): your own Git cheat sheet.

## How to study this phase

1. **Type every command yourself.** Git is learned with the fingers. Do not only read the examples; run them in `~/projects/git-playground` and compare your output with the lesson (your hashes and dates will differ).
2. **Run `git status` all the time.** It tells you where you are and what to do next. Make it a reflex before and after every other command.
3. **Break things on purpose in the practice folders.** Make a conflict, delete a branch, reset too far, and then recover. Mistakes in a throw-away folder are the cheapest way to lose the fear.
4. **Commit small and often,** even in these practice folders. Treat commits as save points.
5. **Keep a short cheat sheet** of the commands you actually use. You will write it in the final project, so start collecting during the phase.

---

[🏠 Start](../START-HERE.md) · [← 00.8 Study routine, notes in Markdown and checkpoint](../00-getting-started/08-study-routine-notes-and-checkpoint.md) · [01.1 What version control is →](01-what-is-version-control.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
