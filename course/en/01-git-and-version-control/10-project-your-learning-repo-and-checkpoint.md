<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 01.9 Teamwork: pull requests, code review and conventions](09-teamwork-pull-requests-and-conventions.md) · [📋 Phase 01](README.md) · [Phase 02 — Programming basics in Rust →](../02-programming-basics/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/01-git-and-version-control/10-project-your-learning-repo-and-checkpoint.md) <!-- nav -->

# Lesson 01.10 — Project: your learning repository, and checkpoint

> **Phase 01 — Git and version control** · Lesson 10 of 10 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How to put everything you do in this course under one clean Git repository, with a good layout and good habits.
- How to practise the full professional loop on your own project: branches, a conflict, a remote, a pull request with automatic checks, and a tagged release.
- How to avoid the one Cargo and Git trap that creates a hidden repository inside your repository.
- How to check yourself on the whole of Phase 01, and where to review if something is not clear.

## Before you start

- You finished Lessons [01.1](01-what-is-version-control.md) to [01.9](09-teamwork-pull-requests-and-conventions.md). Your Git is configured ([Lesson 01.2](02-install-and-configure-git.md)).
- For milestone M5 you need a GitHub or GitLab account with an SSH (Secure Shell) key ([Lesson 01.8](08-remotes-github-and-gitlab.md)). If you cannot create one now, M5 and M6 have an offline fallback.
- Your course folder is `~/projects/rust`. From now on **every project in this course lives in `~/projects/rust/code/pNN-<name>`** (NN is the phase number) and is committed to this repository.

## New words

| Word | Meaning |
|---|---|
| **learning repository** | The single Git repository for all your code and notes in this course. |
| **milestone** (M1, M2, …) | A step of a project with a clear goal and a way to check it. |
| **acceptance test** | A command (or a few) whose expected output proves that a milestone is finished. The "Done when" lines are acceptance tests. |
| **nested repository** | A repository inside another repository's folder. It is almost always a mistake. |
| **changelog** | A file (`CHANGELOG.md`) that lists what changed in each version, for humans. |
| **release** | A named, tagged version of the project, for example `v0.1.0`. |
| **`core.hooksPath`** | A Git setting that tells Git to look for hooks in another folder, which can be committed to the repository. |
| **checkpoint** | A review lesson at the end of a phase: a quiz and practical tasks. |

## 1. What this project is

In this phase you learned Git with small practice folders. Now you build the real thing: **one repository for the whole course**. From Phase 02 on, every project goes into it, and your Git history becomes a diary of your learning. The same workflow, with different names, is what you will use at work.

You will also write your first Rust *library crate* (a Rust package that other code can call) as the thing to put under version control. You do not need to understand the Rust code yet: you copy it, and Phase 02 explains it. The point of this project is **Git**, not Rust.

## 2. The trap: `cargo new` can create a hidden repository

Read this before you start. When you run `cargo new` **outside** a Git repository, Cargo helpfully runs `git init` for you. Inside an existing repository, it should not. But there is a catch: Cargo decides by looking at the **parent folder** of the new project, and if that parent folder does not exist yet, Cargo does not see your repository. Compare:

```bash
# the folder code/ does NOT exist yet
cargo new code/p01-git-practice --lib
find . -maxdepth 4 -name .git
```

```text
./code/p01-git-practice/.git
./.git
```

A second `.git` appeared inside the project: a **nested repository**. The outer repository now sees `code/p01-git-practice` as one strange object, and your history does not contain its files. The correct way is to create the parent folder first:

```bash
mkdir -p code
cargo new code/p01-git-practice --lib
find . -maxdepth 4 -name .git
```

```text
./.git
```

(Only one `.git`.) In this case Cargo also does not create a `.gitignore` inside the project, so the **root** `.gitignore` must ignore `target/` (without a leading slash, so that it matches in every folder). Make `mkdir -p code` a habit before every `cargo new code/…` for the rest of the course.

## Common mistakes

> ⚠️ **Common mistake:** a nested `.git` folder (see above). Check with `find . -maxdepth 4 -name .git`. Only `./.git` should appear. If a project folder has its own `.git` and no history you want to keep, remove that inner folder (`rm -rf` on **that exact path**, after you checked it with `git -C <folder> log --oneline`).

> ⚠️ **Common mistake:** the root `.gitignore` says `/target`, which matches only a top-level `target`. Use `target/`.

