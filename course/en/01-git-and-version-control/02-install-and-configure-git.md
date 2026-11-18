<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 01.1 What version control is](01-what-is-version-control.md) · [📋 Phase 01](README.md) · [01.3 Your first repository and commits →](03-your-first-repository-and-commits.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/01-git-and-version-control/02-install-and-configure-git.md) <!-- nav -->

# Lesson 01.2 — Installing and configuring Git

> **Phase 01 — Git and version control** · Lesson 2 of 10 · ⏱ about 120 minutes

## What you will learn

- How to check whether Git is installed on your Mac, and how to install or update it.
- Where Git keeps its settings, and the three levels of configuration: system, global and local.
- How to tell Git who you are, and how to set a small group of safe defaults that every beginner should have.
- How to ignore files such as `.DS_Store` in every repository with one global ignore file.
- How to find help when you forget a command.

## Before you start

- You finished [Lesson 01.1 — What version control is](01-what-is-version-control.md).
- You know how to open a terminal and run commands ([Lesson 00.4](../00-getting-started/04-terminal-basics.md)).
- You installed Rust and RustRover ([Lesson 00.5](../00-getting-started/05-install-rust-and-rustrover.md)).

## New words

| Word | Meaning |
|---|---|
| **configuration** (config) | A set of settings that change how a program behaves. |
| **config level** | Where a setting is stored and how widely it applies: *system* (all users), *global* (you), *local* (one repository). |
| **`~`** (tilde) | A short name for your home folder, for example `/Users/ada`. |
| **dotfile** | A file or folder whose name starts with a dot (`.gitconfig`). Hidden by default in Finder and `ls`. |
| **Xcode Command Line Tools** | A free package from Apple with developer programs (compiler tools, `git`, `make`). Rust on macOS needs it too. |
| **Homebrew** | A popular *package manager* for macOS: a program that installs other programs with one command, `brew install …`. |
| **package manager** | A program that downloads, installs and updates other programs. |
| **PATH** | A list of folders where the terminal looks for programs when you type a command name. |
| **editor** (for Git) | The text editor Git opens when it needs you to type a message. |
| **alias** | A short custom name for a longer command. |
| **pager** | A program that shows long output one screen at a time (Git uses `less`). Press `q` to leave it. |
| **global ignore file** | A file with patterns of files that Git must ignore in *every* repository on your computer. |
| **GUI** (Graphical User Interface) | A way to use a program with windows and the mouse. |

## 1. Is Git already installed?

On macOS, Git often comes with the **Xcode Command Line Tools**, which Rust also needs. Check:

```bash
git --version
which git
```

```text
git version 2.54.0 (Apple Git-157)
/usr/bin/git
```

**What happens:**

1. `git --version` asks Git to print its version. Your numbers can be different. Any version **2.40 or newer** is good for this course. In this lesson we use settings that need at least 2.37.
2. `which git` prints the full path of the program that runs when you type `git`. `/usr/bin/git` is a small launcher from Apple that starts the Git inside the Xcode tools.

If the terminal says `command not found` (or macOS opens a window that offers to install developer tools), install it in one of two ways.

**Way A — Xcode Command Line Tools (simple, from Apple):**

```bash
xcode-select --install
```

A window opens. Click *Install* and wait. This also installs the compiler tools that Rust needs to link programs. Apple's Git is updated together with macOS updates, so it can be a few versions behind the newest.

**Way B — Homebrew (newer Git, easy to update):**

```bash
brew install git
```

If Homebrew is not installed, do not install it now. Way A is enough. If you already use Homebrew, this gives you the newest Git, and `brew upgrade git` updates it later.

After Way B, there can be two Gits on your Mac. The one that runs is the first in your `PATH`:

```bash
which -a git
```

```text
/usr/bin/git
```

(With Homebrew you would see two lines, for example `/opt/homebrew/bin/git` first, and `/usr/bin/git` second. The first line is the one that runs.)

> 💡 **Tip:** RustRover finds Git by itself. You can check it in *Settings → Version Control → Git*, where the field "Path to Git executable" shows which Git it uses. It uses the same configuration files as the terminal, so everything you set in this lesson works in the IDE (Integrated Development Environment, a program for writing code) too.

## 2. Where Git keeps its settings

Git reads settings from three places. A more specific place wins over a more general one.

