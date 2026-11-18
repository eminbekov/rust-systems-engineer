<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 01.5 Undoing changes safely](05-undoing-changes-safely.md) · [📋 Phase 01](README.md) · [01.7 Merge conflicts →](07-merge-conflicts.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/01-git-and-version-control/06-branches-and-merging.md) <!-- nav -->

# Lesson 01.6 — Branches and merging

> **Phase 01 — Git and version control** · Lesson 6 of 10 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a branch really is (a small movable pointer), and why branches in Git are cheap.
- How to create, list, switch, rename and delete branches.
- How to merge a branch into another, and the difference between a *fast-forward* merge and a *merge commit*.
- How to read the history graph.
- How to name branches, how to switch safely when you have unfinished work, and how to recover a deleted branch.

## Before you start

- You finished [Lesson 01.5 — Undoing changes safely](05-undoing-changes-safely.md).
- You have the Rust project `hello-rust` from [Lesson 01.3](03-your-first-repository-and-commits.md) in `~/projects/git-playground`.

## New words

| Word | Meaning |
|---|---|
| **branch** | A named, movable pointer to a commit. It marks the tip of one line of work. |
| **`main`** | The conventional name of the main branch of a repository. |
| **`HEAD`** | A pointer to the branch (or commit) you are on now. New commits are added to it. |
| **switch** (check out) | To move `HEAD` to another branch and make your files match it. |
| **merge** | To combine the work of one branch into another. |
| **fast-forward** | A merge that only moves the branch pointer forward, because the other branch contains all of the current one. No new commit is made. |
| **merge commit** | A commit with **two** parents. It records that two lines of work were joined. |
| **three-way merge** | A merge that compares both branch tips with their last common ancestor. |
| **common ancestor** (merge base) | The latest commit that both branches share. |
| **feature branch** | A short-lived branch for one task (a feature or a fix). |
| **`--no-ff`** | An option that forces a merge commit even when a fast-forward is possible. |

## 1. The problem branches solve

You work on a program that works. You want to try a big change, but you do not want to break the working version, and you may need to fix a small bug in the working version at the same time. With a single line of history that is hard.

A **branch** gives you a parallel line of history. You can try an idea in one branch, keep a stable version in another, and join them when the idea is ready. Branches in Git are so cheap that you create one for every task.

## 2. A branch is just a pointer

This is the key insight. A branch is not a copy of your files. It is a tiny file with **one commit hash** inside. Make a practice repository:

```bash
cd ~/projects/git-playground
mkdir branch-demo
cd branch-demo
git init
echo "# Recipes" > README.md
git add README.md
git commit -m "Start recipes"
echo "Pancakes: flour, milk, eggs" > recipes.txt
git add recipes.txt
git commit -m "Add pancakes"
```

Now look at the branches and at the files where Git keeps them:

```bash
git branch
cat .git/HEAD
cat .git/refs/heads/main
```

```text
* main
ref: refs/heads/main
bcf3362ed20ba84d4a39cc0062c3aad9f0ed1bf1
```

**What happens:**

1. `git branch` lists local branches. The `*` marks the branch you are on.
2. `.git/HEAD` says `ref: refs/heads/main`: `HEAD` points to the branch `main`.
3. `.git/refs/heads/main` contains the hash of the latest commit. That is the whole branch: 40 characters.

```text
 Start recipes ◀── Add pancakes
                       ▲
                     main ◀── HEAD
```

Create a second branch:

```bash
git branch soup
git branch
git log --oneline --decorate
```

```text
* main
  soup
bcf3362 (HEAD -> main, soup) Add pancakes
d4c72b0 Start recipes
```

**What happens:** `git branch soup` creates a new pointer, `soup`, at the same commit. Nothing was copied. `HEAD -> main` shows that you are still on `main`; `soup` points to the same commit.

```text
 Start recipes ◀── Add pancakes ◀── main ◀── HEAD
                                 ◀── soup
```

## 3. Switching and committing on a branch

Use `git switch` to move `HEAD` to another branch:

```bash
git switch soup
cat .git/HEAD
echo "Soup: water, carrots, salt" >> recipes.txt
git commit -am "Add soup"
git log --oneline --decorate
```

```text
Switched to branch 'soup'
ref: refs/heads/soup
[soup c84d638] Add soup
 1 file changed, 1 insertion(+)
c84d638 (HEAD -> soup) Add soup
bcf3362 (main) Add pancakes
d4c72b0 Start recipes
```

**What happens:**

1. `git switch soup` moves `HEAD` to `soup` and updates your files to match it.
2. The new commit was added on `soup`. Only the pointer `soup` moved forward; `main` stayed.

```text
 Start recipes ◀── Add pancakes ◀── Add soup ◀── soup ◀── HEAD
                        ▲
                      main
```

Go back and see the files change under your eyes:

```bash
git switch main
cat recipes.txt
git switch soup
cat recipes.txt
git switch main
```

```text
Pancakes: flour, milk, eggs
Pancakes: flour, milk, eggs
Soup: water, carrots, salt
```

**What happens:** on `main` the file has one line; on `soup` it has two. Git rewrites the files in your working tree each time you switch.

To create a branch **and** switch to it in one step, use `git switch -c <name>` (`-c` = create). Older tutorials use `git checkout -b <name>`; it does the same.

## 4. Merging: the fast-forward case

The soup recipe is ready. Merge `soup` into `main`: first switch to the branch that should **receive** the work, then merge the other one in.

```bash
git switch main
git merge soup
git log --oneline --decorate
git branch -d soup
```

```text
Updating bcf3362..c84d638
Fast-forward
 recipes.txt | 1 +
 1 file changed, 1 insertion(+)
c84d638 (HEAD -> main, soup) Add soup
bcf3362 Add pancakes
d4c72b0 Start recipes
Deleted branch soup (was c84d638).
```

**What happens:**

1. You were on `main`. `git merge soup` found that `soup` is simply `main` plus extra commits.
2. So Git did a **fast-forward**: it moved the pointer `main` forward to `c84d638`. No new commit was created.
3. `git branch -d soup` deletes the name `soup`. The commits stay in the history, because `main` reaches them. Deleting a branch deletes only the pointer.

```text
before:  A ◀── B ◀── main          after:  A ◀── B ◀── C ◀── main (HEAD)
                  ◀── C ◀── soup
```

## 5. Merging: when both branches moved (merge commit)

Now two people (or two of your tasks) change different things at the same time. Create a branch `salad`, add a commit, then go back to `main` and add a **different** commit there:

```bash
git switch -c salad
echo "Salad: tomato, cucumber" >> recipes.txt
git commit -am "Add salad"
git switch main
echo "All recipes are for two people." >> README.md
git commit -am "Add a note to README"
git log --oneline --graph --all --decorate
```

```text
Switched to a new branch 'salad'
[salad 6481c09] Add salad
 1 file changed, 1 insertion(+)
Switched to branch 'main'
[main de30e3f] Add a note to README
 1 file changed, 1 insertion(+)
* de30e3f (HEAD -> main) Add a note to README
| * 6481c09 (salad) Add salad
|/
* c84d638 Add soup
* bcf3362 Add pancakes
* d4c72b0 Start recipes
```

**What happens:** the history has **forked**. Both `main` and `salad` have a commit that the other one does not have. `--graph` draws the fork, `--all` shows all branches, not only the current one.

```text
                    ┌── Add salad ◀── salad
 … ◀── Add soup ◀───┤
                    └── Add a note to README ◀── main (HEAD)
```

A fast-forward is impossible now. Merge:

```bash
git merge salad --no-edit
git log --oneline --graph --all --decorate
```

```text
Merge made by the 'ort' strategy.
 recipes.txt | 1 +
 1 file changed, 1 insertion(+)
*   4a42176 (HEAD -> main) Merge branch 'salad'
|\
| * 6481c09 (salad) Add salad
* | de30e3f Add a note to README
|/
* c84d638 Add soup
* bcf3362 Add pancakes
* d4c72b0 Start recipes
```

