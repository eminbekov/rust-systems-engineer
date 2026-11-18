<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 01.3 Your first repository and commits](03-your-first-repository-and-commits.md) · [📋 Phase 01](README.md) · [01.5 Undoing changes safely →](05-undoing-changes-safely.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/01-git-and-version-control/04-viewing-history-and-changes.md) <!-- nav -->

# Lesson 01.4 — Viewing history and changes

> **Phase 01 — Git and version control** · Lesson 4 of 10 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How to read the history with `git log`, in short and long forms, and how to filter it.
- How to read a diff, and how to compare working files, staged files and any two commits with `git diff`.
- How to look at one commit with `git show`, and how to name commits (`HEAD~2`, short hashes).
- How to find who changed a line (`git blame`) and when a piece of text appeared (`git log -S`).
- How to find the commit that broke something with `git bisect`.

## Before you start

- You finished [Lesson 01.3 — Your first repository and commits](03-your-first-repository-and-commits.md).
- You work in `~/projects/git-playground`.

## New words

| Word | Meaning |
|---|---|
| **revision** | Any way to name a commit: a hash, `HEAD`, `HEAD~2`, a branch name or a tag. |
| **`HEAD~N`** | The commit N steps before `HEAD`. `HEAD~1` (or `HEAD^`) is the parent. |
| **hunk** | One continuous block of changes inside a diff. |
| **unified diff** | The standard diff format with `-` and `+` lines and `@@` markers. |
| **blame** | Showing, for each line of a file, the commit and the author that last changed it. |
| **bisect** | Finding the commit that introduced a problem by repeatedly cutting the history in half (binary search). |
| **pickaxe** | The nickname of `git log -S`, which finds commits where the number of occurrences of a text changed. |
| **pager** | A program (`less`) that shows long output one screen at a time. `Space` = next page, `q` = quit. |

## 1. A small history to explore

You need a repository with several commits. Create one by pasting this block into your terminal. Read it first: it makes five commits for a tiny program `greeter.rs`, and the third commit is "written" by another author, so you can practise filtering.

```bash
cd ~/projects/git-playground
mkdir history-demo
cd history-demo
git init

printf 'fn main() {\n    println!("Hello");\n}\n' > greeter.rs
echo "# history-demo" > README.md
git add .
git commit -m "Add greeter and README"

printf 'fn main() {\n    println!("Hello, world");\n}\n' > greeter.rs
git commit -am "Greet the whole world"

printf 'fn greeting() -> String {\n    String::from("Hello, world")\n}\n\nfn main() {\n    println!("{}", greeting());\n}\n' > greeter.rs
git commit -am "Move the greeting into a function" --author="Bob Builder <bob@example.com>"

printf '# history-demo\n\nA tiny program that says hello.\n' > README.md
git commit -am "Describe the program in README"

printf 'fn greeting() -> String {\n    String::from("Hello, Rust world")\n}\n\nfn main() {\n    println!("{}", greeting());\n}\n' > greeter.rs
git commit -am "Mention Rust in the greeting"
```

**What happens:** each `printf` rewrites a file, and each `git commit -am` saves all changes to tracked files with a message. The `--author` option sets a different author for one commit. (You do not need to understand the Rust code yet; it is only text to track. Phase 02 teaches the language.)

The outputs below come from a real run. **Your hashes and dates will be different.** Always use your own hashes when a command needs one.

## 2. `git log`: read the history

```bash
git log
```

```text
commit 996bec4df4b786faa0bf3207bd26019a7367d600
Author: Ada Learner <ada@example.com>
Date:   Mon Nov 16 16:45:00 2026 +0500

    Mention Rust in the greeting

commit a499fb029cfeab10cc396e420a4174688b9f0607
Author: Ada Learner <ada@example.com>
Date:   Fri Nov 13 14:00:00 2026 +0500

    Describe the program in README

commit 3c8a092fd0b278e71d36eb7dfba44da1cf9038f4
Author: Bob Builder <bob@example.com>
Date:   Thu Nov 12 09:15:00 2026 +0500

    Move the greeting into a function
…
```