> ⚠️ **Common mistake:** creating the remote repository on GitHub with a README and licence, and then failing to push. Create it **empty**.

> ⚠️ **Common mistake:** committing `Cargo.lock` late or never. For programs and for this learning repository, commit it.

> ⚠️ **Common mistake:** working directly on `main` "because it is only me". The goal here is to practise the team workflow, so use branches for each milestone.

> 🔒 **Security:** the repository must never contain a password, token, `.env` file or private key. Check `git ls-files` and `.gitignore` before your first push.

## Try it yourself

A health check. Before you begin, check your tools. All commands must work:

```bash
git --version
git config --global user.name
git config --global user.email
git config --global init.defaultBranch
cargo --version
ssh -T git@github.com
```

The first five print values (version, your name, your email, `main`, a Cargo version). The last one prints the "successfully authenticated" message (GitHub) or `Permission denied (publickey)` (fix it with [Lesson 01.8](08-remotes-github-and-gitlab.md), or use the offline fallback).

## The project

### What you will build

One Git repository, `~/projects/rust`, that contains:

- a clear layout and a README;
- your own **Git cheat sheet** in `notes/`;
- a first Rust library crate, `code/p01-git-practice`, with tests;
- a history with feature branches, merges and one resolved conflict;
- a private remote on GitHub or GitLab (or an offline bare repository);
- a pull request with automatic checks (CI, Continuous Integration), merged with squash;
- a tagged release `v0.1.0` with a changelog.

### Contract

**Layout** (the final result):

```text
~/projects/rust/
├── .gitignore                  ← target/, .idea/, .DS_Store, *.swp, .env
├── README.md                   ← what this repository is, how to run the tests, your commit style
├── CHANGELOG.md                ← written in M7
├── notes/
│   └── git-cheatsheet.md       ← your own cheat sheet, at least 15 commands
├── code/
│   └── p01-git-practice/       ← library crate (cargo new --lib)
│       ├── Cargo.toml
│       ├── Cargo.lock
│       └── src/lib.rs
└── .github/workflows/ci.yml    ← or .gitlab-ci.yml (written in M6)
```

If your folder also holds other material (for example the course text, if it is stored there), decide for each item: **track it**, **ignore it** (add it to `.gitignore`) or **move it away**. Write your decision in the README.

**Rules:**

1. The default branch is `main`. Work for each milestone on a branch named `type/short-name` (`feature/…`, `docs/…`, `ci/…`, `chore/…`).
2. Commit messages: a short imperative subject (about 50 characters, no final period). Optionally use the Conventional Commits prefixes (`feat:`, `docs:`, `ci:`). Choose one style, write it in the README and use it everywhere.
3. One logical change per commit. At least **10 commits** in total by the end.
4. Never commit `target/`, `.idea/`, `.DS_Store`, `.env` or any secret.
5. Only **one** `.git` folder in the whole tree.
6. Before every commit, run `git status` and `git diff --staged`.

**Library contract** for `code/p01-git-practice` (the code is given; Phase 02 explains it):

| Function | Behaviour |
|---|---|
| `add(a: i32, b: i32) -> i32` | returns `a + b` |
| `subtract(a: i32, b: i32) -> i32` | returns `a - b` |
| `multiply(a: i32, b: i32) -> i32` | returns `a * b` |

Each function has one test. The commands `cargo test`, `cargo fmt --check` and `cargo clippy --all-targets -- -D warnings` must all succeed.

**Final acceptance** (the whole project):

```bash
cd ~/projects/rust
git status -sb                       # first line: ## main...origin/main
git log --oneline | wc -l            # 10 or more
git log --merges --oneline | wc -l   # 2 or more
git ls-files | grep -c 'target/'     # 0
find . -maxdepth 4 -name .git        # only ./.git
git tag -n1                          # v0.1.0 and your message
cargo test --manifest-path code/p01-git-practice/Cargo.toml   # 3 passed
```

### Step-by-step plan

You will not find a full solution here. Each milestone has a goal, guidance, a hint (hidden), and **Done when** commands with the expected output. Commit at the end of every milestone, on its branch.

#### M1 — Create the repository

**Goal:** `~/projects/rust` is a Git repository with a good `.gitignore`, a README and the notes folder, and one first commit on `main`.

**Guidance:**

