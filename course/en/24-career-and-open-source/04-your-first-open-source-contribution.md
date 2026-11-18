<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 24.3 Reading large codebases](03-reading-large-codebases.md) · [📋 Phase 24](README.md) · [24.5 Working with maintainers and communities →](05-working-with-maintainers-and-communities.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/24-career-and-open-source/04-your-first-open-source-contribution.md) <!-- nav -->

# Lesson 24.4 — Your first open-source contribution

> **Phase 24 — Career and open source** · Lesson 4 of 9 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What counts as an open-source contribution, and how to choose a project and a first issue.
- Which project files to read first: CONTRIBUTING, the code of conduct, the licence, and rules such as the DCO,
  a CLA or an AI policy.
- The complete fork workflow — fork, clone, `upstream`, branch, test first, signed-off commit, push, rebase —
  practised in a safe local laboratory.
- How to write a pull request description that a maintainer can review quickly.

## Before you start

- You finished [Lesson 24.3 — Reading large codebases](03-reading-large-codebases.md).
- You know remotes, pull requests, rebase and forks from
  [Lesson 01.8](../01-git-and-version-control/08-remotes-github-and-gitlab.md) and
  [Lesson 01.9 — Teamwork: pull requests, code review and conventions](../01-git-and-version-control/09-teamwork-pull-requests-and-conventions.md).
  Section 14 of Lesson 01.9 promised this lesson.
- Your Git identity is set up ([Lesson 01.2](../01-git-and-version-control/02-install-and-configure-git.md)).

## New words

| Word | Meaning |
|---|---|
| **open source** | Software whose source code is published under a licence that lets anyone use, study, change and share it. |
| **contribution** | Any useful work given to a project: code, tests, documentation, a clear bug report, answers to other users. |
| **maintainer** | A person who looks after a project: reviews and merges changes, makes releases, decides the direction. |
| **issue** | A numbered discussion in the project's tracker about one bug, idea or question, for example `#12`. |
| **fork** | Your own copy of someone else's repository on the hosting site. You push to your fork, never to the original. |
| **upstream** | The original repository you forked from. By convention your clone calls it `upstream` and your fork `origin`. |
| **good first issue** | A label that maintainers put on issues that are suitable for newcomers. Projects use other names too, for example `E-easy` or `help wanted`. |
| **CONTRIBUTING file** | `CONTRIBUTING.md`: the project's rules for contributors: how to build, test, format, write commits and send changes. |
| **code of conduct** | The rules for how people treat each other in the project's spaces, and how problems are reported. |
| **DCO** (Developer Certificate of Origin) | A short statement that you wrote the change (or have the right to submit it) under the project's licence. You agree to it by adding a `Signed-off-by:` line to each commit with `git commit -s`. |
| **CLA** (Contributor License Agreement) | A legal agreement, signed once (often online, through a bot), that gives the project the rights it needs to use your contributions. |
| **sign-off** | The line `Signed-off-by: Name <email>` at the end of a commit message. It is a statement, not a cryptographic signature. |
| **force-with-lease** | `git push --force-with-lease`: replace a remote branch with your rewritten history, but only if nobody else changed it since your last fetch. |
| **CI** (Continuous Integration) | The automatic build and test run that the project starts for every pull request (Lesson 09.12). |
| **ID** (identifier) | A number or name that identifies one thing, for example a thread ID in a panic message. |
| **AI** (Artificial Intelligence) | Tools that generate text or code. Some projects limit how such tools may be used in contributions (Lesson 24.3). |

## 1. What a contribution is

Code is only one kind of contribution, and often not the best first one. Maintainers are equally happy
with:

- a **bug report** that contains the exact steps, versions and output (Lesson 24.5 shows how to write one);
- a **documentation fix**: a wrong example, a missing sentence, an outdated command;
- a **test** for behaviour that has no test yet;
- **answers** to other users' questions in the issue tracker or chat;
- **trying a fix** that someone proposed and reporting whether it works for you.

Your first goal is not a large feature. It is **one small, merged change**, so that you learn the
project's process: how to build it, how it is tested, how review works. After that, each next change is
easier.

## 2. Choose a project and an issue

**Start from what you use.** You know ripgrep from Lesson 24.3, and you used many crates in this course:
`serde`, `clap`, `tokio`, `axum`, `sqlx`, `tracing` and others. Contributing is much easier when you already
understand what the project is for.