**What happens:** commits are listed **newest first**. Each shows the full hash, the author, the date and the message. If the list is longer than the screen, Git opens a pager: use `Space` for the next page and `q` to leave.

Short forms you will use daily:

```bash
git log --oneline
git log -3 --oneline
git log --oneline --graph --decorate
```

```text
996bec4 Mention Rust in the greeting
a499fb0 Describe the program in README
3c8a092 Move the greeting into a function
2c71f79 Greet the whole world
bd338f3 Add greeter and README
```

```text
996bec4 Mention Rust in the greeting
a499fb0 Describe the program in README
3c8a092 Move the greeting into a function
```

```text
* 996bec4 (HEAD -> main) Mention Rust in the greeting
* a499fb0 Describe the program in README
* 3c8a092 Move the greeting into a function
* 2c71f79 Greet the whole world
* bd338f3 Add greeter and README
```

**What happens:**

1. `--oneline` prints one line per commit: short hash and subject.
2. `-3` limits the output to the last 3 commits (`-n 3` works too).
3. `--graph` draws the shape of the history on the left (a straight line now; branches in the next lessons) and `--decorate` shows names attached to commits: `HEAD -> main` means "you are on the branch `main`, and it points here".

Show the files each commit touched, or the full changes:

```bash
git log --stat -2
git log -p -1
```

```text
commit 996bec4df4b786faa0bf3207bd26019a7367d600
Author: Ada Learner <ada@example.com>
Date:   Mon Nov 16 16:45:00 2026 +0500

    Mention Rust in the greeting

 greeter.rs | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
…
```

```text
commit 996bec4df4b786faa0bf3207bd26019a7367d600
Author: Ada Learner <ada@example.com>
Date:   Mon Nov 16 16:45:00 2026 +0500

    Mention Rust in the greeting

diff --git a/greeter.rs b/greeter.rs
index 294d8b3..dac1d51 100644
--- a/greeter.rs
+++ b/greeter.rs
@@ -1,5 +1,5 @@
 fn greeting() -> String {
-    String::from("Hello, world")
+    String::from("Hello, Rust world")
 }
 
 fn main() {
```

`--stat` adds a summary of changed files; `-p` (patch) adds the full diff of each commit. We will read this diff format in section 4.

### Filtering the history

| Command | What it shows |
|---|---|
| `git log --author=Bob --oneline` | commits whose author name contains "Bob" |
| `git log --grep=README --oneline` | commits whose **message** contains "README" |
| `git log --since="2 weeks ago" --oneline` | commits newer than the date (also `--until`) |
| `git log --oneline -- README.md` | commits that changed the file `README.md` (the `--` separates paths from options) |
| `git log --pretty=format:'%h %an %ad %s' --date=short` | your own format: `%h` short hash, `%an` author name, `%ad` date, `%s` subject |

```bash
git log --author=Bob --oneline
git log --grep=README --oneline
git log --oneline -- README.md
```

```text
3c8a092 Move the greeting into a function
```

```text
a499fb0 Describe the program in README
bd338f3 Add greeter and README
```

```text
a499fb0 Describe the program in README
bd338f3 Add greeter and README
```

## 3. Naming commits

Many commands need "a commit". You can name it in several ways:

| Name | Meaning |
|---|---|
| `996bec4df4b786faa0bf3207bd26019a7367d600` | the full hash |
| `996bec4` | a short hash: enough first characters to be unique (7 is the usual) |
| `HEAD` | the commit you are on now |
| `HEAD~1` or `HEAD^` | its parent (one step back) |
| `HEAD~3` | three steps back |
| `main` | the commit that the branch `main` points to |
| `HEAD~3..HEAD` | a **range**: the commits after `HEAD~3` up to `HEAD` |

