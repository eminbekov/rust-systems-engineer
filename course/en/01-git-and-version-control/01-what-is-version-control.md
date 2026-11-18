<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 01 — Git and version control](README.md) · [📋 Phase 01](README.md) · [01.2 Installing and configuring Git →](02-install-and-configure-git.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/01-git-and-version-control/01-what-is-version-control.md) <!-- nav -->

# Lesson 01.1 — What version control is

> **Phase 01 — Git and version control** · Lesson 1 of 10 · ⏱ about 105 minutes

## What you will learn

- What problem version control solves, and why copying folders ("`project_final_v2_REAL`") is not a solution.
- What a repository, a commit, a history and a remote are, in plain words.
- The three kinds of version control systems, and why Git is called "distributed".
- The difference between Git, GitHub and GitLab.
- Why every systems engineer, and every Rust project, uses Git every day.

## Before you start

- You finished [Lesson 00.4 — The terminal: your first commands](../00-getting-started/04-terminal-basics.md). You can open a terminal, move between folders and list files.
- You finished [Lesson 00.8 — Study routine, notes in Markdown and checkpoint](../00-getting-started/08-study-routine-notes-and-checkpoint.md).
- You do not need Git yet. In this lesson you will do version control **by hand**, so you feel the pain that Git removes. You install Git in the next lesson.

## New words

| Word | Meaning |
|---|---|
| **version** | One state of your files at one moment in time. "The essay as it was on Monday" is one version. |
| **version control** | A way to record every version of your files, so you can see what changed, who changed it, why, and go back. Also called *source control* or VC. |
| **VCS** (Version Control System) | A program that does version control. Git is a VCS. |
| **repository** (repo) | A folder whose history is recorded by a VCS. It holds your files and all their saved versions. |
| **commit** | One saved version in the history, together with a short message that says why you saved it. |
| **history** | The list of all commits, from the first to the latest. |
| **snapshot** | A picture of all your project files at one moment. A commit is a snapshot. |
| **diff** (difference) | The list of lines that were added, removed or changed between two versions. |
| **hash** | A short "fingerprint" computed from the content of a file. Same content gives the same hash; any change gives a completely different hash. |
| **SHA** (Secure Hash Algorithm) | A family of hash functions. Git uses SHA-1 by default (and can use SHA-256). |
| **remote** | A copy of your repository stored on another computer, for example on GitHub. |
| **branch** | A separate line of work inside a repository. You will learn it in [Lesson 01.6](06-branches-and-merging.md). |
| **SVN** (Subversion) | An older, *centralized* version control system. |
| **CLI** (Command-Line Interface) | A way to use a program by typing commands in a terminal. |
| **GUI** (Graphical User Interface) | A way to use a program with windows, buttons and the mouse. |
| **IDE** (Integrated Development Environment) | A program for writing code with many helpers built in. RustRover is an IDE. |

## 1. The problem: files change

Imagine you write an essay about Rust. On Monday it has one paragraph. On Tuesday you add a second one. On Wednesday you rewrite the first one, and you notice that the Monday version was better.

Without any tool, you can only copy the file before each change. Soon your folder looks like this:

```text
essay.txt
essay-v2.txt
essay-v2-final.txt
essay-v2-final-REAL.txt
essay-v2-final-REAL-fixed-monday-version.txt
```

Which file is the newest? What is different between them? Why did you create the fourth one? You will not remember in a week.

Now think about a real software project. It has hundreds of files. Five people change them at the same time. A bug appears and nobody knows which change caused it. Copying folders cannot work here.

A version control system answers these questions:

| Question | What the tool gives you |
|---|---|
| What did the project look like last Tuesday? | Go back to any saved version. |
| What exactly changed in this file? | A *diff* between two versions. |
| Who changed this line, and why? | The author, date and message of every commit. |
| I broke everything. Can I undo? | Return to the last good version. |
| Two people edit the same file. Whose work wins? | Tools to combine both changes. |
| How do I share my work with the team? | Send the history to a remote and fetch the work of others. |

> 💡 **Tip:** think of version control as "unlimited save points" in a video game, plus a lab notebook that remembers who did what and why.

## 2. Do it by hand first

