<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 01.2 Installing and configuring Git](02-install-and-configure-git.md) · [📋 Phase 01](README.md) · [01.4 Viewing history and changes →](04-viewing-history-and-changes.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/01-git-and-version-control/03-your-first-repository-and-commits.md) <!-- nav -->

# Lesson 01.3 — Your first repository and commits

> **Phase 01 — Git and version control** · Lesson 3 of 10 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How to create a repository with `git init` and what the hidden `.git` folder is.
- The life of a file: untracked, modified, staged, committed.
- How to read `git status`, and how to use `git add` and `git commit`.
- How to write a good commit message.
- How to keep files out of history with `.gitignore`, and how this works with a Rust project made by `cargo new`.

## Before you start

- You finished [Lesson 01.2 — Installing and configuring Git](02-install-and-configure-git.md). `git config --global --list` shows your name and email.
- The folder `~/projects/git-playground` exists (you created it in the previous lesson). If not, run `mkdir -p ~/projects/git-playground`.
- Rust and Cargo work: `cargo --version` prints a version ([Lesson 00.5](../00-getting-started/05-install-rust-and-rustrover.md)).

## New words

| Word | Meaning |
|---|---|
| **`git init`** | The command that turns a folder into a repository. |
| **`.git` folder** | The hidden folder inside a repository where Git keeps all history. If you delete it, the history is gone. |
| **working tree** | The files and folders you see and edit. |
| **tracked file** | A file that Git already knows (it is in the last commit or in the staging area). |
| **untracked file** | A new file that Git has not been told to follow yet. |
| **staging area** (also *index*) | The place where you collect the changes that will go into the next commit. |
| **stage** (verb) | To put a change into the staging area with `git add`. |
| **`HEAD`** | A pointer that means "the commit I am standing on now", usually the latest commit of the current branch. |
| **commit message** | A short text that explains why a commit was made. |
| **author** | The name and email stored in a commit. |
| **`.gitignore`** | A text file with patterns of files that Git must not track. |
| **pattern** | A rule such as `*.tmp` that matches many file names. `*` means "anything". |
| **Cargo.lock** | A file made by Cargo that records the exact versions of the libraries a project used. |
| **TOML** (Tom's Obvious Minimal Language) | A simple text format for settings. `Cargo.toml` is written in it. |
| **SHA-1** (Secure Hash Algorithm 1) | The hash function Git uses to name commits. |

## 1. Create a repository

A **repository** is a normal folder plus a hidden folder `.git`. Let us make one.

```bash
cd ~/projects/git-playground
mkdir hello-git
cd hello-git
git init
```

```text
Initialized empty Git repository in /Users/ada/projects/git-playground/hello-git/.git/
```

**What happens:**

1. You go to the practice folder, create `hello-git` and enter it.
2. `git init` creates the hidden folder `.git` inside. From now on, this folder is a repository.
3. Git prints where the repository is. Your path will show your own name.

Ask Git what it sees:

```bash
git status
```

```text
On branch main

No commits yet

nothing to commit (create/copy files and use "git add" to track)
```

**What happens:** `git status` is the command you will run the most. It tells you three things: which branch you are on (`main`), that there are no commits yet, and that nothing needs saving.

> ⚠️ **Common mistake:** do not create a repository inside another repository. If you run `git init` in a folder that is already inside a repository, you get a "nested" repository, and the outer one gets confused. Run `git status` first: if it says `fatal: not a git repository`, you are free to run `git init`.

### What is inside `.git`?

```bash
ls -a
ls .git
```

```text
.
..
.git
```

```text
HEAD
config
description
hooks
info
objects
refs
```

(After your first commit a few more entries appear. They are not important.) You never edit these files by hand. Just remember two facts: **the whole history lives in `.git`**, and **deleting `.git` deletes the history** (your normal files stay).

## 2. The life of a file

A file in a repository is always in one of four states. Learn this picture; the whole of Git rests on it.

```text
 untracked ──git add──▶ staged ──git commit──▶ committed (clean)
                           ▲                        │
                           │                        │ you edit the file
                           └───git add──── modified ◀┘
```

| State | Meaning | How it appears in `git status` |
|---|---|---|
| **untracked** | New file. Git does not follow it. | listed under "Untracked files" |
| **staged** | Its current content is chosen for the next commit. | listed under "Changes to be committed" |
| **committed** | Its current content is saved in history. Nothing to do. | not listed (the tree is "clean") |
| **modified** | It is tracked, and you changed it after the last commit. | listed under "Changes not staged for commit" |

Why does Git have the extra step `git add`? Because you decide **what goes into each commit**. Suppose you fixed a bug and also started another feature in the same hour. With staging you can put the fix into one commit and the feature into another. Small, focused commits are much easier to read and undo.

## 3. Your first commit

Create a file. Use RustRover (*File → New → File*) or write it from the terminal:

```bash
echo "# hello-git" > README.md
echo "" >> README.md
echo "My first repository." >> README.md
git status
```

```text
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	README.md

nothing added to commit but untracked files present (use "git add" to track)
```

**What happens:**

1. The three `echo` lines create `README.md` with three lines: a title, an empty line and a sentence. (`>` creates or replaces the file, `>>` adds to the end.)
2. `git status` shows `README.md` as **untracked**. Git sees it but does not follow it. It even tells you the next command.

Stage the file:

```bash
git add README.md
git status
```

```text
On branch main

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
	new file:   README.md

```

**What happens:** `git add README.md` copies the current content of the file into the staging area. The file is now **staged**. The status says "Changes to be committed": this is exactly the list of what the next commit will contain.

Commit:

```bash
git commit -m "Add README"
```

```text
[main (root-commit) 4fb5aaf] Add README
 1 file changed, 3 insertions(+)
 create mode 100644 README.md
```

**What happens:**

1. `git commit` takes everything in the staging area and saves it as a new commit.
2. `-m "Add README"` gives the message on the command line. Without `-m`, Git opens your editor (`nano`) so you can type a longer message: type it, press `Ctrl+O` then `Enter` to save, and `Ctrl+X` to quit.
3. The output says: branch `main`; `root-commit` means "this is the very first commit, with no parent"; `4fb5aaf` is the short hash (the first 7 characters of the long one); then the message; then a summary (1 file, 3 lines added). `100644` is the file mode: a normal, non-executable file.

Your hash will be different. That is normal. It depends on your name, the time and the content.

```bash
git status
git log
```

```text
On branch main
nothing to commit, working tree clean
```

```text
commit 4fb5aafe1d1ebe8ae283f983329fbd147d8307bb
Author: Ada Learner <ada@example.com>
Date:   Tue Nov 17 16:22:18 2026 +0500

    Add README
```

**What happens:** `working tree clean` means every tracked file is exactly as in the last commit. `git log` shows the history: the full 40-character hash (SHA-1), the author, the date and the message. If the output fills the screen, Git shows it in a pager: press `q` to leave.

## 4. The edit → add → commit rhythm

This is what you will do thousands of times. Edit the file:

```bash
echo "It tracks my first steps with Git." >> README.md
git status
```

```text
On branch main
Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git restore <file>..." to discard changes in working directory)
	modified:   README.md

no changes added to commit (use "git add" and/or "git commit -a")
```

See what exactly changed with `git diff` (you will study it in the next lesson):

```bash
git diff
```

```text
diff --git a/README.md b/README.md
index 6aafd58..c4d9981 100644
--- a/README.md
+++ b/README.md
@@ -1,3 +1,4 @@
 # hello-git
 
 My first repository.
+It tracks my first steps with Git.
```

Now stage and commit:

```bash
git add README.md
git commit -m "Describe the project in README"
git log --oneline
```

```text
[main ee34cdb] Describe the project in README
 1 file changed, 1 insertion(+)
ee34cdb Describe the project in README
4fb5aaf Add README
```

**What happens:** `git log --oneline` prints one line per commit: short hash and message, newest first. Your history now has two commits.

### The short status

`git status --short` (or `-s`) is a compact version. Add two files and see:

```bash
echo 'fn main() {}' > main.rs
echo 'notes' > notes.txt
git status -s
git add main.rs
git status -s
git commit -m "Add empty main.rs"
git status -s
```

```text
?? main.rs
?? notes.txt
A  main.rs
?? notes.txt
[main fa488a1] Add empty main.rs
 1 file changed, 1 insertion(+)
 create mode 100644 main.rs
?? notes.txt
```

**What happens:** each line starts with two letters. The **first** letter is the state in the staging area, the **second** is the state in the working tree.

| Code | Meaning |
|---|---|
| `??` | untracked |
| `A ` | added (staged as a new file) |
| `M ` | modified and staged |
| ` M` | modified, **not** staged |
| `D ` / ` D` | deleted (staged / not staged) |
| `R ` | renamed (staged) |

You committed `main.rs` but not `notes.txt`. This is **selective staging** in action: `git commit` saves only what is staged.

## 5. What is a commit, really?

A commit is a small object with four parts. You can look at the last one:

```bash
git cat-file -p HEAD
```

```text
tree d819d0582756f690ade00d55cf1786e4d3879d2d
parent ee34cdb1ae84df7eccb449400b58d350947f965e
author Ada Learner <ada@example.com> 1794914538 +0500
committer Ada Learner <ada@example.com> 1794914538 +0500

Add empty main.rs
```

**What happens:** `git cat-file -p` prints a Git object in a readable form. `HEAD` means "the current commit". The object has:

- **tree** — the hash of the snapshot: the list of all files and their content at that time;
- **parent** — the hash of the commit before this one (the first commit has none);
- **author / committer** — who wrote it and when (the number is the time in seconds since 1970, then the time zone);
- the **message**.

Because the parent hash is part of the commit, and the commit's own hash is computed from everything above, changing an old commit changes all later hashes. This is why history cannot be silently changed.

```text
 ee34cdb  ◀──────  fa488a1  ◀── HEAD
 (parent)           (this commit)
```

## 6. Good commit messages

A message is for your future self and for teammates. Compare:

| Bad | Good |
|---|---|
| `fix` | `Fix crash when the input file is empty` |
| `changes` | `Add README with build instructions` |
| `asdf` | `Rename Config to Settings` |
| `final version 2` | `Print the total at the end of the report` |

📏 **Convention:** the rules that most teams follow:

1. The first line (the **subject**) is up to about 50 characters, with no period at the end.
2. Write it in the **imperative mood**, like an order: "Add", "Fix", "Remove" — not "Added" or "Fixing". A good test: the sentence *"If applied, this commit will … <subject>"* must make sense.
3. If you need more words, leave an **empty line** after the subject and write the body: explain **why** the change was made, not what (the diff shows what). Wrap lines at about 72 characters.
4. One commit = one logical change. Do not mix a bug fix and a formatting change.

For a longer message, run `git commit` without `-m`. The editor opens with an empty first line; type the subject, an empty line, then the body. You will see more message conventions in [Lesson 01.9](09-teamwork-pull-requests-and-conventions.md).

> 💡 **Tip:** commit often. A good rhythm is one commit every time a small piece works. Commits are cheap, and they are your save points.

## 7. `.gitignore`: files Git must not track

Some files should never go into history: build output, temporary files, secrets. You list their patterns in a file named `.gitignore` at the root of the repository.

```bash
printf '*.tmp\nbuild/\n' > .gitignore
mkdir build
touch build/out.bin cache.tmp
git status -s
```

```text
?? .gitignore
?? notes.txt
```

**What happens:** two files, `build/out.bin` and `cache.tmp`, exist, but Git does not list them. They match the patterns, so Git acts as if they were not there. The new `.gitignore` itself is untracked: you **do** commit it, so that everybody gets the same rules.

```bash
git add .gitignore
git commit -m "Ignore temporary files and build output"
```

The pattern rules you need now:

| Pattern | Matches |
|---|---|
| `*.tmp` | any file that ends with `.tmp`, in any folder |
| `build/` | a folder named `build` (the slash at the end means "folder"), anywhere |
| `/target` | a `target` folder, **only** at the top level of the repository (the leading slash anchors it) |
| `!keep.tmp` | an exception: do **not** ignore this file even if another pattern matches |
| `# text` | a comment |

> ⚠️ **Common mistake:** `.gitignore` only affects files that are **not yet tracked**. If you already committed `old.tmp`, adding `*.tmp` later does nothing for it. To stop tracking a file but keep it on disk, run `git rm --cached old.tmp` and commit:

```bash
git rm --cached old.tmp
git commit -m "Stop tracking old.tmp"
```

```text
rm 'old.tmp'
[main 4e0cbfc] Stop tracking old.tmp
 1 file changed, 1 deletion(-)
 delete mode 100644 old.tmp
```

(The file stays in your folder; only Git forgets it. The old content remains in earlier commits.)

> 🔒 **Security:** put files with secrets in `.gitignore` **before** the first commit: `.env`, `*.pem`, `*.key`, `secrets.toml`. A secret that was committed once stays in the history even after you delete the file, and anyone who can read the repository can find it. If it happens, treat the secret as stolen: create a new one and cancel the old one. You will practise a safe way in [Lesson 01.5](05-undoing-changes-safely.md) and [Lesson 01.8](08-remotes-github-and-gitlab.md).

## 8. Git and a Rust project

Now let us put a real Rust project under Git. Cargo helps: when you create a project **outside** of any repository, `cargo new` runs `git init` for you.

```bash
cd ~/projects/git-playground
cargo new hello-rust
cd hello-rust
ls -a
cat .gitignore
```

```text
    Creating binary (application) `hello-rust` package
note: see more `Cargo.toml` keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html
.
..
.git
.gitignore
Cargo.toml
src
```

```text
/target
```

**What happens:**

1. `cargo new hello-rust` makes a project with a `Cargo.toml` (the settings file, in the TOML format), a `src` folder with `main.rs`, **and** a `.git` folder and a `.gitignore` file.
2. The `.gitignore` contains one pattern, `/target`. The `target` folder is where Cargo puts everything it builds. It can be hundreds of megabytes and it can be rebuilt from your code at any time, so it must never be committed.

Build and look at the status:

```bash
cargo build
git status
```

```text
   Compiling hello-rust v0.1.0 (/Users/ada/projects/git-playground/hello-rust)
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.17s
On branch main

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
	.gitignore
	Cargo.lock
	Cargo.toml
	src/

nothing added to commit but untracked files present (use "git add" to track)
```

**What happens:** the build created the `target` folder and a new file `Cargo.lock`. Git lists `Cargo.lock` but not `target`, because `target` is ignored. `Cargo.lock` records the exact versions of the libraries that were used. **Commit it** for programs, so that everybody builds exactly the same thing (you will learn more in Lesson 05.8).

Commit the project:

```bash
git add .
git commit -m "Create the project with cargo new"
git ls-files
git check-ignore -v target
```

```text
[main (root-commit) a43950e] Create the project with cargo new
 4 files changed, 17 insertions(+)
 create mode 100644 .gitignore
 create mode 100644 Cargo.lock
 create mode 100644 Cargo.toml
 create mode 100644 src/main.rs
.gitignore
Cargo.lock
Cargo.toml
src/main.rs
.gitignore:1:/target	target
```

**What happens:** `git add .` stages everything in the current folder (the dot means "here"). `git ls-files` lists the files Git tracks: there is no `target` in the list. `git check-ignore -v target` explains why: line 1 of `.gitignore`.

> ⚠️ **Common mistake:** `git add .` stages *everything that is not ignored*, including files you did not mean to commit (a `.env` file, a big test file). **Always run `git status` before `git commit`,** and read the list.

> 📏 **Convention:** commit `Cargo.toml`, `Cargo.lock` (for programs), `src/` and the project's `.gitignore`. Never commit `target/`.

If you create a project **inside** a folder that is already a repository, `cargo new` does not create another repository. There is one trap with this; you will meet it in [Lesson 01.10](10-project-your-learning-repo-and-checkpoint.md).

## 9. Removing and renaming files

If you delete or rename a file with the normal commands, Git sees a change and you stage it. Git also has its own commands that do two steps at once.

```bash
echo "a" > old-name.txt
git add old-name.txt
git commit -m "Add old-name.txt"
git mv old-name.txt new-name.txt
git status -s
git commit -m "Rename old-name.txt to new-name.txt"
git rm new-name.txt
git commit -m "Remove new-name.txt"
```

```text
[main 2bc8f1b] Add old-name.txt
 1 file changed, 1 insertion(+)
 create mode 100644 old-name.txt
R  old-name.txt -> new-name.txt
[main ac4f9ef] Rename old-name.txt to new-name.txt
 1 file changed, 0 insertions(+), 0 deletions(-)
 rename old-name.txt => new-name.txt (100%)
rm 'new-name.txt'
[main eb766c8] Remove new-name.txt
 1 file changed, 1 deletion(-)
 delete mode 100644 new-name.txt
```

**What happens:**

1. `git mv a b` renames (or moves) the file and stages the change. `R` in the status means "renamed". Git does not store the rename; it *detects* it because the content is the same (`100%`).
2. `git rm file` deletes the file from your folder **and** stages the deletion.
3. The file still exists in older commits, so deleting a file never loses history.

Git does not track **empty folders**, only files. If you need an empty folder in the repository, put an empty file in it, by convention named `.gitkeep`.

## Common mistakes

> ⚠️ **Common mistake:** you changed a file, ran `git commit -m "..."`, and Git says "no changes added to commit". You forgot `git add`. Run `git status`: the file is under "Changes not staged for commit". Stage it and commit again. (Or use `git commit -a -m "..."`, which stages all changes to *already tracked* files before committing. It does not include new files.)

> ⚠️ **Common mistake:** `nothing to commit, working tree clean`. There is nothing new to save. This is not an error; Git exits with code 1 only to say that no commit was made.

> ⚠️ **Common mistake:** a very long commit message typed with `-m` and several quotes. Use `git commit` without `-m` and write in the editor instead.

> ⚠️ **Common mistake:** a giant commit with a week of work and the message "work". When something breaks, you cannot find which part is guilty. Commit small and often.

> ⚠️ **Common mistake:** committing `target/`, `.DS_Store` or `.idea/`. Create the ignore rules at the start of a project ([Lesson 01.2](02-install-and-configure-git.md) covers the last two globally).

## Try it yourself

Follow every step and compare with the expected result.

1. `cd ~/projects/git-playground && mkdir recipes && cd recipes && git init`.
2. Create the file `pancakes.txt` with two lines (an ingredient on each line), using `echo` and `>>`.
3. Run `git status -s`. Expect `?? pancakes.txt`.
4. Run `git add pancakes.txt`, then `git status -s`. Expect `A  pancakes.txt`.
5. Commit with the message `Add pancakes recipe`.
6. Add a third ingredient to the file. Run `git status -s`. Expect ` M pancakes.txt` (a space, then `M`).
7. Run `git diff` and find the line that starts with `+`.
8. Commit with `Add a third ingredient to pancakes`.
9. Run `git log --oneline`. Expect two commits, newest first.
10. Create `scratch.tmp`, and a `.gitignore` with the line `*.tmp`. Check that `git status -s` shows only `?? .gitignore`. Commit it with a good message.

## Exercises

### Exercise 1 — First commit (easy)

Create a repository `todo-list` in `~/projects/git-playground`. Make a file `todo.txt` with two lines (`buy milk` and `write lesson`) and commit it with the message `Add the first todo list`. Show the history.

<details>
<summary>Hint</summary>

Use `git init`, a file, `git add`, `git commit -m`, `git log --oneline`.

</details>

<details>
<summary>Answer</summary>

```bash
cd ~/projects/git-playground
mkdir todo-list
cd todo-list
git init
echo "buy milk" > todo.txt
echo "write lesson" >> todo.txt
git add todo.txt
git commit -m "Add the first todo list"
git log --oneline
```

```text
[main (root-commit) e0aba58] Add the first todo list
 1 file changed, 2 insertions(+)
 create mode 100644 todo.txt
e0aba58 Add the first todo list
```

Why it works: `git add` puts the file into the staging area and `git commit` saves it as the first commit. The hash on your computer will be different.

</details>

### Exercise 2 — Read the status (easy)

In `todo-list` you committed `README.md` and `todo.txt`. Then you added a line to `README.md` and created a new file `notes.txt`. What does `git status -s` print? Check on your computer.

<details>
<summary>Hint</summary>

One file is tracked and changed, the other is new. Remember the two-letter codes.

</details>

<details>
<summary>Answer</summary>

```text
 M README.md
?? notes.txt
```

` M` (space, then M) means "modified but not staged" (the first column is the staging area, which is empty). `??` means "untracked".

Why it works: the first letter describes the staging area and the second letter describes the working tree.

</details>

### Exercise 3 — Commit only one of two changes (medium)

In `todo-list`, change both `README.md` and `todo.txt`. Make **two separate commits**: first only the README change (message `Describe the project in README`), then the todo change (message `Add a todo item`). Show `git status -s` after the first commit and the final `git log --oneline`.

<details>
<summary>Hint</summary>

Stage only one file with `git add README.md`. Use `git commit -a` for the second commit, or `git add todo.txt`.

</details>

<details>
<summary>Answer</summary>

```bash
echo "A small project." >> README.md
echo "call the doctor" >> todo.txt
git add README.md
git commit -m "Describe the project in README"
git status -s
git commit -am "Add a todo item"
git log --oneline
```

```text
[main 25df0e4] Describe the project in README
 1 file changed, 1 insertion(+)
 M todo.txt
[main c61e27a] Add a todo item
 1 file changed, 1 insertion(+)
c61e27a Add a todo item
25df0e4 Describe the project in README
6dc73b4 Add README
e0aba58 Add the first todo list
```

Why it works: only the staged file (`README.md`) goes into the first commit. `todo.txt` stays modified (` M`) until the second commit. `-a` stages every modified tracked file; `-am` combines `-a` and `-m`.

</details>

### Exercise 4 — Ignore and untrack (medium)

In `todo-list`, create `.gitignore` with the patterns `*.tmp` and `build/`. Prove that `build/out.bin` and `cache.tmp` are ignored. Then simulate a mistake: force-add `old.tmp` (`git add -f old.tmp`) and commit it, then make Git stop tracking it **without** deleting the file.

<details>
<summary>Hint</summary>

`git status -s` shows what Git sees. To stop tracking a file but keep it, use `git rm --cached`.

</details>

<details>
<summary>Answer</summary>

```bash
printf '*.tmp\nbuild/\n' > .gitignore
mkdir build
touch build/out.bin cache.tmp
git status -s
git add .gitignore
git commit -m "Ignore temporary files and build output"
echo data > old.tmp
git add -f old.tmp
git commit -m "Oops, tracked a tmp file"
git rm --cached old.tmp
git commit -m "Stop tracking old.tmp"
ls old.tmp
git ls-files
```

The first `git status -s` shows only `?? .gitignore` (and any other new files, such as `notes.txt`); `build/out.bin` and `cache.tmp` are not listed. At the end `ls old.tmp` still finds the file, and `git ls-files` does not list it.

Why it works: `-f` (force) adds an ignored file. `git rm --cached` removes the file from the staging area only, so the next commit records its deletion from the history's point of view, while your disk copy stays.

</details>

### Exercise 5 — A Rust project with a clean history (hard)

Create a Rust project `calc` with `cargo new calc`, inside `~/projects/git-playground`. Make **three commits** with good messages: (1) the project created by Cargo; (2) a changed greeting in `src/main.rs` (change the text inside `println!`); (3) a `README.md`. Then show the three commits with `git log --oneline` and prove that `target/` is not tracked.

<details>
<summary>Hint</summary>

`cargo new` already runs `git init`. Build or run the program once so that `target/` exists, then use `git ls-files`.

</details>

<details>
<summary>Answer</summary>

```bash
cd ~/projects/git-playground
cargo new calc
cd calc
cargo build
git add .
git commit -m "Create the project with cargo new"
sed -i '' 's/Hello, world!/Hello from calc!/' src/main.rs
cargo run
git commit -am "Change the greeting"
echo "# calc" > README.md
git add README.md
git commit -m "Add README"
git log --oneline
git ls-files
```

```text
Hello from calc!
```

(The `git log --oneline` output has three lines with your own hashes: `Add README`, `Change the greeting`, `Create the project with cargo new`. `git ls-files` lists `.gitignore`, `Cargo.lock`, `Cargo.toml`, `README.md` and `src/main.rs` — no `target`.)

Why it works: `cargo new` made the repository and a `.gitignore` with `/target`. `cargo build` and `cargo run` build into `target/`, which Git ignores. Building before the first `git add .` creates `Cargo.lock`, so it is committed too. The `sed` command replaces text in the file; you can also edit the file in RustRover. On macOS, `sed -i ''` edits the file in place.

</details>

## Quiz

1. What does `git init` create?

<details>
<summary>Answer</summary>

A hidden `.git` folder in the current folder, which turns it into a repository.

</details>

2. What is the difference between `git add` and `git commit`?

<details>
<summary>Answer</summary>

`git add` puts changes into the staging area (chooses what will be saved). `git commit` saves the staged changes as a new commit in the history.

</details>

3. What does `?? notes.txt` mean in `git status -s`?

<details>
<summary>Answer</summary>

`notes.txt` is untracked: a new file that Git does not follow yet.

</details>

4. Why should a commit message be in the imperative mood and explain *why*?

<details>
<summary>Answer</summary>

The imperative ("Fix …", "Add …") is the common convention and reads like a task done by the commit. The *why* is important because the diff already shows *what* changed, but not the reason.

</details>

5. You added `*.tmp` to `.gitignore`, but Git still tracks `old.tmp`. Why, and what is the fix?

<details>
<summary>Answer</summary>

`.gitignore` only affects untracked files. Run `git rm --cached old.tmp` and commit; the file stays on your disk but is no longer tracked.

</details>

6. Which Cargo files and folders do you commit, and which one never?

<details>
<summary>Answer</summary>

Commit `Cargo.toml`, `Cargo.lock` (for programs), `src/` and `.gitignore`. Never commit `target/`.

</details>

7. What does `git status` say when everything is committed?

<details>
<summary>Answer</summary>

`nothing to commit, working tree clean`.

</details>

## Summary

- `git init` makes a repository; the history lives in the hidden `.git` folder.
- A file is untracked, modified, staged or committed. `git add` stages, `git commit` saves.
- `git status` (and `git status -s`) is your map; run it before every commit.
- A commit has a snapshot, a parent, an author and a message; its hash identifies it.
- Good messages: a short imperative subject, an optional body that explains *why*, one logical change per commit.
- `.gitignore` keeps build output, temporary files and secrets out of the history, but only for untracked files.
- For Rust, commit `Cargo.toml`, `Cargo.lock` and `src/`; never `target/`.

## Sources and further reading

- [Pro Git — 2.1 Git Basics: Getting a Git Repository](https://git-scm.com/book/en/v2/Git-Basics-Getting-a-Git-Repository) — `git init` and `git clone`.
- [Pro Git — 2.2 Recording Changes to the Repository](https://git-scm.com/book/en/v2/Git-Basics-Recording-Changes-to-the-Repository) — the file life cycle in depth, `.gitignore`, `git rm`, `git mv`.
- [gitignore manual](https://git-scm.com/docs/gitignore) — the exact pattern rules.
- [The Cargo Book — Creating a New Package](https://doc.rust-lang.org/cargo/guide/creating-a-new-project.html) — what `cargo new` creates, including the VCS (Version Control System) option.
- [The Cargo Book — Cargo.toml vs Cargo.lock](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html) — when to commit `Cargo.lock`.

---

[🏠 Start](../START-HERE.md) · [← 01.2 Installing and configuring Git](02-install-and-configure-git.md) · [📋 Phase 01](README.md) · [01.4 Viewing history and changes →](04-viewing-history-and-changes.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