```text
 system   (all users of the computer)   e.g. /etc/gitconfig        weakest
    ↓ overridden by
 global   (you, in every repo)          ~/.gitconfig               ← most of your settings
    ↓ overridden by
 local    (one repository only)         <repo>/.git/config         strongest
```

You change settings with `git config`. The flag tells Git which level to use:

| Command | Level |
|---|---|
| `git config --system …` | system (rarely used; needs administrator rights) |
| `git config --global …` | global: the file `~/.gitconfig` |
| `git config …` (no flag, inside a repository) | local: the file `.git/config` of that repository |

The names of settings have two parts: `section.key`, for example `user.name` (section `user`, key `name`).

## 3. Tell Git who you are

Every commit stores an author: a name and an email. Git does not guess it reliably, so you set it once:

```bash
git config --global user.name "Ada Learner"
git config --global user.email "ada@example.com"
```

Use **your own** name and email instead of the example. Keep the quotes around the name: it has a space in it.

**What happens:**

1. `git config` changes a setting. `--global` says "for me, in every repository".
2. `user.name` and `user.email` are the keys. The last word is the value.
3. Git writes the values into the file `~/.gitconfig`. Nothing is printed when it works.

Read the values back:

```bash
git config user.name
git config user.email
```

```text
Ada Learner
ada@example.com
```

> 🔒 **Security:** the email in a commit becomes **public** when you push the commit to GitHub or GitLab. If you do not want your real address in public history, use the private "noreply" address that GitHub gives you (it looks like `12345678+yourname@users.noreply.github.com`; you find it in your GitHub account, *Settings → Emails*, after you turn on "Keep my email addresses private"). You will create the account in [Lesson 01.8](08-remotes-github-and-gitlab.md). You can change the email later with the same command.

## 4. A small set of safe defaults

Run these commands once. Each one is explained below.

```bash
git config --global init.defaultBranch main
git config --global core.editor nano
git config --global pull.ff only
git config --global push.autoSetupRemote true
git config --global merge.conflictStyle zdiff3
git config --global fetch.prune true
```

| Setting | What it does | Why you want it |
|---|---|---|
| `init.defaultBranch main` | The first branch of every new repository is called `main`. | Git's old default was `master`. Git 3.0 will change the default to `main`; GitHub and GitLab already use `main`. Without this setting, Git prints a long hint each time you create a repository. |
| `core.editor nano` | The text editor that Git opens for commit messages. | Without it, Git may open `vim`, which beginners find very confusing (see the mistake below). `nano` shows its keys at the bottom of the screen. |
| `pull.ff only` | `git pull` may only "fast-forward" (simply add the new commits of others). If your work and theirs have both changed, it stops. | It never creates a surprise merge. You will learn what "fast-forward" means in [Lesson 01.6](06-branches-and-merging.md) and see this setting in action in [Lesson 01.8](08-remotes-github-and-gitlab.md). |
| `push.autoSetupRemote true` | The first `git push` of a new branch connects it to the same-named branch on the remote. | You do not have to type `--set-upstream` each time. |
| `merge.conflictStyle zdiff3` | In a merge conflict, Git also shows the *original* text between the two conflicting versions. | Resolving conflicts is much easier with the third piece of information. See [Lesson 01.7](07-merge-conflicts.md). Needs Git 2.35 or newer. |
| `fetch.prune true` | When you download updates, Git removes your local note of remote branches that were deleted. | Keeps your branch list clean. |

> 📏 **Convention:** these are *safe* defaults: they do not change your files, and they only make Git's behaviour more predictable. Do not copy long "ultimate .gitconfig" files from the internet before you understand each line.

### If you prefer another editor

`core.editor` is any command that opens a file and waits until you close it. For example, with Visual Studio Code installed: `git config --global core.editor "code --wait"`. The `--wait` part is important: it makes Git wait until you close the file tab. For now `nano` is enough.

## 5. Look at the result

```bash
git config --global --list
```

```text
user.name=Ada Learner
user.email=ada@example.com
init.defaultbranch=main
core.editor=nano
pull.ff=only
push.autosetupremote=true
merge.conflictstyle=zdiff3
fetch.prune=true
```

(Key names are shown in lowercase. That is normal; Git ignores letter case in the section and key names.)