```text
 bd338f3 ◀─ 2c71f79 ◀─ 3c8a092 ◀─ a499fb0 ◀─ 996bec4
  HEAD~4     HEAD~3     HEAD~2     HEAD~1      HEAD
```

```bash
git rev-parse HEAD
git rev-parse --short HEAD
git rev-parse HEAD~2
git log --oneline HEAD~3..HEAD
```

```text
996bec4df4b786faa0bf3207bd26019a7367d600
996bec4
3c8a092fd0b278e71d36eb7dfba44da1cf9038f4
996bec4 Mention Rust in the greeting
a499fb0 Describe the program in README
3c8a092 Move the greeting into a function
```

**What happens:** `git rev-parse` turns any name into a full hash (`--short` gives the short one). The range `HEAD~3..HEAD` means "reachable from `HEAD` but not from `HEAD~3`": the last three commits.

## 4. `git show` and how to read a diff

`git show <revision>` shows one commit: the message and its diff. Without a revision it shows `HEAD`.

```bash
git show HEAD~2
```

```text
commit 3c8a092fd0b278e71d36eb7dfba44da1cf9038f4
Author: Bob Builder <bob@example.com>
Date:   Thu Nov 12 09:15:00 2026 +0500

    Move the greeting into a function

diff --git a/greeter.rs b/greeter.rs
index bcb94c2..294d8b3 100644
--- a/greeter.rs
+++ b/greeter.rs
@@ -1,3 +1,7 @@
+fn greeting() -> String {
+    String::from("Hello, world")
+}
+
 fn main() {
-    println!("Hello, world");
+    println!("{}", greeting());
 }
```

How to read it line by line:

| Line | Meaning |
|---|---|
| `diff --git a/greeter.rs b/greeter.rs` | the file being compared; `a/` is the old version, `b/` the new |
| `index bcb94c2..294d8b3 100644` | short hashes of the old and the new file content, and the file mode |
| `--- a/greeter.rs` / `+++ b/greeter.rs` | old file / new file |
| `@@ -1,3 +1,7 @@` | a **hunk** header: in the old file, 3 lines starting at line 1; in the new file, 7 lines starting at line 1 |
| lines starting with `-` | removed |
| lines starting with `+` | added |
| lines starting with a space | unchanged context, shown to help you find the place |

A changed line appears as a `-` line (the old one) followed by a `+` line (the new one).

You can also look at a file *as it was* in a commit, and list only the names of changed files:

```bash
git show 3c8a092:greeter.rs
git show --stat HEAD~1
```

```text
fn greeting() -> String {
    String::from("Hello, world")
}

fn main() {
    println!("{}", greeting());
}
```

```text
commit a499fb029cfeab10cc396e420a4174688b9f0607
Author: Ada Learner <ada@example.com>
Date:   Fri Nov 13 14:00:00 2026 +0500

    Describe the program in README

 README.md | 2 ++
 1 file changed, 2 insertions(+)
```

## 5. `git diff`: compare things

`git diff` compares two states. Which two depends on the options:

| Command | Compares |
|---|---|
| `git diff` | working tree **vs** staging area: your changes that are *not yet staged* |
| `git diff --staged` (or `--cached`) | staging area **vs** last commit: what the next commit will contain |
| `git diff HEAD` | working tree **vs** last commit: everything you changed since the last commit |
| `git diff A B` | commit `A` **vs** commit `B` |
| `git diff A..B` | the same as above |
| `git diff A B -- file` | only one file |
| `git diff --stat` | only a summary of changed files and lines |
| `git diff --name-only` | only the names of changed files |

See the three-level behaviour yourself:

```bash
echo "" >> README.md
echo "More text." >> README.md
git diff
git diff --staged
git add README.md
git diff
git diff --staged
```

