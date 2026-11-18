<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 01.7 Merge conflicts](07-merge-conflicts.md) · [📋 Phase 01](README.md) · [01.9 Teamwork: pull requests, code review and conventions →](09-teamwork-pull-requests-and-conventions.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/01-git-and-version-control/08-remotes-github-and-gitlab.md) <!-- nav -->

# Lesson 01.8 — Remote repositories: GitHub and GitLab

> **Phase 01 — Git and version control** · Lesson 8 of 10 · ⏱ about 2.5 hours (2 sittings)

## What you will learn

- What a remote repository is, and how `clone`, `push`, `fetch` and `pull` move commits between copies.
- How to practise all of it **without internet**, with a local "server" folder.
- What GitHub and GitLab add to Git, and how to create an account and a repository.
- How to log in safely with an SSH key (and why not a password).
- How to read and fix the most common errors: rejected pushes and "permission denied".
- Basic safety rules for secrets and keys.

## Before you start

- You finished [Lesson 01.7 — Merge conflicts](07-merge-conflicts.md).
- You know how to merge and how to read `git log --graph` ([Lesson 01.4](04-viewing-history-and-changes.md), [Lesson 01.6](06-branches-and-merging.md)).
- For the second half you need a free GitHub account (and, optionally, a GitLab account). If you do not want to create an account yet, do the first half; the account part is not needed for the next lesson's exercises, but it is needed for the project in [Lesson 01.10](10-project-your-learning-repo-and-checkpoint.md).

## New words

| Word | Meaning |
|---|---|
| **remote** | Another copy of your repository, usually on a server. |
| **`origin`** | The default name of the remote that you cloned from or first added. |
| **clone** | To make a full copy of a remote repository on your computer, including its whole history. |
| **push** | To send your new commits to a remote. |
| **fetch** | To download new commits from a remote **without** changing your branches or files. |
| **pull** | `fetch` followed by a merge into your current branch. |
| **remote-tracking branch** | Your local note of where a branch was on the remote at the last contact, for example `origin/main`. You cannot commit on it. |
| **upstream** (tracking) | The remote branch that a local branch is linked to. `git push` and `git pull` use it. |
| **bare repository** | A repository without working files, only the history. Servers use bare repositories. |
| **non-fast-forward** | A situation where the remote has commits that you do not have, so your push is rejected. |
| **URL** (Uniform Resource Locator) | An address, such as `https://github.com/user/repo.git`. |
| **HTTPS** (HyperText Transfer Protocol Secure) | The protocol of web pages; Git can use it with a token for login. |
| **SSH** (Secure Shell) | A protocol for secure connections to another computer. Git can use it with a key for login. |
| **key pair** | Two linked files: a **private key** (secret, stays on your computer) and a **public key** (can be shared). |
| **PAT** (Personal Access Token) | A long random string that works like a password for one purpose, created in your GitHub/GitLab settings. |
| **2FA** (Two-Factor Authentication) | A second proof of identity (for example a code from an app) in addition to the password. |
| **`gh`** | The official GitHub command-line tool (GitHub CLI). |
| **CLI** (Command-Line Interface) | A way to use a program by typing commands. |
| **README** | A text file (usually `README.md`) at the top of a repository that explains what the project is. |
| **licence** | A text file that says what other people may do with your code. |

## 1. What is a remote?

So far you worked in one copy of the repository. A **remote** is another copy, on another computer. With a remote you can:

- keep a **backup** of your work outside your laptop;
- **share** code with a team, which pushes and pulls to the same remote;
- run **automatic checks** on every push (next lesson).

```text
   your computer                         server (GitHub, GitLab, a folder…)
 ┌────────────────┐    git push  ───▶   ┌─────────────────────┐
 │  your repo     │                      │  remote repository  │
 │  main          │    git fetch  ◀───   │  main               │
 │  origin/main   │    git pull   ◀───   │                     │
 └────────────────┘                      └─────────────────────┘
```

