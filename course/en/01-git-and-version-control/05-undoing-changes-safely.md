<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 01.4 Viewing history and changes](04-viewing-history-and-changes.md) · [📋 Phase 01](README.md) · [01.6 Branches and merging →](06-branches-and-merging.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/01-git-and-version-control/05-undoing-changes-safely.md) <!-- nav -->

# Lesson 01.5 — Undoing changes safely

> **Phase 01 — Git and version control** · Lesson 5 of 10 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- Which mistakes Git can undo, and which ones it cannot, and the golden rules that keep your work safe.
- How to discard edits (`git restore`), unstage files and fix the last commit (`git commit --amend`).
- How `git reset` works in its three modes, and why `--hard` is dangerous.
- How to undo a commit that other people already have, with `git revert`.
- How to rescue "lost" commits with `git reflog`, how to park unfinished work with `git stash`, and how to remove untracked files with `git clean`.

## Before you start

- You finished [Lesson 01.4 — Viewing history and changes](04-viewing-history-and-changes.md). You can read `git log`, `git diff` and `git status`.
- You work in `~/projects/git-playground`.

## New words

| Word | Meaning |
|---|---|
| **restore** | To bring a file back to the state it had in some commit or in the staging area. |
| **unstage** | To take a change out of the staging area, but keep it in your file. |
| **amend** | To replace the latest commit with a new, corrected one. |
| **reset** | To move the current branch to another commit (and optionally change the staging area and files). |
| **revert** | To create a **new** commit that undoes the changes of an older commit. |
| **reflog** | A private diary of where `HEAD` has been. It lets you find commits that are no longer on any branch. |
| **stash** | A shelf where you put away unfinished changes, so the working tree becomes clean. |
| **WIP** (Work In Progress) | Unfinished work. |
| **detached `HEAD`** | A state where `HEAD` points to a commit directly and not to a branch name. |
| **destructive command** | A command that can permanently delete work that is not saved in a commit. |

## 1. The golden rules

Git is very good at keeping what you **committed**. It is not able to protect what you did **not** commit.

| Your work is… | Can Git get it back? |
|---|---|
| committed | **Yes**, almost always, even after a "bad" command (see `git reflog`). |
| staged (`git add`, no commit) | Mostly no. The content of the staging area is lost if you overwrite it. |
| edited only in the working tree | **No.** If a command overwrites the file, those edits are gone. |

So the rules are:

1. **Commit early.** A commit is a safety net. You can always make a messy commit now and clean it up later.
2. **Read before you run.** Before any "undo" command, look at `git status` and `git log --oneline`.
3. **Know the danger level.** Commands that can destroy uncommitted work: `git restore <file>`, `git reset --hard`, `git clean -f`, `git checkout -- <file>`. All others in this lesson are safe.
4. **Never rewrite history that other people already have.** `amend`, `reset` and (later) `rebase` change commit hashes. If someone else already has the old commits, you create a mess. For shared commits, use `git revert`.

## 2. A map: "I want to…"

| I want to… | Command | Danger |
|---|---|---|
| throw away my edits to a file | `git restore <file>` | **destroys** the edits |
| take a file out of the next commit, keep my edits | `git restore --staged <file>` | safe |
| fix the message of the last commit | `git commit --amend -m "new message"` | safe if not shared |
| add a forgotten file to the last commit | `git add <file>` then `git commit --amend --no-edit` | safe if not shared |
| undo the last commit, keep the changes staged | `git reset --soft HEAD~1` | safe |
| undo the last commit, keep the changes unstaged | `git reset HEAD~1` (mixed) | safe |
| undo the last commit and delete the changes | `git reset --hard HEAD~1` | **destroys** uncommitted work |
| undo a commit that is already shared | `git revert <commit>` | safe |
| find a commit I "lost" | `git reflog` | safe |
| put unfinished work aside | `git stash` | safe |
| delete untracked files | `git clean -n` (preview), then `git clean -f` | **destroys** the files |
| get one old version of a file | `git restore --source=<commit> <file>` | overwrites the file |

Practice repository:

```bash
cd ~/projects/git-playground
mkdir undo-demo
cd undo-demo
git init
echo "apples" > shopping.txt
git add shopping.txt
git commit -m "Start shopping list"
echo "bread" >> shopping.txt
git commit -am "Add bread"
```