Let us do version control manually. You will use terminal commands from [Lesson 00.4](../00-getting-started/04-terminal-basics.md) and two new ones.

Make a practice folder and a file:

```bash
mkdir -p ~/projects/vc-by-hand
cd ~/projects/vc-by-hand
echo "Rust is a systems language." > essay.txt
```

**What happens:**

1. `mkdir -p ~/projects/vc-by-hand` creates the folder. The flag `-p` means "also create missing parent folders, and do not complain if it exists".
2. `cd` moves you into it.
3. `echo "..." > essay.txt` prints the text, but `>` redirects it into the file `essay.txt` instead of the screen. If the file exists, `>` **replaces** its content. (`>>` would **add** to the end.)

Now save "version 1" by copying, change the file, and save "version 2":

```bash
cp essay.txt essay-v1.txt
echo "It has no garbage collector." >> essay.txt
cp essay.txt essay-v2.txt
echo "Rust is a fast systems language." > essay.txt
echo "It has no garbage collector." >> essay.txt
ls -1
```

**What happens:**

1. `cp essay.txt essay-v1.txt` copies the file. This copy is "version 1".
2. `>>` adds a second line to `essay.txt`.
3. Another copy is "version 2".
4. The last two `echo` lines rewrite the file: the first line changes, the second line stays. This is the "current" version.
5. `ls -1` lists the files, one per line.

```text
essay-v1.txt
essay-v2.txt
essay.txt
```

To see what changed, use the program `diff`. It compares two files line by line.

```bash
diff essay-v1.txt essay.txt
```

```text
1c1,2
< Rust is a systems language.
---
> Rust is a fast systems language.
> It has no garbage collector.
```

**What happens:** `diff` prints the changes needed to turn the first file into the second one. `1c1,2` means "line 1 of the first file was **c**hanged into lines 1–2 of the second". Lines that start with `<` come from the first file, lines that start with `>` come from the second file.

A more common format is the *unified diff*. You will see it in Git all the time:

```bash
diff -u essay-v1.txt essay.txt
```

```text
--- essay-v1.txt	2026-11-17 16:21:22
+++ essay.txt	2026-11-17 16:21:22
@@ -1 +1,2 @@
-Rust is a systems language.
+Rust is a fast systems language.
+It has no garbage collector.
```

**What happens:** lines that start with `-` were removed, lines that start with `+` were added. The line `@@ -1 +1,2 @@` says where the change is: "from line 1 of the old file" and "from line 1, two lines, of the new file".

Your own dates will be different. That is fine.

### Why this does not scale

You just did the work of a version control system, and already you can see the problems:

- **Names carry the meaning.** `essay-v2` tells nothing about *why*. Nobody wrote down "added the garbage collector sentence".
- **Space.** Every version is a full copy, even when only one line changed.
- **Easy to break.** You can overwrite `essay-v1.txt` by mistake. Nothing warns you.
- **No authors and no dates** you can trust.
- **No safe way to combine** the work of two people.

Git solves all of these. Let us look at the first idea it uses.

## 3. Fingerprints: how a tool knows a file changed

How can a tool be sure that two files are identical, or that a file was damaged? It computes a **hash**. A hash function reads all the bytes of a file and produces a short string. macOS has a program for this, `shasum`:

```bash
shasum -a 256 essay-v1.txt essay.txt
cp essay.txt essay-copy.txt
shasum -a 256 essay-copy.txt
```

```text
39331024fe2fc799466a9aef292914d32b5c788ba2e63f09e42bdd227f641031  essay-v1.txt
a59be4c9143e0cae7f2748fe2f3191ab0306a020528fc896ed95a8c540f5f542  essay.txt
a59be4c9143e0cae7f2748fe2f3191ab0306a020528fc896ed95a8c540f5f542  essay-copy.txt
```

**What happens:**

1. `shasum -a 256` computes the SHA-256 hash of each file and prints it, followed by the file name.
2. `essay.txt` and its copy `essay-copy.txt` have **exactly the same** hash, because the content is the same. The name does not matter.
3. `essay-v1.txt` has a **completely different** hash, because its content is different.

Even a tiny change gives a totally different result. Compare the SHA-1 hashes of the text `abc` and the text `abc ` (with a space at the end):

```bash
printf 'abc' | shasum -a 1
printf 'abc ' | shasum -a 1
```