Because Git is distributed, the remote is not special. It is just another repository that everybody agrees to use. Everything we do below works with any remote.

## 2. A local laboratory: a "server" in a folder

Before you touch GitHub, practise with a remote that lives in a folder on your computer. It behaves like a real server, and you cannot break anything.

First create the "server". A server keeps only the history, so it is a **bare** repository. By convention its folder name ends in `.git`.

```bash
cd ~/projects/git-playground
mkdir remote-lab
cd remote-lab
git init --bare shared.git
```

```text
Initialized empty Git repository in /Users/ada/projects/git-playground/remote-lab/shared.git/
```

Next create Ada's repository and connect it to the "server":

```bash
git init ada
cd ada
echo "# Team notes" > README.md
git add README.md
git commit -m "Start team notes"
git remote add origin ../shared.git
git remote -v
git push -u origin main
```

```text
origin	../shared.git (fetch)
origin	../shared.git (push)
To ../shared.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.
```

**What happens:**

1. `git remote add origin ../shared.git` gives the remote the short name `origin` and an address (here, a relative path; later it will be a URL).
2. `git remote -v` lists remotes. Each has a fetch address and a push address.
3. `git push -u origin main` sends the branch `main` to `origin`. `-u` (`--set-upstream`) also links your `main` to `origin/main`, so later a plain `git push` or `git pull` knows where to go. (Your setting `push.autoSetupRemote` from [Lesson 01.2](02-install-and-configure-git.md) does this link automatically on the first push of a *new* branch.)
4. `* [new branch] main -> main` says: the remote did not have `main`; it has now.

Check the state:

```bash
git branch -vv
git status
git log --oneline --decorate
```

```text
* main ebcd10d [origin/main] Start team notes
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
ebcd10d (HEAD -> main, origin/main) Start team notes
```

**What happens:** `[origin/main]` in `git branch -vv` shows the upstream of `main`. `origin/main` in the log is the **remote-tracking branch**: your note that "`main` on `origin` is at this commit". `up to date` means your `main` and your note are the same commit.

## 3. A second person: `git clone`

Now Bob joins. He makes a full copy of the "server" with `clone`:

```bash
cd ..
git clone shared.git bob
cd bob
git config user.name "Bob Builder"
git config user.email "bob@example.com"
git remote -v
git branch -a
```

```text
Cloning into 'bob'...
done.
origin	/Users/ada/projects/git-playground/remote-lab/shared.git (fetch)
origin	/Users/ada/projects/git-playground/remote-lab/shared.git (push)
* main
  remotes/origin/HEAD -> origin/main
  remotes/origin/main
```

**What happens:**

1. `git clone <address> <folder>` creates a folder, copies the **whole history**, adds the remote `origin` automatically, and checks out the main branch.
2. `git config user.name …` without `--global` gives Bob his own identity inside this repository only (it is a pretend second person on your computer).
3. `git branch -a` lists local branches and remote-tracking branches (`remotes/origin/…`).

Bob adds a commit and pushes:

```bash
echo "Bob was here" >> README.md
git commit -am "Bob adds a line"
git push
```

```text
[main 3e99a64] Bob adds a line
 1 file changed, 1 insertion(+)
To /Users/ada/projects/git-playground/remote-lab/shared.git
   ebcd10d..3e99a64  main -> main
```

`ebcd10d..3e99a64` means: the remote `main` moved from the first hash to the second.

## 4. The remote has news: `fetch`, `pull` and a rejected push

Ada, meanwhile, did not know about Bob's work. She makes a commit of her own:

```bash
cd ../ada
echo "Ada notes" > ada.txt
git add ada.txt
git commit -m "Ada adds a file"
git push
```