## 3. Discard edits: `git restore <file>`

You edited a file and decided the edit was a mistake.

```bash
echo "socks" >> shopping.txt
git status -s
git restore shopping.txt
git status -s
cat shopping.txt
```

```text
 M shopping.txt
apples
bread
```

**What happens:**

1. `echo "socks" >> …` adds a wrong line; `git status -s` shows ` M` (modified, not staged).
2. `git restore shopping.txt` replaces the file with the version from the staging area (which is the same as the last commit when nothing is staged). The edit is gone.
3. The second `git status -s` prints nothing: the tree is clean. `cat` shows the file as it was.

> ⚠️ **Common mistake:** `git restore <file>` has **no undo**. Git cannot bring back edits that were never committed. If you are not sure, first run `git diff <file>` to see what you are about to lose, or save a copy (`cp file file.bak`).

## 4. Unstage: `git restore --staged <file>`

You staged a file by mistake with `git add`.

```bash
echo "call mum" > todo.txt
git add todo.txt
git status -s
git restore --staged todo.txt
git status -s
rm todo.txt
```

```text
A  todo.txt
?? todo.txt
```

**What happens:** `--staged` tells `restore` to work on the staging area, not the file. The file is taken out of the next commit, and its content stays in the working tree (it is untracked again). This command is **safe**: nothing is lost. The last line, `rm todo.txt`, only deletes the practice file, so that it does not appear in the output of later sections.

> 📏 **Convention:** older tutorials use `git reset HEAD <file>` to unstage, and `git checkout -- <file>` to discard edits. They still work, but `git restore` (Git 2.23+) is clearer, because each command does one thing.

## 5. Fix the last commit: `git commit --amend`

### A typo in the message

```bash
echo "milk" >> shopping.txt
git commit -a -m "Add milq"
git commit --amend -m "Add milk"
git log --oneline
```

```text
[main f5b7bae] Add milq
 1 file changed, 1 insertion(+)
[main af70454] Add milk
 Date: Tue Nov 17 16:43:37 2026 +0500
 1 file changed, 1 insertion(+)
af70454 Add milk
19ccf86 Add bread
16d5fb3 Start shopping list
```

**What happens:** `--amend` does not add a commit. It **replaces** the latest one with a new commit that has the new message. Look at the hash: it changed from `f5b7bae` to `af70454`. The old commit still exists for a while in the reflog, but it is no longer in the history of `main`.

### A forgotten file

```bash
echo "eggs" >> shopping.txt
echo "tea" > drinks.txt
git add shopping.txt
git commit -m "Add eggs and drinks"
git status -s
git add drinks.txt
git commit --amend --no-edit
git show --stat --oneline HEAD
```

```text
[main 04b0399] Add eggs and drinks
 1 file changed, 1 insertion(+)
?? drinks.txt
[main c5a1577] Add eggs and drinks
 Date: Tue Nov 17 16:43:37 2026 +0500
 2 files changed, 2 insertions(+)
 create mode 100644 drinks.txt
c5a1577 Add eggs and drinks
 drinks.txt   | 1 +
 shopping.txt | 1 +
 2 files changed, 2 insertions(+)
```

**What happens:** you forgot `drinks.txt`. You stage it and run `git commit --amend --no-edit`. `--no-edit` keeps the old message. The new commit now contains both files.

> ⚠️ **Common mistake:** amending a commit that you have already **pushed** (sent to a remote, [Lesson 01.8](08-remotes-github-and-gitlab.md)). Your local history now differs from the remote one, and Git will refuse a normal push. Amend only commits that exist on your computer alone.

## 6. `git reset`: move the branch back

A branch name such as `main` is a pointer to a commit. `git reset <commit>` moves the pointer (and with it `HEAD`) to another commit. The three modes decide what happens to the staging area and to your files:

| Mode | Moves the branch | Resets the staging area | Resets the working files |
|---|---|---|---|
| `--soft` | yes | no | no |
| `--mixed` (the default) | yes | **yes** | no |
| `--hard` | yes | **yes** | **yes** (danger!) |

Picture: you made commit C by mistake and run `git reset HEAD~1`.

```text
before:   A ◀── B ◀── C ◀── main (HEAD)
after:    A ◀── B ◀── main (HEAD)         C is no longer on any branch
                      └─ the changes of C are... where? That depends on the mode:
                           --soft:  in the staging area
                           --mixed: in the working files, unstaged
                           --hard:  gone
```