To see **which file** defines each setting, add `--show-origin`:

```bash
git config --global --list --show-origin
```

```text
file:/Users/ada/.gitconfig	user.name=Ada Learner
file:/Users/ada/.gitconfig	user.email=ada@example.com
file:/Users/ada/.gitconfig	init.defaultbranch=main
…
```

The file is plain text. Open it with `cat ~/.gitconfig` (or edit it with `git config --global --edit`):

```ini
[user]
	name = Ada Learner
	email = ada@example.com
[init]
	defaultBranch = main
[core]
	editor = nano
[pull]
	ff = only
[push]
	autoSetupRemote = true
[merge]
	conflictStyle = zdiff3
[fetch]
	prune = true
```

**What happens:** the file has *sections* in square brackets and `key = value` lines. This is the format called INI (from "initialization", a very old, simple settings-file format). `git config` just edits this file for you. On a Mac that has Apple's Git, a *system* file may also exist (Apple puts a small setting there for the macOS keychain). You can ignore it.

## 6. Local settings override global ones

Sometimes one project needs a different identity, for example a work repository with a work email. Inside that repository, run the same command **without** `--global`:

```bash
git init -b main demo-local
cd demo-local
git config user.email "work@example.org"
git config --show-scope --get-all user.email
```

```text
global	ada@example.com
local	work@example.org
```

**What happens:**

1. `git init -b main demo-local` creates a new empty repository (you will study `git init` in the next lesson).
2. Without `--global`, `git config` writes into `demo-local/.git/config`, so the setting applies only here.
3. `--show-scope` prints where each value comes from. There are two values for `user.email`. Git uses the **last and most specific** one: the local one.

Check which one is active:

```bash
git config user.email
```

```text
work@example.org
```

## 7. A global ignore file

Some files appear in every project and should never be committed: the macOS Finder file `.DS_Store`, the RustRover folder `.idea/`, editor swap files. You can tell Git once, for all repositories on your computer:

```bash
printf '.DS_Store\n.idea/\n*.swp\n' > ~/.gitignore_global
git config --global core.excludesFile '~/.gitignore_global'
cat ~/.gitignore_global
```

```text
.DS_Store
.idea/
*.swp
```

**What happens:**

1. `printf '...\n' > file` writes three lines into the file `~/.gitignore_global`. (`\n` means "new line".)
2. `core.excludesFile` tells Git: "also read ignore patterns from this file in every repository".
3. `cat` shows the file.

Each line is a **pattern**: `.DS_Store` matches a file with that name, `.idea/` matches a folder, and `*.swp` matches any file that ends with `.swp` (`*` stands for "anything"). You will use ignore files in a project in [Lesson 01.3](03-your-first-repository-and-commits.md).