```text
diff --git a/README.md b/README.md
index 9c4d855..424b4fd 100644
--- a/README.md
+++ b/README.md
@@ -1,3 +1,5 @@
 # history-demo
 
 A tiny program that says hello.
+
+More text.
```

After the first `git diff`, `git diff --staged` prints nothing (nothing is staged). After `git add`, the situation flips: `git diff` prints nothing, and `git diff --staged` shows the same hunk.

> 💡 **Tip:** before every commit, run `git diff --staged` to review exactly what you are about to save. It is the cheapest code review there is. Undo the experiment with `git restore --staged README.md` and then `git restore README.md` (you will study these in [Lesson 01.5](05-undoing-changes-safely.md)).

Compare two commits:

```bash
git diff HEAD~1 HEAD
git diff --stat HEAD~3 HEAD
```

```text
diff --git a/greeter.rs b/greeter.rs
index 294d8b3..dac1d51 100644
--- a/greeter.rs
+++ b/greeter.rs
@@ -1,5 +1,5 @@
 fn greeting() -> String {
-    String::from("Hello, world")
+    String::from("Hello, Rust world")
 }
 
 fn main() {
```

```text
 README.md  | 2 ++
 greeter.rs | 6 +++++-
 2 files changed, 7 insertions(+), 1 deletion(-)
```

## 6. `git blame`: who wrote this line?

```bash
git blame greeter.rs
```

```text
3c8a092f (Bob Builder 2026-11-12 09:15:00 +0500 1) fn greeting() -> String {
996bec4d (Ada Learner 2026-11-16 16:45:00 +0500 2)     String::from("Hello, Rust world")
3c8a092f (Bob Builder 2026-11-12 09:15:00 +0500 3) }
3c8a092f (Bob Builder 2026-11-12 09:15:00 +0500 4) 
^bd338f3 (Ada Learner 2026-11-10 10:00:00 +0500 5) fn main() {
3c8a092f (Bob Builder 2026-11-12 09:15:00 +0500 6)     println!("{}", greeting());
^bd338f3 (Ada Learner 2026-11-10 10:00:00 +0500 7) }
```

**What happens:** each line is prefixed with the commit that **last changed it**, the author, the date and the line number. A leading `^` marks the first commit of the repository. To jump to the commit, run `git show 3c8a092`. Use `-L 6,6` to blame only one line.

`blame` is not about blaming people. It answers "why does this line look like this?" by leading you to the commit message. In RustRover the same view exists: right-click the left margin of the editor and choose *Annotate with Git Blame*.

## 7. Searching the history

Which commit added the word "Rust" to the greeting? `git log -S` (the *pickaxe*) lists commits where the number of occurrences of a text changed:

```bash
git log -S"Rust world" --oneline
git log -S"greeting()" --oneline
```

```text
996bec4 Mention Rust in the greeting
```

```text
3c8a092 Move the greeting into a function
```

**What happens:** the first command finds the only commit that added `Rust world`. The second finds the commit that introduced the function name.

Search the **current** files (not the history) with `git grep`:

```bash
git grep -n "Hello"
```

```text
greeter.rs:2:    String::from("Hello, Rust world")
```

`git grep` searches only files tracked by Git, so it skips `target/` and other ignored files; that makes it faster and quieter than a plain `grep`.

## 8. `git bisect`: find the commit that broke something

Imagine a test passed last week and fails today, with 100 commits between. Checking each one is slow. `git bisect` finds the guilty commit by **binary search**: it asks you to test the middle commit, throws away half of the history, and repeats. 100 commits need only about 7 checks.

You tell Git a **bad** commit (where the bug exists) and a **good** commit (where it did not). Git checks out the middle and you answer `good` or `bad`.

Prepare a small repository with a hidden "bug" in step 5 (copy and paste):