Check that the project is **alive** before you invest time:

| Signal | Where to look | Good sign |
|---|---|---|
| Recent activity | the commit history | commits in the last few weeks |
| Responsive maintainers | recent issues and pull requests | answers within days or a few weeks |
| Clear rules | `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md`, licence | they exist and are up to date |
| Working tests | the CI status of the main branch | green |

**Find an issue.** Filter the issue list by labels such as `good first issue`, `help wanted`, `E-easy` or
`E-mentor` (the `E-` labels are used in the rust-lang repositories). The newsletter *This Week in Rust* also has
a weekly "Call for Participation" section with tasks that maintainers chose for newcomers.

Before you start work on an issue:

1. **Read the whole thread.** The solution may already be discussed, or rejected.
2. **Check for an open pull request** that already fixes it, and for someone who said "I am working on this".
3. **Ask, briefly.** For example: "I would like to work on this. My plan is to add a check in `parse_pair` and a
   test for an empty key. Does that sound right?" In rust-lang repositories you can also write `@rustbot claim`
   in a comment to assign the issue to yourself.
4. **Build the project and run its tests before you change anything.** If they fail on the unchanged code,
   ask about it; your change is not the cause.

> ⚠️ **Common mistake:** Opening a large pull request without asking first. If the maintainers do not want
> that change, or want it done differently, the work is lost. A two-line question in the issue saves days.

## 3. Read the rules before the code

Every project has its own rules. Spend fifteen minutes on these files before your first commit:

- **`CONTRIBUTING.md`** — how to build and test, formatting and lint rules, the commit message style,
  whether to open an issue first, whether a CHANGELOG entry is needed.
- **`CODE_OF_CONDUCT.md`** — how people are expected to behave, and whom to contact if something goes
  wrong. Many projects use the *Contributor Covenant*; Rust projects often follow the Rust Code of Conduct.
- **The licence** — your contribution is published under it.
- **DCO or CLA.** A project that uses the **DCO** asks for a `Signed-off-by:` line in every commit; you add it
  with `git commit -s`. A project that uses a **CLA** asks you to sign an agreement once, usually through a
  link that a bot posts on your first pull request. Read what you sign. If you are employed, check whether your
  employment contract says anything about open-source work.
- **An AI policy**, if there is one. Lesson 24.3 showed ripgrep's: tools are allowed, but you must understand
  the change, explain it in your own words, and write your comments yourself.

> 🔒 **Security:** The sign-off (`-s`, lowercase) is a *statement*. It is different from a *cryptographic
> signature* of a commit (`-S`, uppercase), which proves with your key that the commit is yours (Lesson 01.9).
> A project may ask for one, the other, or both.

## 4. The fork workflow in a local laboratory

On GitHub or GitLab you would fork a real project. Here you build the same situation from folders on your
computer, so you can practise every step without bothering anyone. You play two people: **Max**, the
maintainer of a small library `kvparse`, and **Ada**, a first-time contributor (you).

```text
          upstream.git  (Max's original project)
            │      ▲
  "Fork"    │      │ pull request (on a real site)
            ▼      │
          fork.git  (Ada's copy on the server)
            │      ▲
   clone    │      │ git push
            ▼      │
          ada/  (Ada's working folder on her computer)
```

### Step 1 — Max publishes the project

```bash
mkdir -p ~/projects/git-playground/oss-lab
cd ~/projects/git-playground/oss-lab
git init -q --bare upstream.git
git clone -q upstream.git maintainer
cd maintainer
git config user.name "Max Maintainer"
git config user.email "max@example.com"
cargo init -q --lib --name kvparse
```

```text
warning: You appear to have cloned an empty repository.
```

The warning is expected: the "server" is still empty. Replace `src/lib.rs` with Max's first version:

```rust
//! Parses `key=value` pairs, as in simple configuration files.

/// Splits `"key=value"` into its two parts, without spaces around them.
/// Returns `None` if there is no `=`.
pub fn parse_pair(line: &str) -> Option<(&str, &str)> {
    let (key, value) = line.split_once('=')?;
    Some((key.trim(), value.trim()))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn splits_and_trims() {
        assert_eq!(parse_pair(" port = 8080 "), Some(("port", "8080")));
    }

    #[test]
    fn needs_an_equals_sign() {
        assert_eq!(parse_pair("port"), None);
    }
}
```