```text
To ../shared.git
 ! [rejected]        main -> main (fetch first)
error: failed to push some refs to '../shared.git'
hint: Updates were rejected because the remote contains work that you do not
hint: have locally. This is usually caused by another repository pushing to
hint: the same ref. If you want to integrate the remote changes, use
hint: 'git pull' before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```

**What happens:** this is the most common Git error in teams. The remote `main` has a commit (Bob's) that Ada's `main` does not have. If Git accepted her push, Bob's commit would be lost. So the push is **rejected**. Nothing is damaged. The rule: **you must first bring the remote's news to your side.** Do it in two steps so you can look before you decide:

```bash
git fetch
git status
git log --oneline --graph --all --decorate
```

```text
From ../shared
   ebcd10d..3e99a64  main       -> origin/main
On branch main
Your branch and 'origin/main' have diverged,
and have 1 and 1 different commits each, respectively.
  (use "git pull" if you want to integrate the remote branch with yours)

nothing to commit, working tree clean
* 604011a (HEAD -> main) Ada adds a file
| * 3e99a64 (origin/main, origin/HEAD) Bob adds a line
|/
* ebcd10d Start team notes
```

**What happens:**

1. `git fetch` downloaded Bob's commit and moved `origin/main`. Your own `main` and your files are **not touched**. That is why `fetch` is always safe.
2. `git status` says the branches have **diverged**: one commit on each side. The graph shows the fork.

Now integrate. `git pull` is a shortcut for `git fetch` + `git merge`. With the safe setting `pull.ff only` from Lesson 01.2, a pull that would need a merge **stops**:

```bash
git pull
```

```text
hint: Diverging branches can't be fast-forwarded, you need to either:
hint:
hint: 	git merge --no-ff
hint:
hint: or:
hint:
hint: 	git rebase
hint:
hint: Disable this message with "git config set advice.diverging false"
fatal: Not possible to fast-forward, aborting.
```

**What happens:** Git refuses to create a merge commit behind your back, and tells you the two ways: merge or rebase. For now, choose merge, explicitly:

```bash
git merge origin/main --no-edit
git log --oneline --graph --all --decorate
git push
git status
```

```text
Merge made by the 'ort' strategy.
 README.md | 1 +
 1 file changed, 1 insertion(+)
*   d87e070 (HEAD -> main) Merge remote-tracking branch 'origin/main'
|\
| * 3e99a64 (origin/main, origin/HEAD) Bob adds a line
* | 604011a Ada adds a file
|/
* ebcd10d Start team notes
To ../shared.git
   3e99a64..d87e070  main -> main
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

**What happens:** `git merge origin/main` merged the remote-tracking branch into `main` (a normal merge from [Lesson 01.6](06-branches-and-merging.md); if the same lines were changed, you would resolve a conflict as in [Lesson 01.7](07-merge-conflicts.md)). Now Ada's `main` contains both commits, so `git push` is a fast-forward for the server and is accepted.

Bob picks up the result:

```bash
cd ../bob
git pull
git log --oneline
```

```text
From /Users/ada/projects/git-playground/remote-lab/shared
   3e99a64..d87e070  main       -> origin/main
Updating 3e99a64..d87e070
Fast-forward
 ada.txt | 1 +
 1 file changed, 1 insertion(+)
 create mode 100644 ada.txt
d87e070 Merge remote-tracking branch 'origin/main'
604011a Ada adds a file
3e99a64 Bob adds a line
ebcd10d Start team notes
```

For Bob it is a plain fast-forward: he had nothing new, so Git only moved his `main` forward.

### The daily routine

| Command | What it does | Changes my branch? |
|---|---|---|
| `git clone <url>` | copies a whole repository | creates it |
| `git remote -v` | shows remotes and their addresses | no |
| `git fetch` | downloads news, moves `origin/*` | **no** |
| `git pull` | `fetch` + merge (with our setting: fast-forward only) | yes |
| `git push` | sends my commits to the upstream | no (changes the remote) |
| `git push -u origin <branch>` | the first push of a branch, sets upstream | no |

> 💡 **Tip:** the habit of a good team member: **`git pull` before you start work, `git pull` before you push.** Small and frequent integration means small conflicts.

> ⚠️ **Common mistake:** "fixing" a rejected push with `git push --force`. This overwrites the remote history and destroys other people's commits. Never force-push to a shared branch like `main`. (You will see the careful variant, `--force-with-lease`, for **your own** branches in [Lesson 01.9](09-teamwork-pull-requests-and-conventions.md).)

## 5. GitHub and GitLab

A hosting service gives you the "server" so you do not run it yourself, and adds:

- a **web page** for your repository (browse files, history, diffs);
- **access control** (private repositories, teams);
- **pull requests** (GitHub) / **merge requests** (GitLab) for review ([Lesson 01.9](09-teamwork-pull-requests-and-conventions.md));
- **issues** (task tracking);
- **CI** (Continuous Integration): automatic builds and tests on every push.

| | GitHub | GitLab |
|---|---|---|
| Address | `github.com` | `gitlab.com` (or your company's own server) |
| Review feature | pull request (PR) | merge request (MR) |
| Automatic checks | GitHub Actions (`.github/workflows/*.yml`) | GitLab CI (`.gitlab-ci.yml`) |
| Command-line tool | `gh` | `glab` |

They are very similar. Learn one well, and the other takes an hour. The Rust project itself and most crates live on GitHub; many companies use GitLab, often on their own servers.

### Create an account

1. Go to github.com and choose *Sign up*. Use a strong, unique password in a password manager.
2. Turn on **2FA** (*Settings → Password and authentication*). Accounts without it are the easiest to steal. Keep the recovery codes safe.
3. In *Settings → Emails* choose "Keep my email addresses private", and use the `…@users.noreply.github.com` address as your Git `user.email` ([Lesson 01.2](02-install-and-configure-git.md)) if you want to keep your address out of public commits.
4. Optional: do the same on gitlab.com.

Create the account yourself, and never share your password or tokens with anyone, including an AI (artificial intelligence) assistant.

## 6. Logging in: SSH keys

To push to a private repository, the server must know who you are. Today GitHub does **not** accept your account password for Git over HTTPS. There are two good methods:

| Method | How it works | Good for |
|---|---|---|
| **SSH key** | You hold a private key; the server knows your public key. | a laptop you use every day (recommended here) |
| **HTTPS + token (PAT)** | You paste a token instead of a password; a helper stores it. | places where SSH is blocked; scripts and CI |

### How a key pair works

You create **two** files. The **private key** proves you are you. It never leaves your computer and nobody else may see it. The **public key** is a matching lock; you give it to GitHub. When you connect, the server uses the public key to check that you hold the matching private key, without the private key ever being sent.

### Create a key

```bash
ssh-keygen -t ed25519 -C "ada@example.com"
```

`ssh-keygen` asks three questions. Press `Enter` to accept the default file `~/.ssh/id_ed25519`, then type a **passphrase** (a password that protects the key file) twice. Use one: if somebody copies your laptop disk, the passphrase still protects the key.

```text
Generating public/private ed25519 key pair.
Your identification has been saved in /Users/ada/.ssh/id_ed25519
Your public key has been saved in /Users/ada/.ssh/id_ed25519.pub
The key fingerprint is:
SHA256:g15nmXfDJKWmpV/+1ncMgfDW00MB/CZXbSSrnCRvFKo ada@example.com
```

**What happens:**

1. `-t ed25519` chooses a modern, short and strong key type (Ed25519 is the name of the algorithm; the older popular type is RSA, named after the initials of its inventors Rivest, Shamir and Adleman).
2. `-C "…"` adds a comment, a label so you remember what the key is for.
3. Two files appear in `~/.ssh/`: `id_ed25519` (**private**, never share) and `id_ed25519.pub` (**public**, safe to share).
4. The fingerprint is a short hash of the public key; you can use it to identify the key.

> 🔒 **Security:** the private key file is `id_ed25519` (no `.pub`). Never email it, never paste it into a website or a chat, never commit it to Git. Only ever paste the `.pub` file.

If you already have a key in `~/.ssh/`, do not overwrite it: `ssh-keygen` will ask before overwriting. Answer `n` and choose a different file name.

### Let the Mac remember the passphrase

```bash
ssh-add --apple-use-keychain ~/.ssh/id_ed25519
```

```text
Identity added: /Users/ada/.ssh/id_ed25519 (ada@example.com)
```

This stores the key in `ssh-agent` (a helper program that holds unlocked keys) and the passphrase in the macOS Keychain, so you type the passphrase only once. To make this automatic after every restart, create or edit the file `~/.ssh/config` and add:

```text
Host github.com gitlab.com
  AddKeysToAgent yes
  UseKeychain yes
  IdentityFile ~/.ssh/id_ed25519
```

**What happens:** for connections to those two hosts, SSH adds the key to the agent, uses the Keychain and uses this key file.

### Give GitHub the public key

1. Copy the **public** key to the clipboard: `pbcopy < ~/.ssh/id_ed25519.pub` (`pbcopy` is a macOS program that copies its input to the clipboard).
2. On GitHub open *Settings → SSH and GPG keys → New SSH key* (GPG, GNU Privacy Guard, is another kind of key that you do not need now), give it a title (for example "MacBook"), choose type "Authentication key", and paste.
3. Do the same on GitLab (*Preferences → SSH Keys*) if you use it.

A public key is one line:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBVfFM8Sx0U7SRhFYumqY73KRYDx0ihD26lO5+PHowNU ada@example.com
```

### Test the connection

```bash
ssh -T git@github.com
```

The first time, SSH asks `Are you sure you want to continue connecting (yes/no/[fingerprint])?` Compare the fingerprint with the one published in GitHub's documentation, then type `yes`. After that, the answer must look like this:

```text
Hi your-username! You've successfully authenticated, but GitHub does not provide shell access.
```

(GitLab says `Welcome to GitLab, @your-username!`.) If you did not add the key (or the key is not offered), you get the most common SSH error:

```text
git@github.com: Permission denied (publickey).
```

This means: the server did not recognise any of your keys. Check that you pasted the **public** key to **your** account, that the key is loaded (`ssh-add -l`), and that the remote address starts with `git@github.com:` and not `https://`.

## 7. Connect your repository to GitHub

1. On GitHub choose *+ → New repository*. Give it a name (for example `git-practice`). Choose **Private**. **Do not** tick "Add a README", ".gitignore" or "license" if you will push an existing local repository, because the new repository must be empty.
2. GitHub shows the addresses. Copy the **SSH** one. It looks like `git@github.com:your-username/git-practice.git`.
3. In your local repository run:

```bash
git remote add origin git@github.com:your-username/git-practice.git
git push -u origin main
```

(Replace `your-username` and the repository name with your own.) Refresh the web page: your files and history are online. To get a repository from GitHub onto a new computer:

```bash
git clone git@github.com:your-username/git-practice.git
```

The two address styles:

| Style | Example | Login |
|---|---|---|
| SSH | `git@github.com:user/repo.git` | your SSH key |
| HTTPS | `https://github.com/user/repo.git` | a token (PAT) |

You can change the address later: `git remote set-url origin <new-address>`.

### The GitHub CLI (optional)

`gh` does the website steps from the terminal:

```bash
brew install gh
gh auth login
gh repo create git-practice --private --source=. --remote=origin --push
```

`gh auth login` logs you in through the browser. `gh repo create … --source=. --push` creates a private repository from the current folder, adds `origin`, and pushes your commits. `gh` also stores its token securely in the system keychain. It is a convenience: everything it does you can do with `git` and the website.

## 8. README, licence and ignore file

A repository on GitHub should open with a **README**. Put in it: what the project is, how to build and run it (`cargo run`, `cargo test`), and how to contribute. A **licence** file (`LICENSE`) tells others what they may do with your code; with no licence, they may not legally reuse it. Rust projects often use the dual licence "MIT OR Apache-2.0" (MIT, named after the Massachusetts Institute of Technology, is a short permissive licence; Apache 2.0 is another). For a private learning repository you do not need one.

## 9. Keep secrets out of the remote

A remote is shared, and public repositories are read by robots that look for secrets within minutes.

- **Never commit** passwords, API (Application Programming Interface) keys, tokens, `.env` files or private keys. Add them to `.gitignore` before the first commit.
- If a secret **was** pushed, assume it is stolen: **revoke it** at the service (delete the token, rotate the key) and create a new one. Deleting the file in a new commit does not help ([Lesson 01.5](05-undoing-changes-safely.md)). GitHub also scans pushes for known token formats and may warn you.
- Prefer a **private** repository for anything unfinished.
- Turn on 2FA and use a password manager.
- A passphrase on your SSH key is cheap insurance.

> 🔒 **Security:** a Personal Access Token is as powerful as a password for the permissions you gave it. Give it the smallest permissions and a short expiry date. Never paste it into a chat or commit it.

## Common mistakes

> ⚠️ **Common mistake:** `error: remote origin already exists.` You ran `git remote add origin …` twice. Change the address instead: `git remote set-url origin <address>`, or remove it with `git remote remove origin`.

> ⚠️ **Common mistake:** `fatal: 'nowhere' does not appear to be a git repository` followed by `Could not read from remote repository`. The remote name or address is wrong. Check `git remote -v`.

> ⚠️ **Common mistake:** `! [rejected] main -> main (fetch first)` or `(non-fast-forward)`. The remote has commits you do not have. `git fetch`, merge (or `git pull`), then push again. Never `--force` on a shared branch.

> ⚠️ **Common mistake:** `Permission denied (publickey)`. The key is not on your account, not loaded, or you used the wrong address. Run `ssh -T git@github.com` to test the key alone.

> ⚠️ **Common mistake:** HTTPS asks for a password and then fails with an authentication error. GitHub does not accept account passwords for Git. Use SSH, or a token as the password.

> ⚠️ **Common mistake:** creating the GitHub repository with a README and then trying to push your own history. The two histories are unrelated and Git refuses (`refusing to merge unrelated histories`). Create the remote repository **empty**.

> ⚠️ **Common mistake:** committing as one identity and logging in as another. The commit author comes from `user.email`, not from your login. Set `user.email` to the address (or noreply address) of your account.

## Try it yourself

Part A — works offline:

1. Repeat the laboratory of sections 2–4 from the start, in a new folder `remote-lab2`.
2. In `ada` make a second commit and push it; in `bob` run `git fetch` and then `git status`: what does it say?
3. In `bob`, run `git pull`. What does the output say? Why is it a fast-forward?
4. Run `git branch -vv` in both folders.

Part B — needs a GitHub account:

5. Create the account, turn on 2FA.
6. Create an SSH key (use a passphrase), add it to `ssh-agent`, add the public key to GitHub, and check it with `ssh -T git@github.com`.
7. Create an **empty private** repository `git-practice`, connect `hello-rust` or `branch-demo` to it, and push.
8. Look at the repository page: commits, files, the number of branches.

## Exercises

### Exercise 1 — A bare remote and a clone (easy)

Create a bare repository `team.git`, clone it as `me`, make a commit, and push. Prove the commit is in the bare repository without cloning again.

<details>
<summary>Hint</summary>

`git init --bare`, `git clone`, then `git -C team.git log --oneline main` shows the log of another folder.

</details>

<details>
<summary>Answer</summary>

```bash
cd ~/projects/git-playground/remote-lab
git init --bare team.git
git clone team.git me
cd me
echo "hello" > hello.txt
git add hello.txt
git commit -m "Add hello"
git push -u origin main
git -C ../team.git log --oneline main
```

Git warns `You appear to have cloned an empty repository.` and the last command prints the commit `Add hello` (with your own hash).

Why it works: cloning an empty repository gives you an empty local repository with `origin` already set and `main` as its first branch (because of `init.defaultBranch`); the first push creates `main` on the remote. `git -C <folder>` runs Git as if you were in that folder.

</details>

### Exercise 2 — Read the status line (easy)

`git status` prints `Your branch is ahead of 'origin/main' by 2 commits.` What happened, and which command is next? What if it says `behind … by 1 commit`?

<details>
<summary>Hint</summary>

Ahead = you have commits the remote lacks. Behind = the remote has commits you lack.

</details>

<details>
<summary>Answer</summary>

*Ahead by 2:* you made two commits that are not on the remote yet. Run `git push`.

*Behind by 1:* someone pushed one commit that you do not have. Run `git pull` (it fast-forwards).

If it says *diverged*, both sides have new commits: fetch, merge `origin/main`, then push.

Why it works: Git compares your branch with the remote-tracking branch `origin/main` (the state at your last fetch). Run `git fetch` first so that this knowledge is fresh.

</details>

### Exercise 3 — Two people, one file (medium)

Using `shared.git`, `ada` and `bob` from the lab: Bob changes `README.md` and pushes. Ada changes a **different** file and tries to push. Show the rejection, then integrate with `fetch` + `merge` and push successfully.

<details>
<summary>Hint</summary>

This is exactly section 4. After `git fetch`, merge `origin/main`.

</details>

<details>
<summary>Answer</summary>

```bash
cd ~/projects/git-playground/remote-lab/bob
echo "Bob again" >> README.md
git commit -am "Bob edits README"
git push
cd ../ada
echo "more notes" >> ada.txt
git commit -am "Ada edits ada.txt"
git push
git fetch
git merge origin/main --no-edit
git push
```

The first `git push` of Ada prints `! [rejected] main -> main (fetch first)`; the last one succeeds.

Why it works: Ada's push would drop Bob's commit, so the server rejects it. After the merge, Ada's branch contains Bob's commit, so the push only adds to the remote history.

</details>

### Exercise 4 — Make a throw-away SSH key (medium)

Generate an Ed25519 key in a **temporary** location (not in `~/.ssh`) with an empty passphrase, only for this exercise. Print its fingerprint. Then delete the key files.

<details>
<summary>Hint</summary>

Use `-f` for the file path and `-N ""` for an empty passphrase. `ssh-keygen -lf <file>.pub` prints the fingerprint.

</details>

<details>
<summary>Answer</summary>

```bash
ssh-keygen -t ed25519 -C "exercise" -f ~/projects/git-playground/test_key -N ""
ssh-keygen -lf ~/projects/git-playground/test_key.pub
rm ~/projects/git-playground/test_key ~/projects/git-playground/test_key.pub
```

```text
256 SHA256:7ygxdFjIz5wiG+h4R5nVMLglbfHnwX10S2l8ZsYq90E exercise (ED25519)
```

(Your fingerprint is different: the key is random.)

Why it works: `-f` chooses the output file and `-N ""` sets an empty passphrase, which is fine for a key you throw away immediately. For a real key, always use a passphrase and keep it in `~/.ssh`.

</details>

### Exercise 5 — Recover from a mistaken remote (hard)

In the `ada` folder of the lab, create the mistake first: `git remote set-url origin git@github.com:wrong/repo.git`. Now show how to see the wrong address, replace it with `../shared.git`, and prove that `git push` works. Also say what to do if you wanted to delete the remote completely.

<details>
<summary>Hint</summary>

`git remote -v`, `git remote set-url`, `git remote remove`.

</details>

<details>
<summary>Answer</summary>

```bash
git remote -v
git remote set-url origin ../shared.git
git remote -v
git push
```

```text
origin	git@github.com:wrong/repo.git (fetch)
origin	git@github.com:wrong/repo.git (push)
origin	../shared.git (fetch)
origin	../shared.git (push)
```

To delete the remote completely: `git remote remove origin` (this removes the name and the `origin/*` notes, not the remote repository itself).

Why it works: a remote is only a name plus an address in `.git/config`. `set-url` changes the address; your commits are not affected.

</details>

## Quiz

1. What is the difference between `git fetch` and `git pull`?

<details>
<summary>Answer</summary>

`fetch` only downloads the news and updates `origin/*`; it does not touch your branches or files. `pull` is `fetch` followed by a merge (with our setting, fast-forward only) into the current branch.

</details>

2. What does `origin/main` mean?

<details>
<summary>Answer</summary>

A remote-tracking branch: your local note of where `main` was on `origin` at the last fetch, push or pull.

</details>

3. Why was Ada's first push rejected?

<details>
<summary>Answer</summary>

The remote `main` had a commit (Bob's) that Ada did not have. Accepting her push would lose it. She must fetch and integrate first.

</details>

4. Why should you never `git push --force` to `main`?

<details>
<summary>Answer</summary>

It overwrites the remote history and destroys other people's commits.

</details>

5. Which key do you give to GitHub, and which one must never leave your computer?

<details>
<summary>Answer</summary>

Give the **public** key (`id_ed25519.pub`). The **private** key (`id_ed25519`) must never leave your computer.

</details>

6. What does `Permission denied (publickey)` usually mean?

<details>
<summary>Answer</summary>

The server did not accept any of your keys: the public key is not added to your account, the key is not loaded in the agent, or you use the wrong address or user.

</details>

7. You committed a token and pushed it. What do you do first?

<details>
<summary>Answer</summary>

Revoke the token at the service and create a new one. Then remove it from the code and the `.gitignore` rules. Deleting it in a later commit is not enough.

</details>

## Summary

- A remote is another copy of the repository; `origin` is the default name. `git clone` makes a full copy.
- `git push` sends your commits, `git fetch` downloads without touching your work, `git pull` is fetch + merge.
- `origin/main` is a remote-tracking branch: your note about the remote.
- A rejected push means the remote has commits you lack: fetch, merge, push. Never force-push to a shared branch.
- You can practise everything with a bare repository in a folder.
- GitHub and GitLab host remotes and add reviews, issues and CI.
- Log in with an SSH key (public key to the service, private key stays with you) and use 2FA.
- Never commit secrets; if one leaks, revoke it.

## Sources and further reading

- [Pro Git — 2.5 Working with Remotes](https://git-scm.com/book/en/v2/Git-Basics-Working-with-Remotes) — the official remote commands.
- [Pro Git — 3.5 Remote Branches](https://git-scm.com/book/en/v2/Git-Branching-Remote-Branches) — remote-tracking branches and upstreams.
- [GitHub Docs — Connecting to GitHub with SSH](https://docs.github.com/en/authentication/connecting-to-github-with-ssh) — generating keys, adding them, testing.
- [GitHub Docs — Securing your account with two-factor authentication](https://docs.github.com/en/authentication/securing-your-account-with-two-factor-authentication-2fa) — turn on 2FA.
- [GitLab Docs — Use SSH keys to communicate with GitLab](https://docs.gitlab.com/user/ssh/) — the same for GitLab.
- [GitHub CLI manual](https://cli.github.com/manual/) — every `gh` command.

---

[🏠 Start](../START-HERE.md) · [← 01.7 Merge conflicts](07-merge-conflicts.md) · [📋 Phase 01](README.md) · [01.9 Teamwork: pull requests, code review and conventions →](09-teamwork-pull-requests-and-conventions.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