1. Open the folder and look at what is already there: `cd ~/projects/rust && ls -a`. Check that it is not already a repository (`git status` should say `fatal: not a git repository`) and that no sub-folder has its own `.git` (`find . -maxdepth 3 -name .git`). If it already is a repository (for example because you published the course together with your own work), skip `git init` in step 2, and instead of creating `.gitignore` and `README.md`, check that the existing ones meet the contract.
2. Run `git init`. (`init.defaultBranch` makes the branch `main`.)
3. Create `.gitignore` with these lines: `target/`, `.idea/`, `.DS_Store`, `*.swp`, `.env`. Add any other folder from your decision (see the contract).
4. Create `README.md` with a title, one paragraph, and a "Commit style" line.
5. Create `notes/git-cheatsheet.md` with just a title for now.
6. Read `git status`, stage the files **by name**, check `git diff --staged`, and commit: `Start the learning repository`.

<details>
<summary>Hint</summary>

Do not use `git add .` before you have read `git status`: it could pick up the course text or an old project. Stage `.gitignore README.md notes` explicitly. If `find` shows an inner `.git` folder, deal with it first (see "Common mistakes").

</details>

**Done when:**

```bash
git status -sb
git log --oneline | wc -l
find . -maxdepth 4 -name .git
```

```text
## main
1
./.git
```

(`wc -l` may print spaces before the number. If other untracked folders exist, `git status -sb` lists them below the first line; that is fine as long as you decided what to do with them.)

#### M2 — The first Rust crate, without a hidden repository

**Goal:** the crate `code/p01-git-practice` is created and committed as part of **your** repository.

**Guidance:**

1. `git switch -c feature/crate`.
2. Create the parent folder **first**: `mkdir -p code`. Then `cargo new code/p01-git-practice --lib`.
3. Check that no nested repository appeared (the `find` command above).
4. Replace `code/p01-git-practice/src/lib.rs` with this code (you will learn all of it in Phase 02):

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

5. Run `cargo test` in the crate folder (this also creates `Cargo.lock`). Stage and commit: `Add p01-git-practice with an add function`.
6. Merge the branch into `main` (a fast-forward is fine) and delete it.

<details>
<summary>Hint</summary>

If `find` shows `./code/p01-git-practice/.git`, you forgot `mkdir -p code`. Delete the inner `.git` (`rm -rf code/p01-git-practice/.git`), then continue. Make sure `Cargo.lock` is staged.

</details>

**Done when:**

```bash
find . -maxdepth 4 -name .git
git ls-files code
cd code/p01-git-practice && cargo test && cd ../..
git ls-files | grep -c 'target/'
```

```text
./.git
code/p01-git-practice/Cargo.lock
code/p01-git-practice/Cargo.toml
code/p01-git-practice/src/lib.rs
```

and `cargo test` ends with `test result: ok. 1 passed`, and the last command prints `0`.

#### M3 — Two branches, one conflict

**Goal:** add `subtract` and `multiply` on two **separate** branches that both start from the same commit, merge them into `main`, and resolve the conflict that this creates.

**Guidance:**

1. From `main`, create `feature/subtract`. Add a function `subtract` (`a - b`) **above** the `#[cfg(test)]` line, and a test `subtracts_two_numbers` (`assert_eq!(subtract(5, 3), 2);`). Commit: `Add subtract with a test`.
2. Go back to `main` and create `feature/multiply` **from `main`** (not from `feature/subtract`). Add `multiply` (`a * b`) in the same place, and the test `multiplies_two_numbers` (`assert_eq!(multiply(4, 3), 12);`). Commit: `Add multiply with a test`.
3. Switch to `main`. Merge `feature/subtract` with `git merge --no-ff feature/subtract` (give a good message). Delete the branch.
4. Merge `feature/multiply`. Git reports a conflict in `src/lib.rs`. Read the markers (the base section from `zdiff3` helps). Decide the final text: **both functions and both tests must remain**, and the file must keep its structure.
5. `cargo fmt`, `cargo test`, `git diff --check`. Then `git add` and `git commit --no-edit`.
6. Delete `feature/multiply`.

<details>
<summary>Hint</summary>

The conflict looks strange, because both branches added code in the same place and the closing `}` lines are shared. Do not try to understand it line by line: write the final file by hand, with `add`, `subtract`, `multiply` and three tests, and compare it to what you want. The test `cargo test` must show `3 passed`. If you get lost: `git merge --abort` and try again.

</details>

**Done when:**

```bash
git grep -n '<<<<<<<' ; echo "exit=$?"
git log --merges --oneline
cd code/p01-git-practice && cargo test 2>&1 | grep 'test result' | head -1 ; cd ../..
```