and create `CONTRIBUTING.md`:

```markdown
# Contributing to kvparse

- Open an issue before a large change.
- Every change needs a test. Run `cargo fmt`, `cargo clippy -- -D warnings`
  and `cargo test` before you push.
- Sign off every commit (`git commit -s`): this project uses the DCO.
```

Max tests, commits and publishes:

```bash
cargo test -q
git add .
git commit -q -m "Add parse_pair"
git push -q origin main
cd ..
```

```text

running 2 tests
..
test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s


running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

```

(The second "running 0 tests" block is for doc tests; the library has none.) A user then opens an issue. It is
invented for this lab, but it looks like a real one:

```text
#12  parse_pair accepts an empty key

parse_pair(" = 8080") returns Some(("", "8080")). A line without a key
is a mistake in the configuration file and should be rejected (None).
```

### Step 2 — Ada forks and clones

On a real site, the **Fork** button creates a copy of the repository under your account. Here a bare copy plays
that role. Then Ada clones **her fork**, sets her identity for this lab and adds the original as `upstream`:

```bash
git clone -q --bare upstream.git fork.git
git clone -q fork.git ada
cd ada
git config user.name "Ada Learner"
git config user.email "ada@example.com"
git remote add upstream ../upstream.git
git remote -v
```

```text
origin	/Users/ada/projects/git-playground/oss-lab/fork.git (fetch)
origin	/Users/ada/projects/git-playground/oss-lab/fork.git (push)
upstream	../upstream.git (fetch)
upstream	../upstream.git (push)
```

Your home folder appears instead of `/Users/ada`. The two names matter: **`origin`** is your fork, where you
may push; **`upstream`** is the original, which you only fetch from. (In a real project you would use your own
global identity from Lesson 01.2; the `git config` lines here only make the two lab people different.)

### Step 3 — A branch and a failing test

Never work on `main`. Create a branch named after the change, then write the test from the issue **first**
(Lesson 09.6). Add it inside `mod tests` in `src/lib.rs`:

```bash
git switch -c fix-empty-key
```

```rust
    #[test]
    fn rejects_an_empty_key() {
        assert_eq!(parse_pair(" = 8080"), None);
    }
```

```bash
cargo test
```

```text
running 3 tests
test tests::needs_an_equals_sign ... ok
test tests::splits_and_trims ... ok
test tests::rejects_an_empty_key ... FAILED

failures:

---- tests::rejects_an_empty_key stdout ----

thread 'tests::rejects_an_empty_key' (6636791) panicked at src/lib.rs:26:9:
assertion `left == right` failed
  left: Some(("", "8080"))
 right: None
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace


failures:
    tests::rejects_an_empty_key

test result: FAILED. 2 passed; 1 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s

error: test failed, to rerun pass `--lib`
```

The test reproduces the issue exactly: `left` is what the code returns, `right` is what the issue asks for.
(The number after the thread name is a thread ID, an identifier of the thread, and differs on every run.)

### Step 4 — The fix, and the project's checks

Change `parse_pair` and its documentation:

```rust
/// Splits `"key=value"` into its two parts, without spaces around them.
/// Returns `None` if there is no `=` or if the key is empty.
pub fn parse_pair(line: &str) -> Option<(&str, &str)> {
    let (key, value) = line.split_once('=')?;
    let key = key.trim();
    if key.is_empty() {
        return None;
    }
    Some((key, value.trim()))
}
```

Then run exactly the checks that `CONTRIBUTING.md` asks for:

```bash
cargo test
cargo fmt --check && cargo clippy -q -- -D warnings && echo "all checks passed"
```

```text
running 3 tests
test tests::needs_an_equals_sign ... ok
test tests::rejects_an_empty_key ... ok
test tests::splits_and_trims ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```text
all checks passed
```

### Step 5 — A signed-off commit, pushed to the fork

```bash
git add src/lib.rs
git commit -s -m "Reject empty keys in parse_pair" -m "A line like \"= 8080\" returned an empty key. Return None instead.

Fixes #12"
git log -1 --format='%h %s%n%n%b'
```

```text
[fix-empty-key 12cc175] Reject empty keys in parse_pair
 1 file changed, 11 insertions(+), 2 deletions(-)