### `--soft`: redo the commit

```bash
echo "cheese" >> shopping.txt
git commit -am "Add cheese too early"
git log --oneline -2
git reset --soft HEAD~1
git status -s
git log --oneline -2
```

```text
[main 3c6d189] Add cheese too early
 1 file changed, 1 insertion(+)
3c6d189 Add cheese too early
c5a1577 Add eggs and drinks
M  shopping.txt
c5a1577 Add eggs and drinks
af70454 Add milk
```

**What happens:** the commit is gone from the history, but `M ` shows that the change is still **staged**. You can now commit it again with a better message or together with other changes. `--soft` is also the classic way to **squash** (combine) several small commits into one: `git reset --soft HEAD~3`, then one `git commit`.

### `--mixed` (default): unstage and keep the edits

```bash
git commit -m "Add cheese"
git reset HEAD~1
git status -s
```

```text
Unstaged changes after reset:
M	shopping.txt
 M shopping.txt
```

**What happens:** the commit is gone, and the change is in your file but **not** staged (` M`). Use it when you want to split one big commit into several.

### `--hard`: throw everything away

```bash
git commit -am "Add cheese"
echo "rice" >> shopping.txt
git commit -am "Add rice (mistake)"
git log --oneline -3
git reset --hard HEAD~1
git status -s
cat shopping.txt
```

```text
a174f76 Add rice (mistake)
7f48478 Add cheese
c5a1577 Add eggs and drinks
HEAD is now at 7f48478 Add cheese
apples
bread
milk
eggs
cheese
```

**What happens:** `--hard` moved the branch **and** reset the file to the state of the earlier commit. The `rice` line is gone from the file. Because it **was committed**, you can still recover it (section 8). If the change had never been committed, `--hard` would destroy it for good.

> ⚠️ **Common mistake:** running `git reset --hard` while you have uncommitted edits. Those edits cannot be recovered. Before any `--hard`, run `git status`; if you see modified files you care about, commit or stash them first.

## 7. `git revert`: undo a commit by adding a new one

`reset` rewrites history. For a commit that **others already have**, you must not rewrite history. Instead, `git revert` creates a **new commit** with the opposite changes.

```bash
echo "socks" >> shopping.txt
git commit -am "Add socks"
git revert HEAD --no-edit
git log --oneline -3
cat shopping.txt
```

```text
[main b3046a1] Revert "Add socks"
 Date: Tue Nov 17 16:43:37 2026 +0500
 1 file changed, 1 deletion(-)
b3046a1 Revert "Add socks"
733b11b Add socks
7f48478 Add cheese
apples
bread
milk
eggs
cheese
```

**What happens:** `git revert HEAD` made a new commit, "Revert "Add socks"", that removes the `socks` line. The history keeps both commits, and the file is back to the older state. Nobody's history changed, so this is safe to push. `--no-edit` accepts the automatic message; without it Git opens your editor. You can revert any old commit: `git revert <hash>`.

```text
  A ◀── B ◀── C(socks) ◀── C'(revert socks)  ◀── main
```

## 8. `git reflog`: the safety net under the safety net

Every time `HEAD` moves (commit, reset, switch…), Git writes a line in the **reflog**. It is a private diary on your computer. Imagine you reset too far:

```bash
git reset --hard HEAD~3
git log --oneline
```

```text
HEAD is now at c5a1577 Add eggs and drinks
c5a1577 Add eggs and drinks
af70454 Add milk
19ccf86 Add bread
16d5fb3 Start shopping list
```

The `Add socks`, its revert and `Add cheese` commits have vanished from `git log`. Do not panic:

```bash
git reflog
```

```text
c5a1577 HEAD@{0}: reset: moving to HEAD~3
b3046a1 HEAD@{1}: revert: Revert "Add socks"
733b11b HEAD@{2}: commit: Add socks
7f48478 HEAD@{3}: reset: moving to HEAD~1
a174f76 HEAD@{4}: commit: Add rice (mistake)
7f48478 HEAD@{5}: commit: Add cheese
…
```

**What happens:** each line is: hash, `HEAD@{n}` (the position in the diary, 0 is now), and what happened. The line `HEAD@{1}` is the state **before** the reset. Go back to it:

```bash
git reset --hard 'HEAD@{1}'
git log --oneline -3
```

```text
HEAD is now at b3046a1 Revert "Add socks"
b3046a1 Revert "Add socks"
733b11b Add socks
7f48478 Add cheese
```

Everything is back. (The quotes around `HEAD@{1}` protect the braces from the shell.)

> 💡 **Tip:** by default Git keeps unreferenced commits in the reflog for at least about 30 days. If you think "I lost a commit", run `git reflog` and look for its message. This is why *committed work is almost impossible to lose*.

## 9. `git stash`: put work aside

You are in the middle of an unfinished change and must switch to something else. You do not want to commit half-done work. `git stash` saves your uncommitted changes on a shelf and cleans the working tree.

```bash
echo "half-written idea" >> shopping.txt
git stash
git status -s
git stash list
git stash pop
git status -s
```

```text
Saved working directory and index state WIP on main: b3046a1 Revert "Add socks"
stash@{0}: WIP on main: b3046a1 Revert "Add socks"
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   shopping.txt

no changes added to commit (use "git add" and/or "git commit -a")
Dropped refs/stash@{0} (dd0bc310092fafbd34f43a33a59fa3e56b7bc33f)
 M shopping.txt
```

**What happens:**

1. `git stash` saved the edit and restored the file to the last commit. The first `git status -s` printed nothing: clean tree.
2. `git stash list` shows the shelf: one entry, with the commit it was made on.
3. `git stash pop` puts the changes back and removes the entry from the shelf. It also prints a full `git status` report (the lines from `On branch main` to `no changes added…`) and the line `Dropped refs/stash@{0}`, which confirms that the entry was removed. `git stash apply` puts the changes back but keeps the entry.
4. The last `git status -s` shows ` M shopping.txt`: the half-written edit is back in the file. (Section 11 below throws it away with `git restore`.)

Useful extras: `git stash -u` also stashes untracked files; `git stash push -m "message"` gives the entry a name; `git stash drop` deletes an entry.

## 10. `git clean`: delete untracked files

`restore` does not touch untracked files. `git clean` does, and it is **destructive**, so always preview first.

```bash
echo x > junk1.tmp
mkdir junkdir
echo y > junkdir/junk2.tmp
git clean -n
git clean -nd
git clean -fd
git status -s
```

```text
Would remove junk1.tmp
Would remove junk1.tmp
Would remove junkdir/
Removing junk1.tmp
Removing junkdir/
```

**What happens:**

1. `-n` means *dry run*: Git lists what it would remove, and removes nothing. By default it ignores folders.
2. `-d` includes untracked folders.
3. `-f` (force) actually deletes. Git requires `-f` on purpose, to stop accidents.

Files that match `.gitignore` are kept unless you add `-x`. Never add `-x` unless you know why.

## 11. Getting an old version of a file

You do not have to go back in time for the whole project. You can take one file from any commit.

```bash
git restore --source=HEAD~3 shopping.txt
cat shopping.txt
git diff
git restore shopping.txt
```

```text
apples
bread
milk
eggs
diff --git a/shopping.txt b/shopping.txt
index 541ae17..41fe824 100644
--- a/shopping.txt
+++ b/shopping.txt
@@ -2,4 +2,3 @@ apples
 bread
 milk
 eggs
-cheese
```

**What happens:** `--source=HEAD~3` takes the file from three commits ago and puts it into your working tree as an edit. `git diff` shows the difference from the current commit. Keep the old version by committing it, or drop it with `git restore shopping.txt`.

To only **look** at an old commit, you can run `git switch --detach HEAD~3`. It puts you in a *detached `HEAD`* state: you see the old files, but you are on no branch. Anything you commit there is easy to lose. Return with `git switch main`. You will meet branches properly in the next lesson.

## Common mistakes

> ⚠️ **Common mistake:** `error: pathspec 'nosuchfile.txt' did not match any file(s) known to git`. A typo in the file name, or the file is not tracked. Check with `git status` or `git ls-files`.

> ⚠️ **Common mistake:** `fatal: ambiguous argument 'HEAD~50': unknown revision or path not in the working tree.` You asked to go further back than the history goes. Count with `git log --oneline`.

> ⚠️ **Common mistake:** using `git reset --hard` to "fix" a conflict or a confusing situation. First run `git status` and read it. Most situations have a safer fix.