```text
exit=1
… Merge branch 'feature/multiply'
… Merge branch 'feature/subtract'
test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

(`exit=1` means `git grep` found no conflict markers. The hashes in front of the two merge lines are yours.)

#### M4 — Your Git cheat sheet and README

**Goal:** the notes and the README describe the repository and what you learned, in your own words.

**Guidance:**

1. `git switch -c docs/cheatsheet`.
2. In `notes/git-cheatsheet.md` write **at least 15 commands** you learned in this phase, each on a line that starts with ``- `git `` and has a short explanation, grouped by topic (setup, snapshots, history, undo, branches, remotes, teamwork). Use only what you understand.
3. In `README.md` describe: what the repository is, the layout, how to run the tests (`cargo test` in a crate folder), and your commit style.
4. Commit in two commits, one per file (`docs: add Git cheat sheet`, `docs: describe the repository`), merge with a **fast-forward** or a merge commit as you prefer, and delete the branch.

<details>
<summary>Hint</summary>

Write the cheat sheet while you look at your terminal history (`history | grep git`), not by copying. Your own words are what make it useful later.

</details>

**Done when:**

```bash
grep -c '^- `git' notes/git-cheatsheet.md
git log --oneline | wc -l
```

The first command prints `15` or more; the second prints `8` or more at this point (the exact number depends on your choices).

#### M5 — The remote

**Goal:** your repository exists on a remote and `main` is pushed.

**Guidance:**

1. On GitHub (or GitLab) create an **empty private** repository, for example `learning-rust`. Do not add a README, `.gitignore` or licence.
2. Copy the SSH address. In your repository: `git remote add origin git@github.com:YOUR-NAME/learning-rust.git`.
3. `git push -u origin main`. If you use GitHub CLI (Command-Line Interface: GitHub's own terminal tool, the `gh` command): `gh repo create learning-rust --private --source=. --remote=origin --push` does steps 1–3.
4. Push your tags later (M7). Look at the repository on the website.

*Offline fallback:* create a bare repository **outside** your learning repository, for example `git init --bare ~/projects/rust-origin.git`, and use `git remote add origin ~/projects/rust-origin.git`. Everything else in M5–M7 is the same.

<details>
<summary>Hint</summary>

If `git push` says `Permission denied (publickey)`, run `ssh -T git@github.com` to test the key ([Lesson 01.8](08-remotes-github-and-gitlab.md)). If it says the remote contains work you do not have, you created the remote repository with a README: delete it and create an empty one.

</details>

**Done when:**

```bash
git remote -v
git status -sb
git ls-remote origin
```

```text
origin	git@github.com:YOUR-NAME/learning-rust.git (fetch)
origin	git@github.com:YOUR-NAME/learning-rust.git (push)
## main...origin/main
<hash>	HEAD
<hash>	refs/heads/main
```

(The two hashes in the last command are identical and equal to your latest commit on `main`.)

#### M6 — A pull request with CI

**Goal:** you add the CI workflow through a **pull request**, review it yourself, and merge it with squash.

**Guidance:**

1. `git switch -c ci/add-checks`.
2. Create `.github/workflows/ci.yml` (GitHub). Because the crate is inside `code/`, tell the job where to run:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  check:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: code/p01-git-practice
    steps:
      - uses: actions/checkout@v7
      - run: rustup toolchain install stable --profile minimal --component rustfmt,clippy
      - run: cargo fmt --check
      - run: cargo clippy --all-targets -- -D warnings
      - run: cargo test
```

   On GitLab use this `.gitlab-ci.yml` instead:

```yaml
stages: [check]

rust-checks:
  stage: check
  image: rust:1
  before_script:
    - cd code/p01-git-practice
  script:
    - rustup component add rustfmt clippy
    - cargo fmt --check
    - cargo clippy --all-targets -- -D warnings
    - cargo test
```

3. First run the three commands locally in the crate folder. Commit: `ci: check format, lints and tests`. Push the branch.
4. Open a pull request (`gh pr create --fill`, or the website). Write a description with What / Why / Testing. Open the "Files changed" tab and review your own change. Wait for the green check.
5. Merge with **Squash and merge**. Delete the remote branch (the button, or `git push origin --delete ci/add-checks`).
6. Update your local `main` (`git switch main`, `git pull`), and delete the local branch (`git branch -d ci/add-checks`, or `-D` if Git says "not fully merged" after the squash).