12cc175 Reject empty keys in parse_pair

A line like "= 8080" returned an empty key. Return None instead.

Fixes #12

Signed-off-by: Ada Learner <ada@example.com>
```

**What happens:**

1. The first `-m` is the subject line, the second `-m` the body (Lesson 01.3). The body says *why*.
2. `Fixes #12` is a keyword that GitHub and GitLab understand: when the change is merged into the main branch,
   issue #12 is closed automatically.
3. `-s` added the `Signed-off-by:` line from your `user.name` and `user.email`, as the DCO requires.
4. The commit hashes (`12cc175` here) will be different on your computer.

Push the branch to **your fork**:

```bash
git push
```

```text
To /Users/ada/projects/git-playground/oss-lab/fork.git
 * [new branch]      fix-empty-key -> fix-empty-key
branch 'fix-empty-key' set up to track 'origin/fix-empty-key'.
```

The plain `git push` went to `origin` and created the branch there, because of the `push.autoSetupRemote` setting
from Lesson 01.2. On a real site you would now open a pull request from `your-account:fix-empty-key` to
`upstream:main`.

### Step 6 — Upstream moves on: rebase and update

While Ada waits for review, Max merges another change. Play Max for a moment:

```bash
cd ../maintainer
printf '# kvparse\n\nParses `key=value` lines.\n' > README.md
git add README.md
git commit -q -m "Add README"
git push -q
cd ../ada
```

Ada's branch is now based on an old `main`. Many projects ask you to **rebase** onto the newest upstream code,
so that the history stays straight and CI tests your change together with the latest code:

```bash
git fetch upstream
git rebase upstream/main
cargo test -q
git log --oneline --graph -3
```

```text
From ../upstream
 * [new branch]      main       -> upstream/main
Successfully rebased and updated refs/heads/fix-empty-key.
```

```text
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

```text
* ebfde23 Reject empty keys in parse_pair
* 47044c0 Add README
* a642456 Add parse_pair
```

The rebase gave Ada's commit a new hash (`ebfde23`), so the fork still has the old one. A normal push would be
rejected. Replace the branch on the fork, safely:

```bash
git push --force-with-lease
```

```text
To /Users/ada/projects/git-playground/oss-lab/fork.git
 + 12cc175...ebfde23 fix-empty-key -> fix-empty-key (forced update)
```

`--force-with-lease` overwrites the branch only if it still is where you last saw it. If someone (or you, from
another computer) had pushed to it in the meantime, the push would stop instead of destroying that work. Use it
only on **your own** branches in **your own** fork, never on `main` and never on a shared branch (Lesson 01.9,
"the golden rule of rebase").

## 5. The pull request description

The description is the first thing the maintainer reads. Keep it short and complete:

```markdown
Fixes #12.

## What
`parse_pair` now returns `None` when the key is empty, for example for `" = 8080"`.

## Why
An empty key is a mistake in the configuration file; accepting it hid the mistake.

## How I tested it
- Added the test `rejects_an_empty_key` (it failed before the fix).
- `cargo fmt --check`, `cargo clippy -- -D warnings` and `cargo test` pass locally.
```

A good first pull request is **small** (one issue, one idea), **tested**, **follows the rules** of
`CONTRIBUTING.md`, and does **not** touch unrelated code. If you notice another problem while working, open a
separate issue for it.

> 🧪 **Testing:** Always show that your test failed before the fix. A test that has never been red may test
> nothing (Lesson 05.6). Many reviewers check this first.

After you open it, CI runs. If it fails, read the log (Lesson 09.12), fix the problem on the same branch and push
again; the pull request updates itself. Then wait. Maintainers are often volunteers; Lesson 24.5 explains how to
handle the wait and the review.

## Common mistakes

**Your new branch tracks `upstream`.** If you create a branch directly from `upstream/main`, Git sets it to track
`upstream`, and a plain `git push` then refuses:

```bash
git switch -c try-tracking upstream/main
git push
```

```text
Switched to a new branch 'try-tracking'
branch 'try-tracking' set up to track 'upstream/main'.
fatal: The upstream branch of your current branch does not match
the name of your current branch.  To push to the upstream branch
on the remote, use

    git push upstream HEAD:main