> ⚠️ **Common mistake:** trying to hide a committed secret by deleting the file and committing again. The old commit still contains it. Treat the secret as stolen: replace it with a new one and revoke the old one. Rewriting history to remove it (with tools such as `git filter-repo`) is a separate, advanced step; if the commit is already on GitHub, replacing the secret is the only reliable fix.

> 🔒 **Security:** a deleted file is not a deleted secret. History keeps everything. Use `.gitignore` for secret files from the start ([Lesson 01.3](03-your-first-repository-and-commits.md)).

## Try it yourself

Use the `undo-demo` repository.

1. Add the line `lemons` to `shopping.txt`. Run `git status -s`, then discard the edit with `git restore shopping.txt`. Check with `cat`.
2. Create `scratch.txt`, stage it, then unstage it. Delete it.
3. Commit the line `lemons` with the typo message `Add lemmons`. Fix the message with `git commit --amend`. Compare the hashes before and after.
4. Make a commit `Add jam`. Run `git reset --soft HEAD~1`, see `git status -s`, then commit again as `Add jam to the list`.
5. Make a commit `Add oil`, then undo it with `git revert HEAD --no-edit`. Run `git log --oneline` and explain the three commits.
6. Run `git reset --hard HEAD~2`, then recover with `git reflog` and `git reset --hard 'HEAD@{1}'`.
7. Add an edit, stash it, check that the tree is clean, then `git stash pop`.

## Exercises

### Exercise 1 — Discard an edit safely (easy)

You changed `shopping.txt` and want to throw away the change. Write the commands that (a) show what you will lose, then (b) discard it, then (c) prove that the tree is clean.

<details>
<summary>Hint</summary>

Use `git diff`, `git restore` and `git status`.

</details>

<details>
<summary>Answer</summary>

```bash
git diff shopping.txt
git restore shopping.txt
git status
```

The last command prints `nothing to commit, working tree clean`.

Why it works: `git diff` previews the lost lines, `git restore` replaces the file with the committed version, and `git status` confirms.

</details>

### Exercise 2 — Unstage without losing the edit (easy)

You ran `git add .` and noticed that `.env` was staged. Remove only `.env` from the next commit, but keep the file on disk.

<details>
<summary>Hint</summary>

Use `restore` with the option that works on the staging area.

</details>

<details>
<summary>Answer</summary>

```bash
git restore --staged .env
git status -s
```

```text
?? .env
```

You should also add `.env` to `.gitignore`.

Why it works: `--staged` only touches the staging area. The file stays in your folder.

</details>

### Exercise 3 — Fix the last commit twice (medium)

Make a commit with the message `Add jelly` that contains `shopping.txt` only. Then (a) fix the message to `Add jam` and (b) add a forgotten file `recipes.txt` to the same commit, without creating a new commit. Show that the log has only one new commit.

<details>
<summary>Hint</summary>

You need `git commit --amend` two times: once with `-m`, once with `--no-edit` after `git add`.

</details>

<details>
<summary>Answer</summary>

```bash
echo "jelly" >> shopping.txt
echo "pancakes" > recipes.txt
git add shopping.txt
git commit -m "Add jelly"
git commit --amend -m "Add jam"
git add recipes.txt
git commit --amend --no-edit
git show --stat --oneline HEAD
git log --oneline -3
```

`git show --stat` lists both `recipes.txt` and `shopping.txt`, and `git log` shows a single `Add jam` commit above the previous history.

Why it works: `--amend` replaces the last commit each time instead of adding a new one.

</details>

### Exercise 4 — Squash two commits into one (medium)

Make two commits `wip` and `wip 2` that each add a line to `notes.txt`, then combine them into **one** commit `Add two lines to notes`. You may not use `rebase`.

<details>
<summary>Hint</summary>

`git reset --soft` moves the branch back but keeps all changes staged.

</details>

<details>
<summary>Answer</summary>

```bash
echo "one" > notes.txt
git add notes.txt
git commit -m "Add notes"
echo "two" >> notes.txt
git commit -am "wip"
echo "three" >> notes.txt
git commit -am "wip 2"
git reset --soft HEAD~2
git commit -m "Add two lines to notes"
git log --oneline
```

```text
7537e84 Add two lines to notes
f1904d8 Add notes
```