```text
a9993e364706816aba3e25717850c26c9cd0d89d  -
eeca658f56ce50a4f36605aaf93e29e87c12009a  -
```

(`printf` prints text without adding a new line; `|` sends it into `shasum`; the `-` means "this came from the pipe, not from a file".)

Git uses hashes everywhere. Every saved version gets a name that **is** the hash of its content, like `4fb5aafe1d1ebe8ae283f983329fbd147d8307bb`. That is why Git can find changes quickly, and why a damaged or secretly modified history is detected at once. You do not need to remember more than this now: **a commit is named by a long hash, and you usually use only its first 7 characters.**

> 📏 **Convention:** Git uses SHA-1 by default and is slowly moving to SHA-256. In this course the hashes you see in Git output are 40 hexadecimal characters (SHA-1).

## 4. Three kinds of version control systems

Version control systems developed in three steps.

**1. Local.** The history lives in a hidden folder on your own computer. Simple, but you cannot work with other people.

**2. Centralized.** One **server** holds the one true history. Developers *check out* files from the server, change them, and *commit* back to the server. Examples: **CVS** (Concurrent Versions System), **Subversion (SVN)**, Perforce. The weakness: if the server is down or you are offline, you cannot save history, and if the server disk dies, the history can be lost.

**3. Distributed.** *Every* developer has a **full copy of the whole history**. A server (GitHub, for example) is just one more copy that everybody agrees to use. Examples: **Git**, Mercurial.

```text
 Centralized                            Distributed

 Alice   Bob   Carol                    Alice          Bob
   \      |     /                      [full history] [full history]
    \     |    /                           \            /
   [ one server ]                           [ shared server ]
   (only copy of history)                       |
                                           Carol [full history]
```

Because every copy is complete, in Git you can:

- save history, view it and go back **without internet**;
- work fast, because almost every command only touches your own disk;
- lose the server and still recover everything from any developer's copy.

## 5. What is Git?

**Git** is a free, open-source distributed version control system. Linus Torvalds, the creator of Linux, started it in 2005 to manage the development of the Linux kernel (the core of the Linux operating system). Today it is the standard tool for almost all software.

How does Git think about your project? It stores **snapshots**. Each commit is a picture of *all* tracked files at one moment, with a message and an author. (Inside, Git is clever: if a file did not change, the new snapshot just points to the old stored copy, so no space is wasted.)

```text
commit 1          commit 2          commit 3
[A1] [B1] [C1] -> [A2] [B1] [C1] -> [A2] [B2] [C1]
 file A, B, C      A changed         B changed
                   (B, C reused)     (A, C reused)
```

Each commit also remembers its **parent** (the commit before it). Following the parents gives you the history.

## 6. Git, GitHub and GitLab are not the same

This is the most common confusion for beginners.

| Name | What it is |
|---|---|
| **Git** | The program on your computer. It records history. It works with no internet and no account. |
| **GitHub** | A website and service that stores Git repositories online, and adds teamwork tools: pull requests, issues, automatic checks. Owned by Microsoft. |
| **GitLab** | Another such service. You can use the company's website or install GitLab on your own servers. |
| **Others** | Bitbucket, Codeberg, Gitea and more. They all speak Git. |

An analogy: Git is like a word processor, GitHub is like a cloud drive where you store and share the documents. You can edit documents without the cloud. You will use GitHub and GitLab in [Lesson 01.8](08-remotes-github-and-gitlab.md) and [Lesson 01.9](09-teamwork-pull-requests-and-conventions.md).

## 7. A map of the words you will meet

Here is the picture for the whole phase. Do not worry if it is new; every part has its own lesson.

```text
 your computer                                               another computer
┌──────────────────────────────────────────────────┐        ┌───────────────┐
│  working tree        staging area     repository │        │    remote     │
│  (your files)  --->  (next commit) ---> (history)│ <----> │ (GitHub etc.) │
│   edit files         git add           git commit│  push/ │               │
│                                                  │  pull  │               │
└──────────────────────────────────────────────────┘        └───────────────┘
```

- **Working tree:** the real files you edit.
- **Staging area:** a waiting room where you choose which changes go into the next commit.
- **Repository:** the hidden database of all commits (the folder `.git`).
- **Remote:** a copy on a server. *Push* sends your commits there; *pull* brings the commits of others.

