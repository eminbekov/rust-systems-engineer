<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 01.8 Remote repositories: GitHub and GitLab](08-remotes-github-and-gitlab.md) · [📋 Phase 01](README.md) · [01.10 Project: your learning repository, and checkpoint →](10-project-your-learning-repo-and-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md) <!-- nav -->

# Lesson 01.9 — Teamwork: pull requests, code review and conventions

> **Phase 01 — Git and version control** · Lesson 9 of 10 · ⏱ about 3.5 hours (2 sittings)

## What you will learn

- The pull request workflow that almost every team uses, step by step.
- How to write a good pull request and how to give and receive a code review.
- How to keep your branch up to date with `rebase`, how to clean up commits with interactive rebase, and the one rule you must never break.
- Commit message conventions (Conventional Commits), version numbers (SemVer) and tags.
- What continuous integration (CI) is, and how to check a Rust project automatically on every push.
- Useful extras: `cherry-pick`, Git hooks, protected branches and the files a team repository usually contains.

## Before you start

- You finished [Lesson 01.8 — Remote repositories: GitHub and GitLab](08-remotes-github-and-gitlab.md).
- You know branches, merging and conflicts ([Lesson 01.6](06-branches-and-merging.md), [Lesson 01.7](07-merge-conflicts.md)).
- A GitHub account is useful but not required to read this lesson.

## New words