Why it works: `reset --soft HEAD~2` moves the branch two commits back but leaves the changes of both commits staged, so one commit can save them together. (In [Lesson 01.9](09-teamwork-pull-requests-and-conventions.md) you will do the same with interactive rebase.)

</details>

### Exercise 5 — Recover a "lost" commit (hard)

Continue from exercise 4: add two commits, `Add four` and `Add five`. Then run `git reset --hard HEAD~2` as a mistake. Recover both commits and prove it with `git log` and `cat notes.txt`.

<details>
<summary>Hint</summary>

Run `git reflog`. The entry just before `reset: moving to HEAD~2` is the state you want, `HEAD@{1}`.

</details>

<details>
<summary>Answer</summary>

```bash
echo "four" >> notes.txt
git commit -am "Add four"
echo "five" >> notes.txt
git commit -am "Add five"
git reset --hard HEAD~2
git reflog -4
git reset --hard 'HEAD@{1}'
git log --oneline
cat notes.txt
```

```text
e9a1d23 Add five
4126c8f Add four
7537e84 Add two lines to notes
f1904d8 Add notes
one
two
three
four
five
```

Why it works: the reflog still remembers where `HEAD` was before the reset. `git reset --hard 'HEAD@{1}'` moves the branch back to that commit and restores the files.

</details>

## Quiz

1. Which kinds of work can Git **not** recover after `git restore <file>`?

<details>
<summary>Answer</summary>

Edits that were never staged or committed. Git has no copy of them.

</details>

2. What is the difference between `git reset --soft`, `--mixed` and `--hard`?

<details>
<summary>Answer</summary>

All three move the branch. `--soft` keeps the staging area and files as they are. `--mixed` also resets the staging area but keeps the files. `--hard` resets the staging area **and** the files.

</details>

3. When should you use `git revert` instead of `git reset`?

<details>
<summary>Answer</summary>

When the commit is already shared (pushed). `revert` adds a new commit and does not rewrite history.

</details>

4. You ran `git reset --hard` and lost a commit. How can you find it?

<details>
<summary>Answer</summary>

Run `git reflog`, find the line for that commit, and use `git reset --hard <hash>` or `git switch -c rescue <hash>` to get it back.

</details>

5. What does `git commit --amend` do to the commit hash?

<details>
<summary>Answer</summary>

It makes a new commit, so the hash changes. That is why you must not amend commits that other people already have.

</details>

6. What is the safe way to preview `git clean`?

<details>
<summary>Answer</summary>

`git clean -n` (add `-d` for folders). It lists what would be deleted and deletes nothing.

</details>

## Summary

- Committed work is almost never lost; uncommitted work can be. Commit early and read `git status` first.
- `git restore <file>` discards edits (destructive); `git restore --staged <file>` unstages (safe).
- `git commit --amend` fixes the last commit; use it only before pushing.
- `git reset` moves the branch: `--soft` keeps changes staged, `--mixed` keeps them unstaged, `--hard` deletes them.
- `git revert` undoes a commit by adding an opposite one; it is the right tool for shared history.
- `git reflog` shows where `HEAD` has been and rescues "lost" commits.
- `git stash` shelves unfinished work; `git clean -n` previews, `git clean -f` deletes untracked files.

## Sources and further reading

- [Pro Git — 2.4 Undoing Things](https://git-scm.com/book/en/v2/Git-Basics-Undoing-Things) — amend, unstage and restore in the official book.
- [Pro Git — 7.7 Reset Demystified](https://git-scm.com/book/en/v2/Git-Tools-Reset-Demystified) — a deep explanation of the three trees and the three reset modes.
- [Pro Git — 7.3 Stashing and Cleaning](https://git-scm.com/book/en/v2/Git-Tools-Stashing-and-Cleaning) — `stash` and `clean` in detail.
- [git-restore manual](https://git-scm.com/docs/git-restore) — the exact options of `restore`.
- [git-reflog manual](https://git-scm.com/docs/git-reflog) — how the reflog works and how long it keeps entries.
- [GitHub Docs — Removing sensitive data from a repository](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository) — what to do if a secret was committed.

---

[🏠 Start](../START-HERE.md) · [← 01.4 Viewing history and changes](04-viewing-history-and-changes.md) · [📋 Phase 01](README.md) · [01.6 Branches and merging →](06-branches-and-merging.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