## 8. Why a systems engineer needs this

- **Every team uses Git.** Code review, releases and automatic tests all start from a Git history.
- **Rust lives on Git.** The Rust compiler, Cargo and almost all crates (Rust libraries) are developed in public Git repositories. Cargo can even take a dependency directly from a Git address.
- **Safety net.** With history you can experiment without fear: a bad idea costs one command to undo.
- **Debugging.** When a bug appears, history tells you *which change* introduced it (you will do this in [Lesson 01.4](04-viewing-history-and-changes.md)).
- **Evidence of your work.** A good history is your portfolio. Employers read it.

## Common mistakes

> ⚠️ **Common mistake:** "Git is GitHub." It is not. Git works fully on your laptop. GitHub is only one place to keep a copy.

> ⚠️ **Common mistake:** "Version control is a backup." History inside one folder disappears if the disk dies. A real backup is a copy on **another** machine. A remote helps with that, but it is not the same idea.

> ⚠️ **Common mistake:** "I will learn Git later, when the project is big." Beginners lose work most often in the first small projects. Start now.

> 🔒 **Security:** a history is forever. If you save a password or a secret key in a file and commit it, it stays in the history even after you delete the file. You will learn how to avoid this in [Lesson 01.3](03-your-first-repository-and-commits.md) and [Lesson 01.8](08-remotes-github-and-gitlab.md).

## Try it yourself

Do this in your terminal. It repeats the manual workflow, with a twist.

1. Go to the practice folder: `cd ~/projects/vc-by-hand`.
2. Create a second file: `echo "Cargo is the Rust build tool." > tools.txt`.
3. "Save a version" of it: `cp tools.txt tools-v1.txt`.
4. Change it: `echo "rustup installs Rust." >> tools.txt`.
5. Run `diff -u tools-v1.txt tools.txt`. You should see one `+` line.
6. Run `shasum -a 256 tools-v1.txt tools.txt`. The two hashes must differ.
7. Now undo the change **by hand**: `cp tools-v1.txt tools.txt`. Run `shasum -a 256 tools-v1.txt tools.txt` again. The two hashes are now equal.

You just saw the three basic powers of version control: **see the change**, **detect the change**, **go back**. Git does all three for every file, automatically, with a message and an author.

## Exercises

### Exercise 1 — Classify the tool (easy)

For each item, say whether it is a **VCS**, a **hosting service**, or **neither**: Git, GitHub, Subversion, GitLab, RustRover, Mercurial.

<details>
<summary>Hint</summary>

A VCS records history on your own computer. A hosting service is a website that stores repositories.

</details>

<details>
<summary>Answer</summary>

- VCS: Git, Subversion, Mercurial.
- Hosting service: GitHub, GitLab.
- Neither: RustRover (it is an IDE, a code editor; it can *use* Git, but it is not a VCS).

Why it works: a VCS is the tool that records versions. A hosting service keeps shared copies and adds teamwork tools.

</details>

### Exercise 2 — Find the change (easy)

In `~/projects/vc-by-hand` you have `essay-v2.txt` and `essay.txt`. Show the difference between them with a command, and say in one sentence what changed.

<details>
<summary>Hint</summary>

Use `diff -u` with the older file first.

</details>

<details>
<summary>Answer</summary>

```bash
diff -u essay-v2.txt essay.txt
```

```text
--- essay-v2.txt	2026-11-17 16:21:22
+++ essay.txt	2026-11-17 16:21:22
@@ -1,2 +1,2 @@
-Rust is a systems language.
+Rust is a fast systems language.
 It has no garbage collector.
```

The first line changed: the word "fast" was added. The second line is the same (it starts with a space, which means "unchanged context").

Why it works: `diff -u` shows removed lines with `-`, added lines with `+`, and unchanged lines around them with a space.

</details>

### Exercise 3 — Detect a one-character change (medium)

Create a file `a.txt` with the text `hello`. Copy it to `b.txt`. Prove with hashes that the files are equal. Then add one `!` to the end of `b.txt` and prove that they are now different.

<details>
<summary>Hint</summary>