> 📏 **Convention:** put things that belong to **your** computer and tools (macOS, your editor) in the *global* ignore file. Put things that belong to the **project** (like Rust's `target/` folder) in the project's own `.gitignore`, so every teammate gets them.

## 8. Aliases and help

An **alias** is a shortcut. For example:

```bash
git config --global alias.st "status --short --branch"
git st
```

It is convenient, but wait with aliases until you know the full commands; otherwise you will be lost on any other computer. Remove an alias with `git config --global --unset alias.st`.

When you forget something, Git has built-in help:

| Command | What you get |
|---|---|
| `git help commit` | The full manual page of `commit`. Move with the arrows or space, leave with `q`. |
| `git commit -h` | A short list of options only. Handy and fast. |
| `git help -a` | All commands. |
| `git help -g` | The list of concept guides, such as `tutorial`, `everyday`, `glossary`. Try `git help everyday`. |
| `git help config` | All settings (very long). |

## Common mistakes

> ⚠️ **Common mistake:** forgetting the quotes around a name that has a space.

```bash
git config --global user.name Ada Learner
git config --global user.name
```

```text
Ada
```

Git did not complain, but it saved only `Ada`. The shell split the text into two words, and Git used the first. **Fix:** put quotes around the value: `git config --global user.name "Ada Learner"`. Always read the setting back to be sure.

> ⚠️ **Common mistake:** a typo in a key name is **not** an error.

```bash
git config --global user.mial "oops@example.com"
git config --global --get user.mial
```

```text
oops@example.com
```

Git happily stores any key. The real `user.email` is unchanged, so your commits would use the old (or a wrong) address. **Fix:** after setting something, check with `git config --global --list`, and remove the wrong key: `git config --global --unset user.mial`.

> ⚠️ **Common mistake:** the editor is stuck. If Git opens `vim` and nothing you type seems to work, do this: press `Esc`, type `:q!` and press `Enter`. That quits without saving. (To save and quit in `vim`: press `Esc`, type `:wq`, press `Enter`.) Setting `core.editor` to `nano` avoids this problem.

> ⚠️ **Common mistake:** setting your identity only after the first commits, or using `--global` when you want only one repository. Old commits keep the old author. Check your settings **before** the first commit.

> 🔒 **Security:** never put passwords, tokens or keys into `~/.gitconfig`. Do not paste someone else's configuration without reading it.

## Try it yourself

1. Check your Git: run `git --version` and `which git`. Write the version down in your notes.
2. Set your name and email (section 3) with **your own** values. Read them back.
3. Run the six default settings from section 4.
4. Run `git config --global --list` and compare with the list in section 5.
5. Create the global ignore file (section 7).
6. Make a test folder and a repository and check that the global ignore works:

```bash
mkdir -p ~/projects/git-playground/ignore-test
cd ~/projects/git-playground/ignore-test
git init
touch .DS_Store notes.txt
git status --short
git check-ignore -v .DS_Store
```

```text
?? notes.txt
/Users/ada/.gitignore_global:1:.DS_Store	.DS_Store
```

**What happens:**

1. `mkdir -p …` creates the practice folder `~/projects/git-playground` (outside the course folder, on purpose) and a subfolder.
2. `git init` turns the folder into a repository.
3. `touch` creates two empty files.
4. `git status --short` lists files that Git sees. Only `notes.txt` appears (`??` means "new, not tracked yet"). `.DS_Store` is hidden from the list because it is ignored.
5. `git check-ignore -v` explains *why* a file is ignored: the file, the line number and the pattern that matched.

Keep the folder `~/projects/git-playground`: later lessons use it.

## Exercises

### Exercise 1 — Where is Git, and which version? (easy)

Write one command that shows the version of Git and one command that shows where the Git program lives. Then find out if the version has the setting `merge.conflictStyle zdiff3` (it needs 2.35 or newer).

<details>
<summary>Hint</summary>

The first is `git --version`. The second is a command from section 1 that "finds" programs.

</details>

<details>
<summary>Answer</summary>

```bash
git --version
which git
```

If the version number is 2.35 or higher (for example 2.54), `zdiff3` is supported.

Why it works: `git --version` prints the version, and `which` shows the first program with that name found in your `PATH`.

</details>

### Exercise 2 — Identity check (easy)

Set your name and email, then write one command that prints **both** values together with the file they come from.

<details>
<summary>Hint</summary>

`git config --global --list --show-origin` prints everything. You can filter it with `--get-regexp '^user\.'`.

</details>

<details>
<summary>Answer</summary>

```bash
git config --global user.name "Ada Learner"
git config --global user.email "ada@example.com"
git config --global --show-origin --get-regexp '^user\.'
```

```text
file:/Users/ada/.gitconfig	user.name Ada Learner
file:/Users/ada/.gitconfig	user.email ada@example.com
```

Why it works: `--get-regexp` prints all keys that match a pattern (here: keys that start with `user.`), and `--show-origin` adds the file name.

</details>

### Exercise 3 — A different email for one repository (medium)

Create a repository `~/projects/git-playground/work-demo`. Set the email `work@example.org` only for this repository. Prove that the global email did **not** change.

<details>
<summary>Hint</summary>

Run `git config user.email …` without `--global`, inside the repository. Compare `git config --local user.email` and `git config --global user.email`.

</details>

<details>
<summary>Answer</summary>

```bash
mkdir -p ~/projects/git-playground/work-demo
cd ~/projects/git-playground/work-demo
git init
git config user.email "work@example.org"
git config --local user.email
git config --global user.email
```

```text
work@example.org
ada@example.com
```

Why it works: without `--global`, the value goes to `.git/config` of this repository. The global file `~/.gitconfig` is untouched, and the local value wins only inside this repository.

</details>

### Exercise 4 — Ignore your editor files everywhere (medium)

Add the pattern `*.log` to your global ignore file. Then prove that a file `debug.log` in a new repository is ignored.

<details>
<summary>Hint</summary>

Append a line with `>>`. Then create a repository and use `git check-ignore -v`.

</details>

<details>
<summary>Answer</summary>

```bash
echo '*.log' >> ~/.gitignore_global
mkdir -p ~/projects/git-playground/ignore-test2
cd ~/projects/git-playground/ignore-test2
git init
touch debug.log
git check-ignore -v debug.log
```

```text
/Users/ada/.gitignore_global:4:*.log	debug.log
```

Why it works: `>>` adds a line to the end of the file (here, line 4). Git reads the file in every repository and `check-ignore -v` shows the line that matched.

</details>

### Exercise 5 — Read the manual (hard)

Using only Git's built-in help, find out what the option `--amend` of `git commit` does (you will use it in [Lesson 01.5](05-undoing-changes-safely.md)), and what `git config --unset` does. Do not use the internet.

<details>
<summary>Hint</summary>

`git commit -h` shows only a short list. `git help commit` opens the manual: press `/` to search inside it, type `amend`, press `Enter`, and `q` to leave.

</details>

<details>
<summary>Answer</summary>

```bash
git help commit      # search for "--amend" with /--amend
git help config      # search for "--unset"
```

`--amend` replaces the tip of the current branch with a new commit (it lets you fix the last commit's message or content). `--unset` removes a key from the configuration file.

Why it works: the manual pages are installed together with Git, so they work offline. `less` (the pager) lets you search with `/`.

</details>

## Quiz

1. Which file stores your global Git settings?

<details>
<summary>Answer</summary>

`~/.gitconfig` (the file `.gitconfig` in your home folder).

</details>

2. You set `user.email` globally and also locally in one repository. Which value does Git use in that repository?

<details>
<summary>Answer</summary>

The local one. The more specific level wins.

</details>

3. You typed `git config --global user.name Ada Learner` and Git printed nothing. What was saved?

<details>
<summary>Answer</summary>

Only `Ada`. The name needs quotes because it contains a space.

</details>

4. Why do we set `init.defaultBranch main`?

<details>
<summary>Answer</summary>

So that every new repository starts with a branch called `main` (the name used by GitHub and GitLab and by future Git versions) and Git does not print the long hint about `master`.

</details>

5. What is the difference between `~/.gitignore_global` and a project's `.gitignore`?

<details>
<summary>Answer</summary>

The global file is only on your computer and applies to all repositories (for your own tools, like `.DS_Store`). A project `.gitignore` is committed to the repository and applies to every person who works on it.

</details>

6. How do you leave `vim` without saving?

<details>
<summary>Answer</summary>

Press `Esc`, type `:q!` and press `Enter`.

</details>

## Summary

- Check Git with `git --version`; install it with `xcode-select --install` or `brew install git`.
- Settings live on three levels: system, global (`~/.gitconfig`) and local (`.git/config`). The most specific level wins.
- Set `user.name` and `user.email` once, with quotes, and read them back.
- A few safe defaults: `init.defaultBranch main`, an easy editor, `pull.ff only`, `push.autoSetupRemote true`, `merge.conflictStyle zdiff3`, `fetch.prune true`.
- A global ignore file keeps `.DS_Store` and `.idea/` out of every repository.
- `git config --list --show-origin` shows every setting and its file; `git help <command>` and `git <command> -h` are your offline documentation.
- Typos in setting names are not errors. Always read the result back.

## Sources and further reading

- [Pro Git — 1.6 First-Time Git Setup](https://git-scm.com/book/en/v2/Getting-Started-First-Time-Git-Setup) — the official first configuration.
- [Pro Git — 1.5 Installing Git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git) — install options for each system.
- [git-config manual](https://git-scm.com/docs/git-config) — every setting, with its exact meaning.
- [gitignore manual](https://git-scm.com/docs/gitignore) — the exact rules for ignore patterns.
- [GitHub Docs — Setting your commit email address](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-email-preferences/setting-your-commit-email-address) — how to use the private "noreply" address.

---

[🏠 Start](../START-HERE.md) · [← 01.1 What version control is](01-what-is-version-control.md) · [📋 Phase 01](README.md) · [01.3 Your first repository and commits →](03-your-first-repository-and-commits.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