```bash
cd ~/projects/git-playground
mkdir bisect-demo
cd bisect-demo
git init
for i in 1 2 3 4 5 6 7 8; do
  echo "line $i" >> app.txt
  if [ $i = 5 ]; then echo "BUG" >> app.txt; fi
  git add app.txt
  git commit -m "Step $i"
done
git log --oneline
```

```text
30f26db Step 8
be02adb Step 7
7e63b37 Step 6
491dfe9 Step 5
e1a05de Step 4
38ae93a Step 3
f0432b8 Step 2
4669310 Step 1
```

(The loop repeats the steps for numbers 1 to 8. The `if` adds the line `BUG` at step 5.) Now bisect by hand:

```bash
git bisect start
git bisect bad
git bisect good HEAD~7
```

```text
status: waiting for both good and bad commits
status: waiting for good commit(s), bad commit known
Bisecting: 3 revisions left to test after this (roughly 2 steps)
[e1a05ded1cffb1d3f583dd3549403081d4e41ed0] Step 4
```

**What happens:**

1. `git bisect start` begins the session.
2. `git bisect bad` marks the current commit (`HEAD`, step 8) as bad.
3. `git bisect good HEAD~7` marks step 1 as good.
4. Git checks out the middle commit (step 4) for you to test. Your files now look like step 4.

Test it (here: does `app.txt` contain the word `BUG`?) and answer:

```bash
grep BUG app.txt
git bisect good
```

`grep` prints nothing, so step 4 is good. Git moves on to step 6, then to step 5. Each time you test and answer `git bisect good` or `git bisect bad`. When it has found the answer, Git prints:

```text
491dfe96ef4781af3dd52b3af4888eb7b644fa27 is the first bad commit
```

Finish with `git bisect reset`, which returns you to where you started:

```bash
git bisect reset
```

```text
Previous HEAD position was 491dfe9 Step 5
Switched to branch 'main'
```

You can also let a script do the testing. A script that exits with code 0 means "good", and non-zero means "bad":

```bash
printf '#!/bin/sh\n! grep -q BUG app.txt\n' > check.sh
chmod +x check.sh
git bisect start HEAD HEAD~7
git bisect run ./check.sh
git bisect reset
```

```text
Bisecting: 3 revisions left to test after this (roughly 2 steps)
[e1a05ded1cffb1d3f583dd3549403081d4e41ed0] Step 4
running './check.sh'
Bisecting: 1 revision left to test after this (roughly 1 step)
[7e63b373500d785f2995587315b13582541f088e] Step 6
running './check.sh'
Bisecting: 0 revisions left to test after this (roughly 0 steps)
[491dfe96ef4781af3dd52b3af4888eb7b644fa27] Step 5
running './check.sh'
491dfe96ef4781af3dd52b3af4888eb7b644fa27 is the first bad commit
bisect found first bad commit
```

**What happens:** `git bisect start HEAD HEAD~7` starts with the bad and good commits given at once. `git bisect run ./check.sh` runs the script on each middle commit. The script `check.sh` runs `grep -q BUG app.txt` (`-q` = quiet) and the `!` reverses the result: success if `BUG` is **not** found. In a Rust project the script would usually be `cargo test`, because it also exits with 0 for success and non-zero for failure.

> 🧪 **Testing:** `git bisect run cargo test` is a powerful combination: a failing test and a long history are enough to find the exact commit. You will write your own tests from [Lesson 02.8](../02-programming-basics/08-your-first-tests.md).

## Common mistakes

> ⚠️ **Common mistake:** `git diff` prints nothing, and you think nothing changed. If you already ran `git add`, the changes are in the staging area: use `git diff --staged`.

> ⚠️ **Common mistake:** you are stuck in the pager. The screen shows `:` or `(END)` at the bottom. Press `q`.

> ⚠️ **Common mistake:** `fatal: ambiguous argument 'HEAD~10': unknown revision`. The history is shorter than you asked for. Run `git log --oneline` and count.