**What happens:**

1. Git found the **common ancestor** (`Add soup`) and compared it with both tips. The two changes touch different files, so Git combined them by itself. This is called a **three-way merge**.
2. It created a **merge commit**, `4a42176`, with two parents. `--no-edit` accepts the default message "Merge branch 'salad'"; without it, Git opens your editor.
3. (`ort` is the name of Git's current merge algorithm. You do not need it now.)

Look inside the merge commit:

```bash
git cat-file -p HEAD
```

```text
tree cdd1d66f2dba6e65632a193271076c8d2a41df37
parent de30e3f6e144c13e802e88369512d86fef9867d4
parent 6481c095ae50a818cb9e1d6d06ea01bb55a41527
author Ada Learner <ada@example.com> 1794915965 +0500
committer Ada Learner <ada@example.com> 1794915965 +0500

Merge branch 'salad'
```

Two `parent` lines: the tip of `main` and the tip of `salad`. That is what a merge commit is.

> 💡 **Tip:** what if the same line was changed on both branches? Then Git cannot decide and reports a **merge conflict**. This is normal, and you will learn to solve it in [Lesson 01.7](07-merge-conflicts.md).

### Always make a merge commit: `--no-ff`

Sometimes a team wants a merge commit even when a fast-forward is possible, so that the history shows "this group of commits was one feature".

```bash
git switch -c rice
echo "Rice: rice, water" >> recipes.txt
git commit -am "Add rice"
git switch main
git merge --no-ff rice -m "Merge branch 'rice'"
git log --oneline --graph -4
```

```text
Merge made by the 'ort' strategy.
 recipes.txt | 1 +
 1 file changed, 1 insertion(+)
*   e5be826 Merge branch 'rice'
|\
| * 8ecd99c Add rice
|/
*   4a42176 Merge branch 'salad'
|\
| * 6481c09 Add salad
```

You will see in [Lesson 01.9](09-teamwork-pull-requests-and-conventions.md) how teams choose between merge commits, squash and rebase.

## 6. Listing and deleting branches

| Command | What it does |
|---|---|
| `git branch` | lists local branches |
| `git branch -v` | adds the last commit of each branch |
| `git branch --merged` | branches already merged into the current one (safe to delete) |
| `git branch --no-merged` | branches with work that is not merged yet |
| `git branch -d <name>` | deletes a branch, but **only if it is merged** |
| `git branch -D <name>` | deletes a branch **even if it is not merged** |
| `git branch -m <old> <new>` | renames a branch |

```bash
git branch --merged
git branch -d salad
```

```text
* main
  salad
Deleted branch salad (was 6481c09).
```

Git protects you from losing unmerged work:

```bash
git switch -c dessert
echo "Pie: apples, flour" >> recipes.txt
git commit -am "Add pie"
git switch main
git branch -d dessert
git branch --no-merged
git branch -D dessert
```

```text
error: the branch 'dessert' is not fully merged
hint: If you are sure you want to delete it, run 'git branch -D dessert'
  dessert
Deleted branch dessert (was 8b94729).
```

**What happens:** `-d` refuses, because the commit `Add pie` is not reachable from any other branch: deleting the pointer would make that commit "lost". `-D` forces it. Notice that Git prints the hash `8b94729`. Save it: with the hash (or with `git reflog`) you can bring the branch back with `git branch dessert 8b94729`.

## 7. Switching with unfinished work

Git changes your files when you switch. If you have uncommitted edits to a file that **differs** between the two branches, Git refuses, to protect your edits:

```bash
git switch -c tea
echo "Tea: water, tea leaves" >> recipes.txt
git commit -am "Add tea"
git switch main
echo "local edit" >> recipes.txt
git switch tea
```

```text
error: Your local changes to the following files would be overwritten by checkout:
	recipes.txt
Please commit your changes or stash them before you switch branches.
Aborting
```

**What happens:** `recipes.txt` has different content on `tea`, and you have an unsaved edit in it. Switching would overwrite your edit. You have two safe options:

1. **Commit** the edit (even as a "WIP" commit; WIP means Work In Progress), then switch.
2. **Stash** it ([Lesson 01.5](05-undoing-changes-safely.md)):

```bash
git stash
git switch tea
git switch main
git stash pop
```

```text
Saved working directory and index state WIP on main: e5be826 Merge branch 'rice'
Switched to branch 'tea'
Switched to branch 'main'
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   recipes.txt

no changes added to commit (use "git add" and/or "git commit -a")
Dropped refs/stash@{0} (85cba267b09db8662ee4be1c2d3a0b7186a9ae61)
```

If the edited file is the **same** on both branches, Git lets you switch and carries your edit along. Untracked files are carried too.

## 8. Renaming, jumping back, naming rules

```bash
git restore recipes.txt
git branch -m tea drinks
git branch -vv
git switch drinks
git switch -
```

```text
  drinks 498de6a Add tea
* main   e5be826 Merge branch 'rice'
Switched to branch 'drinks'
Switched to branch 'main'
```

**What happens:** `git branch -m tea drinks` renames the branch. `git switch -` jumps back to the branch you were on before (like "back" in a browser).

A branch name cannot contain spaces and some special characters:

```bash
git branch "bad name"
```

```text
fatal: 'bad name' is not a valid branch name
hint: See 'git help check-ref-format'
```

📏 **Convention:** teams usually name branches like this:

- all lowercase, words separated by hyphens: `fix-empty-input`;
- a prefix that says the type of work, separated by a slash: `feature/login-page`, `fix/crash-on-start`, `docs/readme`, `chore/update-deps`;
- short but meaningful; a ticket number if your team uses tickets: `fix/1234-null-pointer`;
- no spaces, no capital letters, no personal names for shared work.

The main branch is `main`. Some old projects still use `master`.

## 9. Branches in a Rust project

Branches work the same in a Cargo project. Try it in `hello-rust`:

```bash
cd ~/projects/git-playground/hello-rust
git switch -c feature/greeting
sed -i '' 's/Hello, world!/Hello from a branch!/' src/main.rs
git commit -am "Change the greeting"
cargo run -q
git switch main
cargo run -q
git merge feature/greeting
cargo run -q
git branch -d feature/greeting
git log --oneline --graph --decorate
```

```text
Switched to a new branch 'feature/greeting'
[feature/greeting 9781128] Change the greeting
 1 file changed, 1 insertion(+), 1 deletion(-)
Hello from a branch!
Switched to branch 'main'
Hello, world!
Updating cf73604..9781128
Fast-forward
 src/main.rs | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
Hello from a branch!
Deleted branch feature/greeting (was 9781128).
* 9781128 (HEAD -> main) Change the greeting
* cf73604 Create the project with cargo new
```

**What happens:** the program prints a different text on each branch, because `src/main.rs` differs. (`-q` makes Cargo quiet. `sed -i '' 's/old/new/' file` replaces text in a file on macOS; you can also just edit the file in RustRover.) The folder `target/` is ignored by Git, so it is not switched: Cargo rebuilds when the sources change.

> 💡 **Tip:** RustRover shows the current branch at the bottom right of the window (and in the *Git* widget). Click it to create or switch branches without the terminal.

## Common mistakes

> ⚠️ **Common mistake:** committing on the wrong branch. Before you commit, look at `git status` (the first line) or the prompt or the IDE (Integrated Development Environment) widget. If it already happened and you have not pushed: make a new branch at the current commit (`git branch feature/x`), then move `main` back (`git switch main`, `git reset --hard HEAD~1`). Use this carefully ([Lesson 01.5](05-undoing-changes-safely.md)).

> ⚠️ **Common mistake:** merging in the wrong direction. `git merge A` merges `A` **into the branch you are on**. Always switch to the receiving branch first (usually `main`).

> ⚠️ **Common mistake:** `error: the branch 'x' is not fully merged`. The branch has commits that no other branch has. Merge it first, or use `-D` only if you really want to throw the work away.

> ⚠️ **Common mistake:** keeping a branch alive for weeks. The longer a branch lives, the more it differs from `main`, and the harder the merge is. Make branches small and short-lived, and merge often.

> ⚠️ **Common mistake:** committing while in the "detached `HEAD`" state (after `git switch --detach` or after checking out a commit hash). The commit is not on any branch and is easy to lose. If it happens, create a branch right away: `git switch -c rescue`.

## Try it yourself

1. In `branch-demo`, create a branch `drinks`, add a line `Lemonade: water, lemon` to `recipes.txt`, and commit.
2. Switch to `main`. Is the lemonade line there? Why not?
3. Run `git log --oneline --graph --all`. Where does `drinks` point?
4. Merge `drinks` into `main`. Was it a fast-forward? Check the message.
5. Delete `drinks` with `-d`. Does it work? Why?
6. Create `fix/typo` and `feature/menu`, and commit different files on each. Merge both into `main` and look at `git log --oneline --graph`.
7. Open the project in RustRover and find the branch widget at the bottom right.

## Exercises

### Exercise 1 — The first branch (easy)

In `branch-demo`, create the branch `notes`, add a file `notes.txt` with one line, and commit it there. Prove that the file does **not** exist on `main`.

<details>
<summary>Hint</summary>

Use `git switch -c`, then `git switch main` and `ls`.

</details>

<details>
<summary>Answer</summary>

```bash
git switch -c notes
echo "Try the soup with bread." > notes.txt
git add notes.txt
git commit -m "Add notes.txt"
git switch main
ls
```

`ls` on `main` does not list `notes.txt`.

Why it works: the commit exists only on the branch `notes`. When you switch to `main`, Git rewrites the working tree to match `main`, so the file disappears from the folder (it is safe in the commit).

</details>

### Exercise 2 — Predict the merge (easy)

You are on `main` at commit B. The branch `feature` has commits C and D on top of B. Nothing was added to `main`. What kind of merge will `git merge feature` be? How many parents has the result?

<details>
<summary>Hint</summary>

Does `feature` contain everything that `main` has?

</details>

<details>
<summary>Answer</summary>

A **fast-forward**. `main` simply moves to D. No merge commit is created, so no new parents are involved: D keeps its single parent C.

Why it works: all of `main` (up to B) is already inside `feature`, so Git only has to move the pointer.

</details>

### Exercise 3 — Force a merge commit (medium)

Create two branches `a` and `b` from `main`, each with a commit on a **different** file (`a.txt`, `b.txt`). Merge both into `main`. Count the parents of the last merge commit using one command.

<details>
<summary>Hint</summary>

After merging `a` (a fast-forward), `b` has a common ancestor older than `main`, so a merge commit is needed. `git cat-file -p HEAD` lists parents; `grep -c parent` counts the lines.

</details>

<details>
<summary>Answer</summary>

```bash
git switch -c a
echo "a" > a.txt
git add a.txt
git commit -m "Add a.txt"
git switch main
git switch -c b
echo "b" > b.txt
git add b.txt
git commit -m "Add b.txt"
git switch main
git merge a
git merge b --no-edit
git cat-file -p HEAD | grep -c parent
```

```text
2
```

Why it works: merging `a` is a fast-forward (main had no new commits). After that, `main` has moved and `b` still starts from the old commit, so the history has forked and Git makes a merge commit with two parents.

</details>

### Exercise 4 — Bring back a deleted branch (medium)

Make a branch `dessert` with one commit, delete it with `-D`, and bring it back **with its commit**.

<details>
<summary>Hint</summary>

Git printed the hash when it deleted the branch. You can also find it with `git reflog`. Then `git branch <name> <hash>` creates a branch at that commit.

</details>

<details>
<summary>Answer</summary>

```bash
git switch -c dessert
echo "Pie: apples, flour" >> recipes.txt
git commit -am "Add pie"
git switch main
git branch -D dessert
git reflog -3
git branch dessert 'HEAD@{1}'
git branch -v
```

The last command lists `dessert` again, at the `Add pie` commit.

Why it works: the commit was never deleted; only the pointer was. The reflog remembers where `HEAD` was, so `HEAD@{1}` (the commit `Add pie`) is a valid starting point for a new branch.

</details>

### Exercise 5 — An urgent fix while you work (hard)

Setup: run `git switch -c feature/long-work` and then `echo "idea" >> recipes.txt` (do not commit). You are now in the middle of an unfinished edit on branch `feature/long-work`. A typo must be fixed on `main` **right now**. Describe and run the commands: park your work, fix the typo on a branch `fix/typo` from `main`, merge it, return to your feature branch with your edit intact.

<details>
<summary>Hint</summary>

Use `git stash`, then `git switch main`, `git switch -c fix/typo`, a commit, merge, `git switch feature/long-work`, `git stash pop`.

</details>

<details>
<summary>Answer</summary>

```bash
git stash
git switch main
git switch -c fix/typo
echo "typo fixed" >> README.md
git commit -am "Fix a typo"
git switch main
git merge fix/typo
git switch feature/long-work
git stash pop
git status -s
```

The final `git status -s` shows your unfinished edit as modified, as before.

Why it works: `stash` shelves the unfinished edit so that the working tree is clean and switching is allowed. After the fix is merged, `stash pop` puts the edit back on top of your feature branch.

</details>

## Quiz

1. What exactly is a branch in Git?

<details>
<summary>Answer</summary>

A small, movable pointer (a name) to a commit. It marks the tip of a line of work. It is not a copy of files.

</details>

2. What does `HEAD -> main` in `git log --decorate` mean?

<details>
<summary>Answer</summary>

`HEAD` (where you are) points to the branch `main`, so you are on `main`, and new commits will move `main`.

</details>

3. When is a merge a fast-forward?

<details>
<summary>Answer</summary>

When the current branch has no commits that the other branch lacks, so Git only has to move the current pointer forward. No merge commit is created.

</details>

4. How many parents has a merge commit?

<details>
<summary>Answer</summary>

Two (the tips of the two merged lines of work).

</details>

5. Why does `git branch -d` sometimes refuse, and what does `-D` do?

<details>
<summary>Answer</summary>

`-d` refuses if the branch has commits that are not merged anywhere else, because deleting the pointer would lose them. `-D` forces the deletion.

</details>

6. You have an unsaved edit and `git switch` refuses. What are two safe ways out?

<details>
<summary>Answer</summary>

Commit the edit, or stash it (`git stash`), then switch. Later `git stash pop` brings the edit back.

</details>

## Summary

- A branch is a pointer to a commit; creating and deleting branches costs almost nothing.
- `HEAD` points to the current branch; commits move that branch forward.
- `git switch <name>` moves between branches; `git switch -c <name>` creates and moves.
- `git merge <name>` brings `<name>` into the branch you are on: a fast-forward if possible, otherwise a merge commit with two parents.
- `--no-ff` forces a merge commit. `git log --oneline --graph --all` draws the history.
- `git branch -d` deletes only merged branches; `-D` forces. A deleted branch can be rebuilt from its hash or the reflog.
- Use short, lowercase, prefixed names (`feature/…`, `fix/…`), and keep branches small and short-lived.

## Sources and further reading

- [Pro Git — 3.1 Branches in a Nutshell](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell) — the pointer model with pictures.
- [Pro Git — 3.2 Basic Branching and Merging](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging) — the same workflow in the official book.
- [Pro Git — 3.4 Branching Workflows](https://git-scm.com/book/en/v2/Git-Branching-Branching-Workflows) — long-running and topic branches.
- [git-switch manual](https://git-scm.com/docs/git-switch) — every option of `switch`.
- [git-merge manual](https://git-scm.com/docs/git-merge) — merge strategies and options.
- [Learn Git Branching](https://learngitbranching.js.org/) — an interactive visual tutorial in the browser (needs internet, optional).

---

[🏠 Start](../START-HERE.md) · [← 01.5 Undoing changes safely](05-undoing-changes-safely.md) · [📋 Phase 01](README.md) · [01.7 Merge conflicts →](07-merge-conflicts.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