| Word | Meaning |
|---|---|
| **PR** (Pull Request) | A request on GitHub to merge your branch into another (usually `main`), with a place for review and discussion. |
| **MR** (Merge Request) | The same thing on GitLab. |
| **code review** | Other people read your changes and comment before they are merged. |
| **reviewer** / **author** | The person who checks the change / the person who wrote it. |
| **LGTM** ("Looks Good To Me") | A common short approval in a review. |
| **nit** | A tiny, optional comment in a review (a "nitpick"), for example about naming. |
| **rebase** | To take your commits and replay them on top of another base commit. This creates new commits. |
| **squash** | To combine several commits into one. |
| **cherry-pick** | To copy one commit from another branch onto the current branch. |
| **fork** | Your own copy of someone else's repository on GitHub/GitLab. |
| **upstream** (in forks) | The original repository you forked from. |
| **protected branch** | A branch (for example `main`) that the server does not allow to be pushed to directly or to be force-pushed. |
| **Conventional Commits** | A convention for commit messages: `type(scope): description`. |
| **SemVer** (Semantic Versioning) | A rule for version numbers: `MAJOR.MINOR.PATCH`. |
| **tag** | A permanent name for one commit, used for releases such as `v1.2.0`. |
| **CI** (Continuous Integration) | A service that automatically builds and tests your code on every push. |
| **CD** (Continuous Delivery / Deployment) | Automatically delivering or deploying the result of CI. You only meet the idea here. |
| **YAML** (YAML Ain't Markup Language) | A text format for configuration based on indentation. CI files are written in it. |
| **workflow** / **job** / **step** | In GitHub Actions: a whole automation file / a group of steps run on one machine / one command or action. |
| **runner** | The machine (often a virtual machine, VM) that runs CI jobs. |
| **VM** (Virtual Machine) | A computer simulated by software. |
| **hook** | A script that Git runs automatically at certain moments, for example before a commit. |
| **Clippy** | Rust's linter: a tool that finds suspicious or unidiomatic code. |
| **CLI** (Command-Line Interface) | A way to use a program by typing commands. |

## 1. Why pull requests?

Imagine ten developers pushing straight to `main`. Sooner or later someone pushes code that does not compile, and everybody is blocked. A **pull request** adds a checkpoint between "my work" and "the shared branch":

- somebody else **reads** the change (code review): fewer bugs, shared knowledge;
- **automatic checks** (CI) run on the proposed change;
- there is a **discussion** and a record of why the change was made;
- `main` stays **always working**.

(The name says: "please *pull* my changes into your branch".) On GitLab it is a *merge request*; the idea is identical.

## 2. The workflow, step by step

This is the "GitHub flow". It is simple, and it works for small and large teams.

```text
 main:   ●────────────●───────────────●
          \          /               ▲
 branch:   ●──●──●──●  ── review ── merge ── delete branch
           (commits)   (open PR)
```

| # | Step | Commands |
|---|---|---|
| 1 | Start from an up-to-date `main` | `git switch main` then `git pull` |
| 2 | Create a branch for **one** task | `git switch -c feature/short-name` |
| 3 | Commit small, meaningful changes | `git add …` and `git commit` |
| 4 | Push the branch | `git push` (first time `git push -u origin feature/short-name`; with our config the `-u` is automatic) |
| 5 | Open a pull request on the website (or `gh pr create`) | |
| 6 | CI runs; reviewers comment; you push more commits | `git push` again; the PR updates by itself |
| 7 | The PR is approved and **merged** (by the button) | |
| 8 | Clean up: delete the branch, update your `main` | `git switch main`, `git pull`, `git branch -d feature/short-name` |

With the GitHub CLI the middle steps look like this (these commands need a real GitHub repository):

```bash
git push -u origin feature/short-name
gh pr create --fill
gh pr list
gh pr checkout 12
gh pr merge --squash --delete-branch
```

**What happens:** `gh pr create --fill` opens a PR with the title and description taken from your commits. `gh pr list` lists open PRs. `gh pr checkout 12` downloads the branch of PR number 12 so you can test a colleague's work. `gh pr merge --squash --delete-branch` merges using the squash strategy (section 5) and deletes the branch.

> 💡 **Tip:** open a **draft** PR early (`gh pr create --draft`) for work in progress. Mark it "ready for review" when you are done.

## 3. Writing a good pull request

A reviewer is a busy person. Make their work easy.

1. **One purpose.** One PR = one change. If a PR does two things, make two PRs.
2. **Small.** Aim for under about 400 changed lines. Large PRs get skimmed, not reviewed.
3. **A clear title,** in the same style as a commit subject: "Fix crash when the config file is empty".
4. **A useful description.** A template that works everywhere:

```text
## What
Short summary of the change.

## Why
The problem it solves, or a link to the issue (for example: Closes #12).

## How
Anything non-obvious about the approach.

## Testing
How you checked it: commands run, tests added.
```

5. **Review your own PR first.** Open the "Files changed" tab and read it like a stranger. You will catch debug prints and stray files.
6. **Green checks.** Do not ask for review while CI is red.

In a PR description, `Closes #12` (or `Fixes #12`) links the PR to issue 12 and closes the issue automatically when the PR is merged.

A team can put a ready template in `.github/pull_request_template.md`; GitHub fills the description with it for every new PR.

## 4. Code review

### When you are the author

- A review is about the **code**, not about you. Thank people for their time.
- Answer every comment, even with "Done". If you disagree, explain calmly with facts.
- Push fixes as **new commits** during review, so reviewers can see what changed. (You may squash at the end.)
- If a comment is unclear, ask a question.

### When you are the reviewer

Read in this order: *does it solve the problem?* → *is it correct and tested?* → *is it readable?* → *style details*. Questions to ask:

| Area | Ask |
|---|---|
| Correctness | Does it do what the PR says? What about empty input, errors, large input? |
| Tests | Is there a test that would fail without this change? |
| Design | Is it simple? Does it fit the existing code? |
| Readability | Are names clear? Would a new teammate understand it? |
| Security | Is any input trusted? Any secret in the code? |
| Rust | Does `cargo fmt --check` pass? `cargo clippy` without warnings? Any `unwrap()` that can panic in real use? |

Kind and clear comment styles:

- `nit: rename x to count (optional)` — a small optional suggestion.
- `question: why is this loop needed?` — you are asking, not demanding.
- `suggestion: this could use …` — an idea.
- `blocking: this panics on empty input` — must be fixed before merge.
- `LGTM` or *Approve* when you are satisfied.

📏 **Convention:** comment on the code, not the person ("this function is hard to follow", not "you write confusing code"). Praise good work too.

## 5. How a PR is merged: three strategies

When you press *Merge*, GitHub and GitLab offer these options:

| Strategy | What `main` gets | History shape |
|---|---|---|
| **Merge commit** | all the branch commits **plus** one merge commit | branches and merges visible (graph with forks) |
| **Squash and merge** | **one** new commit that contains the whole PR | a straight line; one commit per PR; messy "wip" commits disappear |
| **Rebase and merge** | the branch commits replayed one by one on top of `main`, no merge commit | a straight line; every commit kept |

```text
 merge commit:         squash:              rebase and merge:
 ●─●───────●  main      ●─●─S  main          ●─●─a'─b'─c'  main
  \       /              (S = a+b+c)
   a──b──c
```

Teams choose one and use it everywhere. **Squash and merge** is very popular for small PRs because `main` gets one tidy commit per PR (its message should follow the conventions below). After a squash merge, `git branch -d` may say the branch is "not fully merged" because the squashed commit is a different commit; after you checked that the PR is merged, `git branch -D` is fine.

## 6. Keeping your branch up to date: `rebase`

While your PR is open, `main` moves on. You must integrate it. There are two ways: **merge** `main` into your branch (you know it), or **rebase** your branch onto `main`.

A rebase takes your commits, temporarily sets them aside, moves your branch to the newest `main`, and **replays** your commits on top. Practise on a small repository:

```bash
cd ~/projects/git-playground
mkdir rebase-demo
cd rebase-demo
git init
echo "one" > a.txt
git add a.txt
git commit -m "A: start"
git switch -c feature
echo "feature work" > feature.txt
git add feature.txt
git commit -m "F1: add feature file"
echo "more feature work" >> feature.txt
git commit -am "F2: extend feature file"
git switch main
echo "two" >> a.txt
git commit -am "B: update a.txt on main"
echo "three" >> a.txt
git commit -am "C: update a.txt again"
git log --oneline --graph --all
```

```text
* 5b4cd87 F2: extend feature file
* 5a1c6b0 F1: add feature file
| * 78f9dc8 C: update a.txt again
| * 867371c B: update a.txt on main
|/
* 42d4976 A: start
```

Now rebase `feature` onto `main`:

```bash
git switch feature
git rebase main
git log --oneline --graph --all
```

```text
Rebasing (1/2)Rebasing (2/2)Successfully rebased and updated refs/heads/feature.
* 40e6f13 F2: extend feature file
* 4c229ba F1: add feature file
* 78f9dc8 C: update a.txt again
* 867371c B: update a.txt on main
* 42d4976 A: start
```

```text
before:  A ── B ── C            (main)         after:  A ── B ── C ── F1' ── F2'   (feature)
          \                                                    (main is at C)
           F1 ── F2   (feature)
```

**What happens:** the history is now a **straight line**. Look at the hashes: `F1` was `5a1c6b0` and is now `4c229ba`. The commits `F1'` and `F2'` are **new commits** with the same changes and messages but a different parent, so they have different hashes. Merging `feature` into `main` is now a clean fast-forward:

```bash
git switch main
git merge feature
```

```text
Updating 78f9dc8..40e6f13
Fast-forward
 feature.txt | 2 ++
 1 file changed, 2 insertions(+)
 create mode 100644 feature.txt
```

### The golden rule of rebase

> ⚠️ **Common mistake:** **never rebase commits that other people already have** (commits you pushed to a shared branch such as `main`). Rebase creates new commits with new hashes. If somebody has built on the old ones, you create duplicates and conflicts for everyone. Rebase **only your own, not yet shared work**, or a PR branch that you alone use.

If you rebase a branch that you **already pushed** (for example your own PR branch), the remote will refuse a normal push, because the histories differ. For your own branch only, use the careful force:

```bash
git push --force-with-lease
```

`--force-with-lease` overwrites the remote branch **only if it is still what you last saw**. If a teammate pushed to your branch in the meantime, it refuses:

```text
To /Users/ada/projects/git-playground/fwl/remote.git
 ! [rejected]        feature -> feature (stale info)
error: failed to push some refs to '/Users/ada/projects/git-playground/fwl/remote.git'
```

(Note: if you run `git fetch` first, Git thinks you have seen the new commits, and the force goes through and overwrites them. So **look** at what you are about to overwrite.) Plain `--force` has no such protection. Never use it on `main`.

### Conflicts during a rebase

A rebase replays commits one at a time, so conflicts can appear in the middle. The recipe is almost the same as in [Lesson 01.7](07-merge-conflicts.md):

```text
Auto-merging doc.txt
CONFLICT (content): Merge conflict in doc.txt
error: could not apply 4d71444... Change title on topic
hint: Resolve all conflicts manually, mark them as resolved with
hint: "git add/rm <conflicted_files>", then run "git rebase --continue".
hint: You can instead skip this commit: run "git rebase --skip".
hint: To abort and get back to the state before "git rebase", run "git rebase --abort".
```

1. Open the file, remove the markers, make the final text.
2. `git add <file>`.
3. `git rebase --continue` (Git may open an editor for the message; save and close).
4. Repeat if another commit conflicts.
5. To give up: `git rebase --abort` returns everything to how it was before.

Remember from Lesson 01.7: during a rebase, "ours" is the branch you are rebasing **onto**, and "theirs" is **your own commit** being replayed. It feels backwards, so check the result.

## 7. Cleaning up commits: interactive rebase

Before you open a PR, you may want to tidy up messy commits ("wip", "fix typo", "oops"). `git rebase -i` (interactive) lets you edit the list of commits. Make three small commits:

```bash
git switch -c wip
echo "line 1" >> notes.txt
git add notes.txt
git commit -m "wip 1"
echo "line 2" >> notes.txt
git commit -am "wip 2"
echo "line 3" >> notes.txt
git commit -am "wip 3"
git log --oneline -4
git rebase -i HEAD~3
```

```text
c91c95e wip 3
bef0851 wip 2
b66cc6b wip 1
40e6f13 F2: extend feature file
```

`git rebase -i HEAD~3` opens your editor (nano) with a list, oldest first:

```text
pick b66cc6b # wip 1
pick bef0851 # wip 2
pick c91c95e # wip 3

# Rebase 40e6f13..c91c95e onto 40e6f13 (3 commands)
#
# Commands:
# p, pick <commit> = use commit
# r, reword <commit> = use commit, but edit the commit message
# e, edit <commit> = use commit, but stop for amending
# s, squash <commit> = use commit, but meld into previous commit
# f, fixup [-C | -c] <commit> = like "squash" but keep only the previous
#                    commit's log message, unless -C is used, in which case
#                    keep only this commit's message; -c is same as -C but
#                    opens the editor
# x, exec <command> = run command (the rest of the line) using shell
# b, break = stop here (continue rebase later with 'git rebase --continue')
# d, drop <commit> = remove commit
…
```

Change the word at the start of a line to tell Git what to do with that commit:

| Command | Effect |
|---|---|
| `pick` | keep the commit as it is |
| `reword` | keep the commit, edit its message |
| `squash` | meld into the previous commit, and let you combine the messages |
| `fixup` | meld into the previous commit and **discard** this message |
| `drop` | delete the commit |
| (reorder lines) | change the order of commits |

To merge `wip 2` and `wip 3` into `wip 1`, change the file to:

```text
pick b66cc6b # wip 1
fixup bef0851 # wip 2
fixup c91c95e # wip 3
```

Save and close the editor (in `nano`: `Ctrl+O`, `Enter`, `Ctrl+X`). Git replays the list:

```text
Rebasing (2/3)Rebasing (3/3)Successfully rebased and updated refs/heads/wip.
```

```bash
git log --oneline -3
```

```text
9317beb wip 1
40e6f13 F2: extend feature file
4c229ba F1: add feature file
```

**What happens:** the three commits became one (its message is `wip 1`; use `reword` if you want a better one). Same rule as before: only rewrite commits that are not shared.

> 📏 **Convention:** a typical PR has a handful of clean, logical commits, or it is squashed on merge. Do not spend hours polishing; clear and small is enough.

## 8. `cherry-pick`: take one commit

Sometimes you need exactly one commit from another branch, for example a bug fix that was made on a side branch while `main` must get it now.

```bash
git switch -c hotfix
echo "fix" > fix.txt
git add fix.txt
git commit -m "Fix the crash"
git switch main
git cherry-pick hotfix
git log --oneline -3
```

```text
[main 59835de] Fix the crash
 Date: Tue Nov 17 16:58:53 2026 +0500
 1 file changed, 1 insertion(+)
 create mode 100644 fix.txt
59835de Fix the crash
40e6f13 F2: extend feature file
4c229ba F1: add feature file
```

**What happens:** `git cherry-pick hotfix` copied the changes of the **tip commit** of `hotfix` onto `main` as a new commit (new hash). You can give a commit hash instead of a branch name. If it conflicts, resolve it like a merge and run `git cherry-pick --continue`. Use it sparingly; it duplicates commits.

## 9. Commit message conventions

You met the basic rules in [Lesson 01.3](03-your-first-repository-and-commits.md): short imperative subject, blank line, body that explains *why*. Many teams add the **Conventional Commits** format, which machines and humans can both read:

```text
<type>(<scope>): <description>

[optional body]

[optional footer, e.g. "Closes #12" or "BREAKING CHANGE: …"]
```

| Type | Use for |
|---|---|
| `feat` | a new feature (appears in the changelog; usually a MINOR version) |
| `fix` | a bug fix (PATCH) |
| `docs` | documentation only |
| `style` | formatting only, no code meaning change |
| `refactor` | code change that neither fixes a bug nor adds a feature |
| `perf` | a performance improvement |
| `test` | adding or fixing tests |
| `build` | build system, dependencies (`Cargo.toml`) |
| `ci` | CI configuration |
| `chore` | other maintenance |

Examples:

```text
feat(parser): support quoted strings
fix: avoid panic on empty input
docs(readme): add build instructions
refactor(config)!: rename Settings to Config

BREAKING CHANGE: the public type Settings no longer exists.
```

A `!` after the type (or a `BREAKING CHANGE:` footer) marks a change that breaks users of your code (a MAJOR version). The `(scope)` is optional and names the part of the project.

Why bother? Tools can generate a changelog and the next version number from such commits, and `git log --oneline` becomes easy to scan. If your team does not use the format, do not force it; use the basic rules. Whatever you choose, **be consistent**.

## 10. Versions and tags: SemVer

A **release** needs a version number. The Rust world uses **Semantic Versioning** (SemVer): `MAJOR.MINOR.PATCH`, for example `1.4.2`.

| Part | Increase it when you… | Example |
|---|---|---|
| MAJOR | make a change that **breaks** users' code | `1.4.2` → `2.0.0` |
| MINOR | add a feature in a backward-compatible way | `1.4.2` → `1.5.0` |
| PATCH | fix a bug in a backward-compatible way | `1.4.2` → `1.4.3` |

Before `1.0.0` (versions like `0.3.1`) the API (Application Programming Interface: the public functions and types that other code uses) is considered unstable, and a MINOR change may break things. In Rust the version lives in `Cargo.toml` (`version = "0.1.0"`). Cargo itself follows SemVer for dependencies (you will study this in Lesson 05.8).

A **tag** gives a commit a permanent name:

```bash
git tag v0.1.0
git tag -a v0.2.0 -m "Second release"
git tag
git show v0.2.0 --stat
git log --oneline --decorate -3
git tag -d v0.1.0
```

```text
v0.1.0
v0.2.0
tag v0.2.0
Tagger: Ada Learner <ada@example.com>
Date:   Tue Nov 17 16:58:53 2026 +0500

Second release

commit 59835de1f808395ae8d48978f9c43ed161315d51
Author: Ada Learner <ada@example.com>
Date:   Tue Nov 17 16:58:53 2026 +0500

    Fix the crash

 fix.txt | 1 +
 1 file changed, 1 insertion(+)
59835de (HEAD -> main, tag: v0.2.0, tag: v0.1.0) Fix the crash
40e6f13 (feature) F2: extend feature file
4c229ba F1: add feature file
Deleted tag 'v0.1.0' (was 59835de)
```

**What happens:**

1. `git tag v0.1.0` makes a **lightweight** tag: just a name for the current commit.
2. `git tag -a v0.2.0 -m "…"` makes an **annotated** tag: a real object with a tagger, date and message. Use annotated tags for releases.
3. `git tag` lists tags; `git show` displays one; `git tag -d` deletes one (locally).

Tags are **not** pushed automatically:

```bash
git push origin v0.2.0
git push origin --tags
```

The first sends one tag, the second all tags. On GitHub a pushed tag can become a **Release** page with notes and downloads. By convention, tag names start with `v`: `v1.2.0`.

## 11. Protecting `main` and the team files

Teams do not rely on discipline alone. In *Settings → Branches* (GitHub) a repository admin can add a **branch protection rule** for `main`:

- require a pull request (no direct pushes);
- require a number of approvals;
- require the CI checks to pass;
- forbid force-pushes and deletion.

Typical files in a team repository:

| File | Purpose |
|---|---|
| `README.md` | what the project is, how to build and use it |
| `CONTRIBUTING.md` | how to propose changes (branches, commit style, how to run tests) |
| `LICENSE` | what others may do with the code |
| `CHANGELOG.md` | what changed in each version (see the "Keep a Changelog" format) |
| `.gitignore` | files Git must not track |
| `.github/pull_request_template.md` | the default PR description |
| `.github/CODEOWNERS` | who must review changes to which files |
| `.github/workflows/*.yml` | GitHub Actions (CI) |

## 12. Continuous integration

**CI** means a robot builds and tests every change. The result appears next to the PR as a green tick or a red cross. For a Rust project, three checks are the standard:

| Check | Command | Fails when |
|---|---|---|
| Formatting | `cargo fmt --check` | the code is not formatted the standard way |
| Lints | `cargo clippy --all-targets -- -D warnings` | Clippy finds a problem (`-D warnings` turns warnings into errors) |
| Tests | `cargo test` | any test fails |

Run them on your own computer first. Here is what they look like for a small library (a project made with `cargo new ci-demo --lib`):

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
```

All three pass silently or with `ok`. Now see them fail. Badly formatted code:

```text
Diff in /Users/ada/projects/git-playground/ci-demo/src/lib.rs:1:
-pub fn add(a: i32,b: i32) -> i32 {
-        a+b
+pub fn add(a: i32, b: i32) -> i32 {
+    a + b
 }
 
 #[cfg(test)]
```

(Fix it with `cargo fmt`, which rewrites the files.) A lint problem (an unnecessary `return`):

```text
error: unneeded `return` statement
 --> src/lib.rs:2:5
  |
2 |     return a + b;
  |     ^^^^^^^^^^^^
  |
  = help: for further information visit https://rust-lang.github.io/rust-clippy/rust-1.100.0/index.html#needless_return
  = note: `-D clippy::needless-return` implied by `-D warnings`
  = help: to override `-D warnings` add `#[allow(clippy::needless_return)]`
help: remove `return`
  |
2 -     return a + b;
2 +     a + b
  |

error: could not compile `ci-demo` (lib) due to 1 previous error
```

(You do not need to understand the Rust code yet; the point is how the tools report problems. `fn`, `return` and tests come in Phase 02.)

### A GitHub Actions workflow

Put this file at `.github/workflows/ci.yml` in your repository:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install stable --profile minimal --component rustfmt,clippy
      - run: cargo fmt --check
      - run: cargo clippy --all-targets -- -D warnings
      - run: cargo test
```

**What happens:**

1. `name: CI` is the label shown on GitHub.
2. `on:` says when to run: on every push to `main`, and on every pull request.
3. `jobs:` lists jobs; here one job called `check`.
4. `runs-on: ubuntu-latest` picks the runner: a fresh Linux virtual machine that GitHub gives you.
5. `steps:` run in order on that machine. `uses: actions/checkout@v7` is a ready-made action that downloads your repository into the machine. Each `run:` line is a shell command. If a command fails, the job is red.
6. The `rustup … --component rustfmt,clippy` line makes sure the formatter and Clippy are installed.

YAML uses **indentation with spaces** (never tabs) to show structure; one wrong space breaks the file.

The same idea on GitLab is a file `.gitlab-ci.yml` at the top of the repository:

```yaml
stages: [check]

rust-checks:
  stage: check
  image: rust:1
  script:
    - rustup component add rustfmt clippy
    - cargo fmt --check
    - cargo clippy --all-targets -- -D warnings
    - cargo test
```

(`image: rust:1` is the official Rust container image; you will study containers much later.)

> 🧪 **Testing:** CI is the safety net for tests: it runs them for everyone, in a clean machine, every time. In your PR the red cross means "do not merge yet".

## 13. Git hooks: checks on your own computer

A **hook** is a script in `.git/hooks/` that Git runs at a certain moment. The `pre-commit` hook runs before each commit and can stop it:

```bash
cat > .git/hooks/pre-commit <<'EOF'
#!/bin/sh
# Stop the commit when the code is not formatted.
if ! cargo fmt --check > /dev/null 2>&1; then
    echo "pre-commit: run 'cargo fmt' before committing"
    exit 1
fi
EOF
chmod +x .git/hooks/pre-commit
```

Now a commit with badly formatted code is refused:

```text
pre-commit: run 'cargo fmt' before committing
```

(In a Rust project, run this in the project folder. The first line `#!/bin/sh` says which program runs the script, and `chmod +x` makes the file executable.)

You can skip hooks in an emergency with `git commit --no-verify`. Remember that `.git/hooks/` is **not** part of the repository: your teammates do not get your hooks. That is why the **real** guard is CI, and hooks are only a convenience for you.

## 14. Working on someone else's project: forks

You cannot push to a project you do not own. For open source, you:

1. **Fork** the repository (a button on GitHub): you get your own copy under your account.
2. **Clone** your fork. Add the original as a second remote called `upstream`: `git remote add upstream git@github.com:owner/project.git`.
3. Create a branch, commit, `git push` to **your** fork (`origin`).
4. Open a pull request from your fork's branch to the original repository.
5. Keep your fork current: `git fetch upstream` and `git rebase upstream/main` (or merge).

You will practise open-source contributions in a later phase; the ideas are the same as above.

> 🔒 **Security:** commits can claim any author name. To prove that a commit is really yours, you can **sign** commits with your SSH (Secure Shell) key; GitHub then shows a "Verified" badge. Look at GitHub's page "About commit signature verification" in the sources if you are curious. It is optional for now.

## Common mistakes

> ⚠️ **Common mistake:** a PR with 40 files and three unrelated topics. Split it. Small PRs are reviewed faster and break less.

> ⚠️ **Common mistake:** pushing directly to `main` "just this once". That is how `main` gets broken. Use a branch and a PR, even when you work alone, to build the habit.

> ⚠️ **Common mistake:** rebasing a shared branch, then force-pushing. You rewrite other people's history. Only rebase your own private branches.

> ⚠️ **Common mistake:** `git add .` before opening a PR, and a `.env` or a debug file appears in "Files changed". Always read the file list of your PR before asking for review.

> ⚠️ **Common mistake:** ignoring a red CI. A red check is a bug report. Open the log, find the failing command, reproduce it locally with the same command, fix it.

> ⚠️ **Common mistake:** a vague commit like `update` in a PR that is merged with "merge commit". On `main` it stays forever. Use squash merge, or clean up first.

> ⚠️ **Common mistake:** inconsistent YAML indentation in CI files. If the workflow does not appear on GitHub, check the indentation first.

## Try it yourself

1. In `rebase-demo` (from section 6) make a new branch `docs`, add a commit, move `main` forward by one commit, then rebase `docs` onto `main` and merge it with a fast-forward.
2. Make four commits `wip a`, `wip b`, `wip c`, `wip d`. Use `git rebase -i HEAD~4` and `fixup` to leave one commit. Then reword it.
3. Tag the last commit `v0.1.0` (annotated) and list the tag with `git show v0.1.0`.
4. In a Rust project of your own (for example `hello-rust` from Lesson 01.3), run `cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo test`. Break the formatting on purpose, read the output of the first command, then run `cargo fmt` and check again.
5. Write the file `.github/workflows/ci.yml` from section 12 in that project. You will push it in the project of [Lesson 01.10](10-project-your-learning-repo-and-checkpoint.md).

## Exercises

### Exercise 1 — Conventional commit messages (easy)

Write a Conventional Commits message for each change: (a) you added a `--verbose` option to a tool; (b) you fixed a crash on empty input; (c) you updated the README; (d) you renamed a public function, which breaks other people's code.

<details>
<summary>Hint</summary>

Types: `feat`, `fix`, `docs`, `refactor`. A breaking change gets `!`.

</details>

<details>
<summary>Answer</summary>

```text
feat: add --verbose option
fix: avoid crash on empty input
docs(readme): update build instructions
refactor!: rename parse_line to parse

BREAKING CHANGE: parse_line was renamed to parse.
```

Why it works: each message states the type and a short imperative description. The `!` and the footer warn that users must change their code.

</details>

### Exercise 2 — Write a PR description (easy)

You fixed an issue (#12) where the program panics when the input file is empty. You added a test. Write the PR title and description with the template from section 3.

<details>
<summary>Hint</summary>

Use the four headings What / Why / How / Testing, and `Closes #12`.

</details>

<details>
<summary>Answer</summary>

```text
Title: Fix panic on empty input file

## What
The program now prints a clear message and exits with code 1 when the input file is empty.

## Why
Closes #12. Before this change the program panicked on an empty file.

## How
Added an early check after reading the file.

## Testing
Added the test `empty_file_is_rejected`. `cargo test` passes.
```

Why it works: the reviewer sees the purpose, the link to the issue, the approach and the proof in one place, and GitHub closes issue 12 on merge.

</details>

### Exercise 3 — Rebase a branch (medium)

Create a repository with `main` (two commits) and a branch `feature` that started from the first commit and has one commit that touches a **different** file. Rebase `feature` onto `main`, show the linear history, and merge into `main` as a fast-forward.

<details>
<summary>Hint</summary>

Create `feature` after the first commit, then go back to `main` and add one more commit. Then `git switch feature`, `git rebase main`, `git switch main`, `git merge feature`.

</details>

<details>
<summary>Answer</summary>

```bash
git init -b main rebase-ex
cd rebase-ex
echo "one" > a.txt
git add a.txt
git commit -m "A"
git switch -c feature
echo "f" > f.txt
git add f.txt
git commit -m "F"
git switch main
echo "two" >> a.txt
git commit -am "B"
git switch feature
git rebase main
git log --oneline --graph --all
git switch main
git merge feature
```

The graph after the rebase is a single straight line `F`, `B`, `A`; the final merge says `Fast-forward`.

Why it works: the rebase replays `F` on top of `B`, so `feature` contains all of `main`. Then `main` can simply move forward.

</details>

### Exercise 4 — Squash three commits with `fixup` (medium)

On a branch, make commits `wip 1`, `wip 2`, `wip 3` that each add one line to `notes.txt`. Combine them into one commit with interactive rebase. Show the log before and after.

<details>
<summary>Hint</summary>

`git rebase -i HEAD~3`, then change the second and third `pick` to `fixup`. To do it without an editor: set `GIT_SEQUENCE_EDITOR` to a command that edits the list.

</details>

<details>
<summary>Answer</summary>

```bash
git switch -c wip
echo "line 1" >> notes.txt
git add notes.txt
git commit -m "wip 1"
echo "line 2" >> notes.txt
git commit -am "wip 2"
echo "line 3" >> notes.txt
git commit -am "wip 3"
git log --oneline -3
git rebase -i HEAD~3
git log --oneline -3
```

In the editor change the list to `pick` / `fixup` / `fixup`, save and close. Before: three `wip` commits. After: one commit `wip 1`.

Without an editor (macOS): `GIT_SEQUENCE_EDITOR="sed -i '' -e '2,3s/^pick/fixup/'" git rebase -i HEAD~3`.

Why it works: `fixup` melds a commit into the previous one and drops its message. The three changes remain, in one commit.

</details>

### Exercise 5 — Run CI on your computer (hard)

In a project made with `cargo new ci-exercise --lib`: replace `src/lib.rs` with the code below, run the three CI commands, read the failures, fix them, and commit when all three pass. Also create the file `.github/workflows/ci.yml` and commit it in a separate commit with a Conventional Commits message.

```rust
pub fn add(a: i32,b: i32) -> i32 {
    return a+b;
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_two_numbers() {
        assert_eq!(add(2, 3), 5);
    }
}
```

<details>
<summary>Hint</summary>

`cargo fmt --check` fails first (spaces). Run `cargo fmt` to fix it. Then `cargo clippy` complains about `return`; remove the word `return` and the semicolon.

</details>

<details>
<summary>Answer</summary>

```bash
cargo new ci-exercise --lib
cd ci-exercise
# put the code above into src/lib.rs
cargo fmt --check
cargo fmt
cargo clippy --all-targets -- -D warnings
```

`cargo fmt --check` prints a diff and fails; after `cargo fmt` the file is formatted. Then Clippy reports `unneeded return statement`. Fix `src/lib.rs`:

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn adds_two_numbers() {
        assert_eq!(add(2, 3), 5);
    }
}
```

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test
git add .
git commit -m "feat: add the add function with a test"
mkdir -p .github/workflows
# put the workflow from section 12 into .github/workflows/ci.yml
git add .github
git commit -m "ci: check format, lints and tests"
```

`cargo test` ends with `test result: ok. 1 passed`.

Why it works: the three commands are the same ones that CI will run, so if they pass locally they pass on GitHub. `cargo fmt` fixes formatting automatically, but Clippy's advice must be applied by you.

</details>

## Quiz

1. What is the purpose of a pull request?

<details>
<summary>Answer</summary>

To propose merging a branch into another (usually `main`), with a place for review, discussion and automatic checks, so that the shared branch stays working.

</details>

2. What is the difference between "squash and merge" and "rebase and merge"?

<details>
<summary>Answer</summary>

Squash combines all commits of the PR into one new commit on `main`. Rebase-and-merge replays each commit separately on top of `main`. Both give a straight history.

</details>

3. What is the golden rule of rebase?

<details>
<summary>Answer</summary>

Never rebase commits that other people already have (shared history). Rebase only your own, unshared work.

</details>

4. Why is `git push --force-with-lease` safer than `--force`?

<details>
<summary>Answer</summary>

It refuses to overwrite the remote branch if it changed since you last saw it, so you do not destroy a teammate's new commits by accident. (It cannot protect you after you ran `git fetch` and ignored what came in.)

</details>

5. A version goes from `1.4.2` to `1.5.0`. What kind of change was it?

<details>
<summary>Answer</summary>

A MINOR change: a new feature that is backward-compatible.

</details>

6. What are the three standard CI checks for a Rust project?

<details>
<summary>Answer</summary>

`cargo fmt --check` (formatting), `cargo clippy --all-targets -- -D warnings` (lints) and `cargo test` (tests).

</details>

7. Why is a `pre-commit` hook not enough to guarantee quality in a team?

<details>
<summary>Answer</summary>

Hooks live in `.git/hooks/`, which is not part of the repository. Teammates do not receive them and can skip them with `--no-verify`. CI runs for everyone and cannot be skipped.

</details>

## Summary

- The pull request workflow: branch → commits → push → PR → review and CI → merge → clean up.
- Good PRs are small, have one purpose, a clear title and a useful description; review the code, not the person.
- Merge strategies: merge commit, squash, rebase-and-merge. Teams pick one.
- `git rebase` replays your commits on a new base. Never rebase shared history; for your own pushed branch use `--force-with-lease`.
- `git rebase -i` cleans commits (`reword`, `squash`, `fixup`, `drop`); `git cherry-pick` copies one commit.
- Conventional Commits (`feat:`, `fix:` …), SemVer (`MAJOR.MINOR.PATCH`) and annotated tags (`v1.2.0`) make history and releases readable.
- CI runs `cargo fmt --check`, `cargo clippy -- -D warnings` and `cargo test` on every push; protect `main` so that it requires them.
- Hooks help locally, but CI is the real guard.

## Sources and further reading

- [GitHub Docs — About pull requests](https://docs.github.com/en/pull-requests) — pull requests, reviews and merging on GitHub.
- [GitLab Docs — Merge requests](https://docs.gitlab.com/user/project/merge_requests/) — the same ideas on GitLab.
- [Pro Git — 3.6 Rebasing](https://git-scm.com/book/en/v2/Git-Branching-Rebasing) — the official explanation, with the "perils of rebasing".
- [Pro Git — 7.6 Rewriting History](https://git-scm.com/book/en/v2/Git-Tools-Rewriting-History) — amend, interactive rebase, squash, reorder.
- [Pro Git — 2.6 Tagging](https://git-scm.com/book/en/v2/Git-Basics-Tagging) — lightweight and annotated tags.
- [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/) — the full specification.
- [Semantic Versioning 2.0.0](https://semver.org/) — the rules for `MAJOR.MINOR.PATCH`.
- [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) — a format for `CHANGELOG.md`.
- [Google Engineering Practices — Code Review](https://google.github.io/eng-practices/review/) — how to review and how to be reviewed.
- [GitHub Docs — Understanding GitHub Actions](https://docs.github.com/en/actions) — workflows, jobs and steps.
- [GitLab Docs — GitLab CI/CD](https://docs.gitlab.com/ci/) — pipelines and `.gitlab-ci.yml`.
- [GitHub Docs — About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) — branch protection rules.
- [GitHub Docs — About commit signature verification](https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification) — signed commits.
- [The Cargo Book — SemVer compatibility](https://doc.rust-lang.org/cargo/reference/semver.html) — what counts as a breaking change in Rust.

---

[🏠 Start](../START-HERE.md) · [← 01.8 Remote repositories: GitHub and GitLab](08-remotes-github-and-gitlab.md) · [📋 Phase 01](README.md) · [01.10 Project: your learning repository, and checkpoint →](10-project-your-learning-repo-and-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