To push to the branch of the same name on the remote, use

    git push upstream HEAD

To choose either option permanently, see push.default in 'git help config'.

To avoid automatically configuring an upstream branch when its name
won't match the local branch, see option 'simple' of branch.autoSetupMerge
in 'git help config'.
```

Do **not** follow the suggestions: both would push to the original project (on a real site you usually do not
have the rights anyway). Create such branches without tracking, `git switch -c <name> --no-track upstream/main`,
or push explicitly to your fork with `git push -u origin <name>`. Clean up this test branch with
`git switch fix-empty-key` and `git branch -D try-tracking`.

**Forgetting the sign-off.** Projects with a DCO check fail the pull request. Fix the last commit with
`git commit --amend -s --no-edit` and push with `--force-with-lease` (Exercise 2).

## Try it yourself

1. Build the laboratory from Section 4 and run every step yourself, including the rebase.
2. Write the pull request description for your change in `notes/oss-log.md`.
3. Choose one real project that you use. Check the four signals from Section 2 and read its `CONTRIBUTING.md`.
   Write down: DCO, CLA or neither; the required checks; the label for newcomer issues.
4. Find one issue with a newcomer label and write (in your notes, not on the site yet) the short comment you
   would post to ask whether you may work on it.

## Exercises

### Exercise 1 — Issue #13 (medium)

A second issue arrives: "`parse_pair("my port = 8080")` returns a key with a space inside. Keys must be single
words." In the lab, start a new branch `reject-spaces` from the newest upstream code (without tracking), write
the test first, fix the code, and commit. Spaces **around** the key must still be allowed.

<details>
<summary>Hint</summary>

`str::contains` accepts a function: `key.contains(char::is_whitespace)` is `true` if any character is a space,
tab or other whitespace. Trim first, then check.

</details>

<details>
<summary>Answer</summary>

```bash
git switch -c reject-spaces --no-track upstream/main
```

The function and the new test (in `src/lib.rs`; this branch starts from upstream, so it does not have Ada's
first fix yet):

```rust
/// Splits `"key=value"` into its two parts, without spaces around them.
/// Returns `None` if there is no `=` or if the key contains a space.
pub fn parse_pair(line: &str) -> Option<(&str, &str)> {
    let (key, value) = line.split_once('=')?;
    let key = key.trim();
    if key.contains(char::is_whitespace) {
        return None;
    }
    Some((key, value.trim()))
}

#[test]
fn rejects_a_key_with_spaces_inside() {
    assert_eq!(parse_pair("my port = 8080"), None);
    assert_eq!(parse_pair("  port = 8080"), Some(("port", "8080")));
}
```

```text
running 3 tests
test tests::needs_an_equals_sign ... ok
test tests::splits_and_trims ... ok
test tests::rejects_a_key_with_spaces_inside ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Then `git add src/lib.rs` and `git commit -m "Reject keys with spaces inside"` (without `-s`, on purpose: Exercise 2
repairs that). Why it works: `trim` removes
the spaces around the key first, so only spaces *inside* it are rejected. Two separate branches mean two separate
pull requests, each easy to review; the maintainer can merge them in any order.

</details>

### Exercise 2 — The forgotten sign-off (easy)

You committed on `reject-spaces` without `-s`. Add the sign-off to the last commit and push the branch to your
fork. Show the final commit message.

<details>
<summary>Hint</summary>

`--amend` rewrites the last commit; `--no-edit` keeps its message. The branch has no upstream on the fork yet,
so the first push creates it.

</details>

<details>
<summary>Answer</summary>

```bash
git commit --amend -s --no-edit
git push
git log -1 --format=%B
```

```text
[reject-spaces 330f9c9] Reject keys with spaces inside
 Date: Wed Nov 18 09:20:03 2026 +0000
 1 file changed, 12 insertions(+), 2 deletions(-)
To /Users/ada/projects/git-playground/oss-lab/fork.git
 * [new branch]      reject-spaces -> reject-spaces
branch 'reject-spaces' set up to track 'origin/reject-spaces'.
Reject keys with spaces inside

Signed-off-by: Ada Learner <ada@example.com>
```

Why it works: `--amend` makes a new commit with the same changes and the extra line. Because the branch was
created with `--no-track`, the plain `git push` goes to `origin`, your fork. If you had already pushed the
unsigned commit, you would need `git push --force-with-lease` instead.