> ⚠️ **Common mistake:** leaving a bisect session open and forgetting about it. The status line shows `(no branch, bisect started on main)`. Always finish with `git bisect reset`.

> ⚠️ **Common mistake:** using `git blame` to find someone to blame. The point is to read the commit message and the context. Keep your reviews kind.

## Try it yourself

In `history-demo`:

1. Run `git log --oneline` and write down the hash of the second commit.
2. Run `git show <that hash>`. Which line changed? Read the `-` and `+` lines.
3. Run `git log --stat` and find out how many lines the third commit added.
4. Run `git diff HEAD~4 HEAD --stat`. Which files changed overall?
5. Edit `README.md`: add a line, then compare `git diff` and `git diff --staged` before and after `git add`.
6. Run `git blame README.md`. Which commit wrote the last line?
7. Run `git restore --staged README.md && git restore README.md` to leave the repository clean.

## Exercises

### Exercise 1 — Count and name (easy)

Print only the number of commits in `history-demo`, and print only the subject (message) of the latest commit.

<details>
<summary>Hint</summary>

`git log --oneline` prints one line per commit; `wc -l` counts lines. For the subject use `git log -1 --format=%s` (`%s` is the subject).

</details>

<details>
<summary>Answer</summary>

```bash
git log --oneline | wc -l
git log -1 --format=%s
```

```text
       5
Mention Rust in the greeting
```

Why it works: `|` sends the output of `git log` into `wc -l`, which counts lines (the spaces before the number are normal on macOS). `-1` limits to one commit and `--format=%s` prints only its subject.

</details>

### Exercise 2 — What changed in a commit? (easy)

Show which files the commit "Describe the program in README" changed, **without** showing the diff.

<details>
<summary>Hint</summary>

`git show` has a `--stat` option, and also `--name-only`.

</details>

<details>
<summary>Answer</summary>

```bash
git show --stat HEAD~1
```

or, names only:

```bash
git show --name-only --format=%h HEAD~1
```

```text
a499fb0

README.md
```

Why it works: `HEAD~1` is the second newest commit. `--stat` or `--name-only` replace the full diff with a summary.

</details>

### Exercise 3 — Who wrote the function? (medium)

Find out who wrote the line `fn greeting() -> String {` and in which commit. Use one command, then confirm by showing that commit's subject.

<details>
<summary>Hint</summary>

`git blame -L 1,1 greeter.rs` blames only line 1. Then use `git show -s --format=%s <hash>`.

</details>

<details>
<summary>Answer</summary>

```bash
git blame -L 1,1 greeter.rs
git show -s --format='%an: %s' 3c8a092
```

```text
3c8a092f (Bob Builder 2026-11-12 09:15:00 +0500 1) fn greeting() -> String {
Bob Builder: Move the greeting into a function
```

Use your own hash from `git blame`. `-s` suppresses the diff, and `%an` is the author name.

Why it works: `blame` shows the last commit that touched the line. `show -s` with a format prints the facts you asked for.

</details>

### Exercise 4 — Compare the first and the last version (medium)

Show the full difference of `greeter.rs` between the very first commit and the latest, and then only a summary of all changed files.

<details>
<summary>Hint</summary>

Use `git diff <old> <new> -- greeter.rs`, with `HEAD~4` for the old one. Add `--stat` for the summary.

</details>

<details>
<summary>Answer</summary>

```bash
git diff HEAD~4 HEAD -- greeter.rs
git diff --stat HEAD~4 HEAD
```

```text
diff --git a/greeter.rs b/greeter.rs
index 244f744..dac1d51 100644
--- a/greeter.rs
+++ b/greeter.rs
@@ -1,3 +1,7 @@
+fn greeting() -> String {
+    String::from("Hello, Rust world")
+}
+
 fn main() {
-    println!("Hello");
+    println!("{}", greeting());
 }
```

```text
 README.md  | 2 ++
 greeter.rs | 6 +++++-
 2 files changed, 7 insertions(+), 1 deletion(-)
```