*Offline fallback:* merge locally with `git merge --squash ci/add-checks` and `git commit -m "ci: check format, lints and tests (#1)"`, and push `main`. Note in your journal what a real PR would have added (review, checks).

<details>
<summary>Hint</summary>

If the workflow does not appear in the *Actions* tab, check the folder name (`.github/workflows`, with the dot) and the indentation (spaces, never tabs). If CI is red, open the log, find the failing command and reproduce it locally.

</details>

**Done when:**

```bash
git log --oneline -1
git branch -a
ls .github/workflows
```

```text
<hash> ci: check format, lints and tests (#1)
* main
  remotes/origin/main
ci.yml
```

(The number in the PR title comes from your service, for example `(#1)`.) On the website, the *Actions* tab shows a green check for the latest run on `main`.

#### M7 — Changelog and release

**Goal:** the first release `v0.1.0` is documented, tagged and pushed.

**Guidance:**

1. `git switch -c docs/changelog`. Create `CHANGELOG.md` in the "Keep a Changelog" style: a heading `## [0.1.0] - 2026-11-17` (use today's date) and bullet points under `### Added` (the crate, the three functions, the CI, the notes).
2. Commit (`docs: add changelog for 0.1.0`), open a PR or merge locally as in M6, and update `main`.
3. Create an **annotated tag** on the latest `main` commit: `git tag -a v0.1.0 -m "First release of the learning repository"`.
4. Push the tag: `git push origin v0.1.0`.
5. (GitHub) Open *Releases → Draft a new release*, choose the tag `v0.1.0`, and paste the changelog section. Optional.

<details>
<summary>Hint</summary>

`git tag` without `-a` makes a lightweight tag; use `-a` and `-m`. Tags are not pushed by `git push` alone. If you tagged the wrong commit, delete the tag (`git tag -d v0.1.0`; and on the remote `git push origin --delete v0.1.0`) and make it again.

</details>

**Done when:**

```bash
git tag -n1
git ls-remote --tags origin
```

```text
v0.1.0          First release of the learning repository
<hash>	refs/tags/v0.1.0
<hash>	refs/tags/v0.1.0^{}
```

(Two lines appear for an annotated tag: the tag object and the commit it points to, marked `^{}`.)

Now run the **final acceptance** commands from the contract. All of them must match.

### Stretch goals

1. **Versioned hook.** Create `scripts/hooks/pre-commit` that runs `cargo fmt --check` in the crate, make it executable and commit it. Tell Git to use that folder: `git config core.hooksPath scripts/hooks`. Now the hook is part of the repository (teammates run `git config core.hooksPath scripts/hooks` once).
2. **Protect `main`.** On the website add a branch protection rule: require a pull request and a passing CI check. Try to push directly to `main` and read the error.
3. **PR template.** Add `.github/pull_request_template.md` with the What / Why / How / Testing headings.
4. **Detective.** Add a deliberately broken commit in the middle of a new branch, then use `git bisect run cargo test` to find it.
5. **Signed commits.** Follow GitHub's guide to sign commits with your SSH key and see the "Verified" badge.
6. **Second remote.** Add a GitLab remote called `gitlab` and push `main` to both (`git push gitlab main`).

## Self-test quiz

Answer without looking at the lessons, then open the answers.

1. What problem does version control solve, and what does the word "distributed" mean for Git?

<details>
<summary>Answer</summary>

It records every version of your files with a message and an author, so you can compare, find who changed what and why, go back, and work in parallel. "Distributed" means every developer has a full copy of the whole history, so most commands work offline and any copy can restore the project.

</details>

2. What is the difference between Git and GitHub?

<details>
<summary>Answer</summary>

Git is the program that records history on your computer. GitHub is a website and service that hosts Git repositories and adds pull requests, issues and CI. GitLab is a similar service.

</details>

3. Name the three places where a change can live before it is pushed, in order.

<details>
<summary>Answer</summary>

The working tree (your edited files), the staging area (what `git add` collected for the next commit) and the repository (the commits, in `.git`).

</details>

4. You edited `a.rs` and `b.rs` but want to commit only `a.rs`. Which commands?

<details>
<summary>Answer</summary>

`git add a.rs` and `git commit -m "…"`. Only the staged file goes into the commit; `b.rs` stays modified.

</details>

5. What does `git status -s` show for a new, untracked file, and for a modified file that is not staged?

<details>
<summary>Answer</summary>

`?? file` for the untracked file and ` M file` (a space, then `M`) for the modified, unstaged file.

</details>

6. Why do you add `target/` to `.gitignore` in a Rust project? Does `.gitignore` help for a file you already committed?

<details>
<summary>Answer</summary>

`target/` holds build output that can be recreated and can be huge; it must not be in the history. `.gitignore` does not affect files that are already tracked: use `git rm --cached <file>` and commit.

</details>

7. What do `HEAD`, `HEAD~1` and `HEAD~3` mean?

<details>
<summary>Answer</summary>

`HEAD` is the commit you are on now; `HEAD~1` is its parent; `HEAD~3` is three steps back.

</details>

8. How do you see exactly what the next commit will contain?

<details>
<summary>Answer</summary>

`git diff --staged` (and `git status`).

</details>

9. Which of these can destroy uncommitted work: `git restore <file>`, `git restore --staged <file>`, `git reset --hard`, `git stash`, `git clean -f`?

<details>
<summary>Answer</summary>

`git restore <file>`, `git reset --hard` and `git clean -f`. `git restore --staged` and `git stash` are safe.

</details>

10. When do you use `git revert` instead of `git reset`?

<details>
<summary>Answer</summary>

When the commit is already shared (pushed). `revert` adds a new commit that undoes the old one, and does not rewrite history.

</details>

11. You ran `git reset --hard HEAD~3` by mistake. How do you get the three commits back?

<details>
<summary>Answer</summary>

Run `git reflog`, find the entry from before the reset (for example `HEAD@{1}`), and run `git reset --hard 'HEAD@{1}'` (or use the hash).

</details>

12. What is a branch, and what does `git switch -c fix/typo` do?

<details>
<summary>Answer</summary>

A branch is a small movable pointer to a commit. `git switch -c fix/typo` creates a new branch at the current commit and switches to it.

</details>

13. When is a merge a fast-forward, and when does Git create a merge commit?

<details>
<summary>Answer</summary>

Fast-forward: the current branch has no commits that the other lacks, so Git only moves the pointer. Merge commit: both branches have new commits, so Git joins them with a commit that has two parents (or you force it with `--no-ff`).

</details>

14. In a conflict block, what do `<<<<<<< HEAD`, `|||||||`, `=======` and `>>>>>>> other` mark? What are the steps to finish?

<details>
<summary>Answer</summary>

Your version (ours), the common-ancestor version (base, with `zdiff3`), the divider, and the end with the other branch's version (theirs). Edit the file to its final text without markers, `git add` it, and `git commit` (or `git merge --continue`). `git merge --abort` cancels.

</details>

15. What is the difference between `git fetch` and `git pull`? Why was a push rejected with "fetch first"?

<details>
<summary>Answer</summary>

`fetch` downloads and updates `origin/*` without touching your work; `pull` is `fetch` plus a merge. The push was rejected because the remote had commits you did not have; you must integrate them first.

</details>

16. Which SSH key goes to GitHub, and what happens if the other one leaks?

<details>
<summary>Answer</summary>

The public key (`.pub`). If the private key leaks, anyone can act as you: remove the public key from the services and create a new key pair.

</details>

17. Why is `git push --force` dangerous, and what is safer on your own branch?

<details>
<summary>Answer</summary>

It overwrites the remote history and can destroy other people's commits. `--force-with-lease` refuses if the remote branch changed since you last saw it. Never force-push to `main`.

</details>

18. What is the golden rule of `rebase`?

<details>
<summary>Answer</summary>

Do not rebase commits that other people already have; rebase only your own unshared work.

</details>

19. A version changes from `0.4.2` to `0.5.0`. What does SemVer say about the change? What is a good commit message for a bug fix in Conventional Commits style?

<details>
<summary>Answer</summary>

A MINOR change (new backward-compatible feature; before `1.0.0` MINOR may also break). A bug fix message: `fix: avoid panic on empty input`.

</details>

20. Which three commands does CI usually run for a Rust project?

<details>
<summary>Answer</summary>

`cargo fmt --check`, `cargo clippy --all-targets -- -D warnings` and `cargo test`.

</details>

## Practical tasks

Do these in a new folder `~/projects/git-playground/checkpoint`. Each task starts with a setup block.

### Task 1 — Rescue

Setup:

```bash
mkdir -p ~/projects/git-playground/checkpoint
cd ~/projects/git-playground/checkpoint
git init -b main rescue-task
cd rescue-task
for n in 1 2 3 4; do echo "line $n" >> story.txt; git add story.txt; git commit -m "Add line $n"; done
git switch -c idea
echo "special idea" >> story.txt
git commit -am "Add the special idea"
git switch main
git branch -D idea
git reset --hard HEAD~2
```

Disaster: the branch `idea` is gone, and `main` lost two commits. Bring back **both**: `main` must show four lines again, and the branch `idea` must contain the special idea.

<details>
<summary>Hint</summary>

`git reflog` remembers every position of `HEAD`. The message of `git branch -D` also printed a hash.

</details>

<details>
<summary>Answer</summary>

```bash
git reflog
git reset --hard 'HEAD@{1}'
git branch idea 28622a0
git log --oneline --all --graph --decorate
```

Use the hash that **you** see in the reflog line `commit: Add the special idea` (here `28622a0`). The result:

```text
* 28622a0 (idea) Add the special idea
* c3802a1 (HEAD -> main) Add line 4
* a17aec2 Add line 3
* c5b0e53 Add line 2
* 99a925a Add line 1
```

Why it works: `HEAD@{1}` is where `HEAD` was before the reset (the tip of `main`), and a branch can be created at any commit, including one that no branch points to.

</details>

### Task 2 — Resolve a conflict

Setup:

```bash
cd ~/projects/git-playground/checkpoint
git init -b main conflict-task
cd conflict-task
printf 'name = "demo"\nversion = "0.1.0"\nauthor = "ada"\n' > app.conf
git add app.conf
git commit -m "Add app.conf"
git switch -c release
printf 'name = "demo"\nversion = "0.2.0"\nauthor = "ada"\n' > app.conf
git commit -am "Bump version to 0.2.0"
git switch main
printf 'name = "demo-tool"\nversion = "0.1.5"\nauthor = "ada"\n' > app.conf
git commit -am "Rename the tool and bump to 0.1.5"
```

Merge `release` into `main`. The final file must have the new name **and** the version `0.2.0`.

<details>
<summary>Hint</summary>

Merge, read the block, write the final three lines, `git add`, commit.

</details>

<details>
<summary>Answer</summary>

```bash
git merge release
printf 'name = "demo-tool"\nversion = "0.2.0"\nauthor = "ada"\n' > app.conf
git diff --check
git add app.conf
git commit --no-edit
git log --oneline --graph
cat app.conf
```

```text
*   1635a42 Merge branch 'release'
|\
| * 85c32fa Bump version to 0.2.0
* | e5e6963 Rename the tool and bump to 0.1.5
|/
* 9067988 Add app.conf
name = "demo-tool"
version = "0.2.0"
author = "ada"
```

Why it works: the two branches changed neighbouring lines, so Git showed one conflict block; you wrote the combination by hand and marked it resolved.

</details>

### Task 3 — A rejected push

Setup:

```bash
cd ~/projects/git-playground/checkpoint
git init --bare hub.git
git clone hub.git alice
cd alice
git config user.name Alice
git config user.email alice@example.com
echo base > shared.txt
git add shared.txt
git commit -m "Base"
git push -u origin main
cd ..
git clone hub.git bob
cd bob
git config user.name Bob
git config user.email bob@example.com
cd ../alice
echo "from alice" > alice.txt
git add alice.txt
git commit -m "Alice work"
git push
cd ../bob
echo "from bob" > bob.txt
git add bob.txt
git commit -m "Bob work"
git push
```

Bob's last `git push` is rejected. Fix it and show the shared history.

<details>
<summary>Hint</summary>

Fetch, merge `origin/main` (your `pull.ff only` setting stops a plain `git pull`), push.

</details>

<details>
<summary>Answer</summary>

```bash
git fetch
git merge origin/main --no-edit
git push
git log --oneline --graph
```

```text
*   a9575b9 Merge remote-tracking branch 'origin/main'
|\
| * eccd231 Alice work
* | 3d713ec Bob work
|/
* ca9e9ff Base
```

Why it works: Bob's `main` and the remote `main` diverged. The merge puts Alice's commit into Bob's history, so the push only adds commits to the remote.

</details>

### Task 4 — Find the bad commit

Setup:

```bash
cd ~/projects/git-playground/checkpoint
git init -b main detective
cd detective
for i in 1 2 3 4 5 6 7 8 9 10; do
  echo "ok $i" >> status.txt
  if [ $i = 7 ]; then echo "FAIL" >> status.txt; fi
  git add status.txt
  git commit -m "Step $i"
done
printf '#!/bin/sh\n! grep -q FAIL status.txt\n' > check.sh
chmod +x check.sh
```

Find the first commit that contains `FAIL`, automatically, and then return to the normal state.

<details>
<summary>Hint</summary>

`git bisect start <bad> <good>` and `git bisect run ./check.sh`. The first commit is good.

</details>

<details>
<summary>Answer</summary>

```bash
git bisect start HEAD HEAD~9
git bisect run ./check.sh
git bisect reset
```

The output of the second command ends with `… is the first bad commit` for `Step 7`, after about three runs of the script.

Why it works: the script succeeds when `FAIL` is absent. Bisect halves the range each time (10 commits need about 3 checks) and prints the first commit where the script fails.

</details>

## If you got something wrong

| Topic | Review |
|---|---|
| What version control is, hashes, Git vs GitHub | [Lesson 01.1](01-what-is-version-control.md) |
| Installing Git, config levels, identity, global ignore | [Lesson 01.2](02-install-and-configure-git.md) |
| `init`, `add`, `commit`, `status`, `.gitignore`, Cargo and Git | [Lesson 01.3](03-your-first-repository-and-commits.md) |
| `log`, `show`, `diff`, `blame`, `bisect`, naming commits | [Lesson 01.4](04-viewing-history-and-changes.md) |
| `restore`, `amend`, `reset`, `revert`, `reflog`, `stash`, `clean` | [Lesson 01.5](05-undoing-changes-safely.md) |
| Branches, `switch`, fast-forward and merge commits | [Lesson 01.6](06-branches-and-merging.md) |
| Conflict markers, resolving, aborting, `Cargo.lock` | [Lesson 01.7](07-merge-conflicts.md) |
| Remotes, `clone`, `fetch`, `pull`, `push`, SSH keys, secrets | [Lesson 01.8](08-remotes-github-and-gitlab.md) |
| Pull requests, review, `rebase`, tags, SemVer, CI, hooks | [Lesson 01.9](09-teamwork-pull-requests-and-conventions.md) |
| The terminal commands used everywhere | [Lesson 00.4](../00-getting-started/04-terminal-basics.md) |

## What's next

You now have a professional safety net. From this point on:

- every project goes into `~/projects/rust/code/pNN-<name>` (do `mkdir -p code` once, and always create projects with `cargo new code/pNN-<name>`);
- work on a branch, commit small and often, and merge into `main` at the end of each milestone;
- keep your `notes/` up to date.

In Phase 02 you start to program in Rust. Your first lesson there is [Lesson 02.1 — Variables and mutability](../02-programming-basics/01-variables-and-mutability.md). The words from this phase are explained in the [glossary](../GLOSSARY.md), and the [start page](../START-HERE.md) shows the whole path.

## Summary

- One learning repository with a clear layout, a good `.gitignore` and a README is the base of all your later work.
- Create the parent folder (`mkdir -p code`) **before** `cargo new code/…`, or Cargo creates a nested repository.
- A project is done when the acceptance commands print what the plan says; make one commit (or a few) per milestone.
- The professional loop: branch → commit → push → pull request with CI → squash merge → update `main` → tag a release.
- Conflicts are resolved by hand, tested and then committed; remotes reject pushes when you are behind.
- Keep secrets out of history; protect `main`; sign and tag with care.

## Sources and further reading

- [Pro Git](https://git-scm.com/book/en/v2) — the full free book; reread chapters 2, 3 and 7 now that you know the basics.
- [The Cargo Book — Creating a New Package](https://doc.rust-lang.org/cargo/guide/creating-a-new-project.html) — the VCS option of `cargo new`.
- [GitHub Docs — Quickstart for repositories](https://docs.github.com/en/repositories/creating-and-managing-repositories/quickstart-for-repositories) — creating and managing a repository.
- [GitHub Docs — About protected branches](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches) — for the stretch goal.
- [Keep a Changelog](https://keepachangelog.com/en/1.1.0/) — the changelog format of M7.
- [git-hooks manual](https://git-scm.com/docs/githooks) — every hook and `core.hooksPath`.
- [Oh Shit, Git!?!](https://ohshitgit.com/) — a short, friendly list of common Git disasters and their fixes.

---

[🏠 Start](../START-HERE.md) · [← 01.9 Teamwork: pull requests, code review and conventions](09-teamwork-pull-requests-and-conventions.md) · [📋 Phase 01](README.md) · [Phase 02 — Programming basics in Rust →](../02-programming-basics/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