`echo "hello" > a.txt`, then `cp`, then `shasum -a 256 a.txt b.txt`. To add a character, use `>>` — but `echo` adds a new line, so the file also gets one more line. That is fine; any change is enough.

</details>

<details>
<summary>Answer</summary>

```bash
echo "hello" > a.txt
cp a.txt b.txt
shasum -a 256 a.txt b.txt
echo "!" >> b.txt
shasum -a 256 a.txt b.txt
```

The first `shasum` prints two identical hashes. The second prints two different hashes.

Why it works: the hash is computed from the content only. The same content gives the same hash; any change gives a different one. Git uses exactly this idea to find out what changed.

</details>

### Exercise 4 — Why Git? (hard)

Your team of three shares one network folder and copies files into dated subfolders (`2026-11-01`, `2026-11-02`, …). Write down **four** concrete problems that Git would solve. Use the table in section 1 to help you.

<details>
<summary>Hint</summary>

Think about: two people editing the same file; finding who changed a line; how much disk space is used; going back to one file from last month; working on a train without network.

</details>

<details>
<summary>Answer</summary>

Examples (any four are correct):

1. Two people edit the same file on the same day: one overwrites the other. Git detects this and helps to *merge* both changes.
2. No reason is recorded for a change. In Git every commit has a message and an author.
3. Full copies of everything every day waste disk space. Git stores unchanged files only once.
4. Finding which change introduced a bug means opening many folders by hand. Git can show a diff and even search the history automatically.
5. Without network access you cannot reach the shared folder. In Git you have the full history locally.

Why it works: the manual workflow lacks messages, authors, safe merging and efficient storage. These are the four problems Git was built to solve.

</details>

## Quiz

1. What is a *commit*?

<details>
<summary>Answer</summary>

A saved version of the project (a snapshot of all tracked files) stored in the history, with an author, a date and a message.

</details>

2. Name one advantage of a *distributed* VCS over a *centralized* one.

<details>
<summary>Answer</summary>

Every developer has the full history, so you can commit and view history offline, work fast, and the project survives the loss of any single computer, including the server.

</details>

3. Is Git the same as GitHub?

<details>
<summary>Answer</summary>

No. Git is the program that records history. GitHub is a website that hosts Git repositories and adds teamwork tools.

</details>

4. If two files have the same SHA-256 hash, what do you know about them?

<details>
<summary>Answer</summary>

Their content is the same (the names can be different).

</details>

5. Which of these is *not* part of the picture in section 7: working tree, staging area, repository, compiler?

<details>
<summary>Answer</summary>

The compiler. The other three are the places where your changes live before and after a commit.

</details>

6. Why can a committed password be a problem even if you delete the file later?

<details>
<summary>Answer</summary>

The history keeps every version. Anyone with access to the repository can still read the old version that contains the password.

</details>

## Summary

- Version control records every version of your files with a message and an author, so you can compare and go back.
- Copying folders by hand does not scale: no reasons, no authors, wasted space, unsafe merging.
- A *hash* is a fingerprint of content; Git names every commit by one.
- A distributed VCS gives every developer a full copy of the history; Git is the standard one.
- A commit is a snapshot; commits point to their parents and form the history.
- Git is the tool; GitHub and GitLab are services that host repositories.
- The key places are the working tree, the staging area, the repository and the remote.

## Sources and further reading

- [Pro Git — 1.1 About Version Control](https://git-scm.com/book/en/v2/Getting-Started-About-Version-Control) — the official free book, the same story in more detail.
- [Pro Git — 1.2 A Short History of Git](https://git-scm.com/book/en/v2/Getting-Started-A-Short-History-of-Git) — why and how Git was created.
- [Pro Git — 1.3 What is Git?](https://git-scm.com/book/en/v2/Getting-Started-What-is-Git%3F) — snapshots, integrity and the three states of a file.
- [About Git (git-scm.com)](https://git-scm.com/about) — the main ideas on the official site.
- [GitHub Docs — About Git](https://docs.github.com/en/get-started/using-git/about-git) — the difference between Git and GitHub.

---

[🏠 Start](../START-HERE.md) · [← Phase 01 — Git and version control](README.md) · [📋 Phase 01](README.md) · [01.2 Installing and configuring Git →](02-install-and-configure-git.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