</details>

### Exercise 3 — Is this project a good first choice? (easy, no code)

Take one project from your `Cargo.toml` files of this course. Fill in the table from Section 2 with what you
find, then decide: is it a good first project for you? Give two reasons.

<details>
<summary>Hint</summary>

The crate's page on crates.io links to its repository. Look at the dates of the last commits and of the
last answered issues.

</details>

<details>
<summary>Answer</summary>

There is no single answer. A good answer fills all four rows with facts (dates, file names, the CI status) and
gives reasons such as "I used it in Phase 16, the maintainers answer within a week, and there are five open
`good first issue` issues". "Too big for a first contribution" is also a valid conclusion; then choose a smaller
project that you use.

</details>

## Quiz

1. What are `origin` and `upstream` in the fork workflow?

<details>
<summary>Answer</summary>

`origin` is your fork, where you push your branches. `upstream` is the original project, which you fetch from to
stay current. You never push to `upstream`.

</details>

2. What is the difference between a DCO and a CLA?

<details>
<summary>Answer</summary>

The DCO is a statement you make in every commit with a `Signed-off-by:` line (`git commit -s`). A CLA is a legal
agreement you sign once, usually online, that gives the project rights to your contributions.

</details>

3. Why should you ask in the issue before you start a bigger change?

<details>
<summary>Answer</summary>

The maintainers may not want the change, may want it done another way, or someone may already be working on
it. A short question avoids wasted work on both sides.

</details>

4. Why `git push --force-with-lease` and not `git push --force` after a rebase?

<details>
<summary>Answer</summary>

`--force-with-lease` refuses to overwrite the remote branch if it changed since your last fetch, so you cannot
destroy work you have not seen. `--force` overwrites in any case.

</details>

5. What does `Fixes #12` in a commit message or pull request do?

<details>
<summary>Answer</summary>

It links the change to issue 12, and on GitHub and GitLab it closes the issue automatically when the change is
merged into the default branch.

</details>

## Summary

- Contributions include bug reports, documentation, tests and answers, not only code. Aim for one small, merged
  change first.
- Choose a project you use and that is alive; pick an issue with a newcomer label, read the thread and ask before
  you start.
- Read `CONTRIBUTING.md`, the code of conduct, the licence, and rules such as the DCO, a CLA or an AI policy.
- The fork workflow: fork, clone your fork (`origin`), add `upstream`, branch, test first, fix, run the checks,
  commit with `-s`, push to `origin`, open the pull request.
- Keep up with `git fetch upstream` and `git rebase upstream/main`, then `git push --force-with-lease` on your
  own branch.
- A good pull request is small, tested, explained and limited to one idea.

## Sources and further reading

- [Open Source Guides — How to Contribute to Open Source](https://opensource.guide/how-to-contribute/) — a
  friendly overview of the whole process.
- [GitHub Docs — Contributing to a project](https://docs.github.com/en/get-started/exploring-projects-on-github/contributing-to-a-project) and
  [GitLab Docs — Forking workflow](https://docs.gitlab.com/user/project/repository/forking_workflow/) — the
  fork workflow on the two big sites.
- [GitHub Docs — Linking a pull request to an issue](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/linking-a-pull-request-to-an-issue) —
  keywords such as `Fixes #12`.
- [Developer Certificate of Origin](https://developercertificate.org/) — the full text you agree to with `git commit -s`.
- [Contributor Covenant](https://www.contributor-covenant.org/) and the [Rust Code of Conduct](https://www.rust-lang.org/policies/code-of-conduct) —
  two widely used codes of conduct.
- [Rust Forge — Issue assignment](https://forge.rust-lang.org/triagebot/issue-assignment.html) and
  [Rust Compiler Development Guide — Contributing](https://rustc-dev-guide.rust-lang.org/contributing.html) — how
  contributions work in the rust-lang repositories, including `@rustbot claim`.
- [This Week in Rust](https://this-week-in-rust.org/) — the weekly "Call for Participation" lists tasks for new
  contributors.

---

[🏠 Start](../START-HERE.md) · [← 24.3 Reading large codebases](03-reading-large-codebases.md) · [📋 Phase 24](README.md) · [24.5 Working with maintainers and communities →](05-working-with-maintainers-and-communities.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