Why it works: `git diff A B` compares two commits directly, ignoring those in between.

</details>

### Exercise 5 — Find the culprit with a script (hard)

In `bisect-demo`, find the first commit where `app.txt` contains `BUG` using `git bisect run`, and then go back to the normal state. Write all commands.

<details>
<summary>Hint</summary>

You need a script that exits with a non-zero code when the file contains `BUG`. Start with `git bisect start <bad> <good>`.

</details>

<details>
<summary>Answer</summary>

```bash
printf '#!/bin/sh\n! grep -q BUG app.txt\n' > check.sh
chmod +x check.sh
git bisect start HEAD HEAD~7
git bisect run ./check.sh
git bisect reset
```

The output ends with `… is the first bad commit` for the commit `Step 5`.

Why it works: the script returns 0 (good) when `BUG` is absent and non-zero (bad) when it is present, so the binary search finds the first commit with `BUG` in about three steps. `git bisect reset` ends the session.

</details>

## Quiz

1. What is the difference between `git diff` and `git diff --staged`?

<details>
<summary>Answer</summary>

`git diff` shows changes not yet staged (working tree vs staging area). `git diff --staged` shows what is staged for the next commit (staging area vs last commit).

</details>

2. What does `HEAD~2` mean?

<details>
<summary>Answer</summary>

The commit two steps before `HEAD` (the grandparent of the current commit).

</details>

3. In a diff, what do lines starting with `-` and `+` mean?

<details>
<summary>Answer</summary>

`-` lines were removed, `+` lines were added. A changed line shows as one of each.

</details>

4. Which command lists only the commits that changed `README.md`?

<details>
<summary>Answer</summary>

`git log --oneline -- README.md`.

</details>

5. How does `git bisect` find a bad commit in 100 commits with only about 7 checks?

<details>
<summary>Answer</summary>

It uses binary search: it tests the middle commit, discards the half that cannot contain the first bad commit, and repeats.

</details>

6. Which command finds the commit that added a given piece of text?

<details>
<summary>Answer</summary>

`git log -S"text"` (the pickaxe).

</details>

## Summary

- `git log` reads the history; `--oneline`, `--graph`, `--stat`, `-p`, `-n` and filters like `--author`, `--grep` and `-- path` make it useful.
- A commit can be named by a hash, a short hash, `HEAD`, `HEAD~N` or a branch name; `A..B` is a range.
- `git show` displays one commit; `git show <commit>:<file>` displays a file at that time.
- A diff has `-` (removed) and `+` (added) lines, context lines and `@@` hunk headers.
- `git diff` (unstaged), `git diff --staged` (staged), `git diff HEAD` (all) and `git diff A B` (two commits).
- `git blame` finds the commit of each line; `git log -S` and `git grep` search history and files.
- `git bisect` (with `run`) finds the commit that introduced a bug by binary search.

## Sources and further reading

- [Pro Git — 2.3 Viewing the Commit History](https://git-scm.com/book/en/v2/Git-Basics-Viewing-the-Commit-History) — all options of `git log`.
- [git-log manual](https://git-scm.com/docs/git-log) — the complete reference, including format placeholders.
- [git-diff manual](https://git-scm.com/docs/git-diff) — every way to compare.
- [Pro Git — 7.1 Revision Selection](https://git-scm.com/book/en/v2/Git-Tools-Revision-Selection) — all the ways to name commits (`~`, `^`, ranges).
- [Pro Git — 7.10 Debugging with Git](https://git-scm.com/book/en/v2/Git-Tools-Debugging-with-Git) — `blame` and `bisect` in detail.

---

[🏠 Start](../START-HERE.md) · [← 01.3 Your first repository and commits](03-your-first-repository-and-commits.md) · [📋 Phase 01](README.md) · [01.5 Undoing changes safely →](05-undoing-changes-safely.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
