<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 01.6 Branches and merging](06-branches-and-merging.md) · [📋 Phase 01](README.md) · [01.8 Remote repositories: GitHub and GitLab →](08-remotes-github-and-gitlab.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/01-git-and-version-control/07-merge-conflicts.md) <!-- nav -->

# Lesson 01.7 — Merge conflicts

> **Phase 01 — Git and version control** · Lesson 7 of 10 · ⏱ about 120 minutes

## What you will learn

- Why merge conflicts happen, and why they are normal and not a disaster.
- How to read the conflict markers `<<<<<<<`, `|||||||`, `=======` and `>>>>>>>`.
- The exact steps to resolve a conflict and finish the merge, and how to abort.
- How to choose one whole side, how to handle "modified here, deleted there", and what to do with conflicts in `Cargo.lock`.
- How to make conflicts rarer.

## Before you start

- You finished [Lesson 01.6 — Branches and merging](06-branches-and-merging.md).
- You set `merge.conflictStyle zdiff3` in [Lesson 01.2](02-install-and-configure-git.md). (If you did not, the markers below will show one part less. Everything else is the same.)

## New words

| Word | Meaning |
|---|---|
| **merge conflict** | A situation where Git cannot combine two changes by itself, because both branches changed the same lines (or one changed a file that the other deleted). |
| **conflict markers** | Special lines (`<<<<<<<`, `|||||||`, `=======`, `>>>>>>>`) that Git writes into the file to show the competing versions. |
| **ours** | In a merge: the version from the branch you are on (the receiving branch). |
| **theirs** | In a merge: the version from the branch you merge in. |
| **base** | The version in the common ancestor: how the text looked before either side changed it. |
| **unmerged path** | A file with an unresolved conflict. |
| **resolve** | To edit the file into its final form, remove the markers and tell Git with `git add`. |
| **merge tool** | A program with a visual view of the conflict (RustRover has one). |

## 1. Why conflicts happen

In [Lesson 01.6](06-branches-and-merging.md) Git merged two branches by itself, because they changed **different** files. Git compares both branch tips with their common ancestor:

- If only one side changed a line, Git takes that change.
- If **both** sides changed **the same place** in different ways, Git cannot know which is right. It stops and asks you.

That is all a conflict is: a question from Git. It is not an error, and nothing is lost. Developers on active teams meet conflicts every week.

## 2. Create a conflict

Make a repository where two branches change the same line. Paste the block (it is just a setup):

```bash
cd ~/projects/git-playground
mkdir conflict-demo
cd conflict-demo
git init
printf 'fn main() {\n    println!("Welcome");\n    println!("Have a nice day");\n}\n' > main.rs
git add main.rs
git commit -m "Add main.rs"

git switch -c polite
printf 'fn main() {\n    println!("Welcome, dear guest");\n    println!("Have a nice day");\n}\n' > main.rs
git commit -am "Make the welcome polite"

git switch main
printf 'fn main() {\n    println!("Welcome to Rust");\n    println!("Have a nice day");\n}\n' > main.rs
git commit -am "Mention Rust in the welcome"
```

**What happens:** the first commit has a line `Welcome`. On branch `polite` this line becomes `Welcome, dear guest`. On `main` the **same line** becomes `Welcome to Rust`. (The Rust code is only text here; you will learn it in Phase 02.)

```text
              ┌── Make the welcome polite ◀── polite
 Add main.rs ◀┤
              └── Mention Rust in the welcome ◀── main (HEAD)
```

Now try to merge:

```bash
git merge polite
```

```text
Auto-merging main.rs
CONFLICT (content): Merge conflict in main.rs
Automatic merge failed; fix conflicts and then commit the result.
```

**What happens:** Git tried to combine the files, and failed on `main.rs`. The merge is now **in progress**: it is paused, waiting for you. Nothing is broken. Look at the situation:

```bash
git status
```

```text
On branch main
You have unmerged paths.
  (fix conflicts and run "git commit")
  (use "git merge --abort" to abort the merge)

Unmerged paths:
  (use "git add <file>..." to mark resolution)
	both modified:   main.rs

no changes added to commit (use "git add" and/or "git commit -a")
```

The short form shows `UU main.rs` (`UU` = unmerged on both sides). `git status` always tells you the two ways out: fix and commit, or abort.

## 3. Read the markers

Git wrote both versions into the file. Open it:

```bash
cat main.rs
```

```text
fn main() {
<<<<<<< HEAD
    println!("Welcome to Rust");
||||||| 65e17d3
    println!("Welcome");
=======
    println!("Welcome, dear guest");
>>>>>>> polite
    println!("Have a nice day");
}
```

The conflict block has up to four parts:

| Marker | What follows |
|---|---|
| `<<<<<<< HEAD` | **Ours**: the lines from your current branch (`HEAD` is where you are, `main`). |
| `\|\|\|\|\|\|\| 65e17d3` | **Base**: the lines in the common ancestor (commit `65e17d3`). Only with `zdiff3`. |
| `=======` | A divider. |
| (lines after the divider) | **Theirs**: the lines from the branch you merge in (`polite`). |
| `>>>>>>> polite` | The end of the block, with the name of the other branch. |

Everything **outside** the markers (`fn main() {`, `Have a nice day`, `}`) merged without trouble.

Reading it: the original line was `Welcome`. On your side it became `Welcome to Rust`. On their side it became `Welcome, dear guest`. The base is the key: it lets you see **what each side changed**. Without `zdiff3` Git leaves the base out, and you must guess.

> 📏 **Convention:** the middle part (base) makes conflicts much easier to understand. That is why Lesson 01.2 sets `merge.conflictStyle zdiff3`.

## 4. Resolve it in five steps

1. **Understand** what each side wanted. Here: one wanted "to Rust", the other wanted "dear guest".
2. **Decide** the final text. It may be one side, the other side, both, or something new. Here we want both ideas: `Welcome to Rust, dear guest`.
3. **Edit the file** to the final text. **Delete the markers and the rejected lines.** Use RustRover or any editor. In the terminal:

```bash
printf 'fn main() {\n    println!("Welcome to Rust, dear guest");\n    println!("Have a nice day");\n}\n' > main.rs
git diff --check
```

`git diff --check` prints nothing: no conflict markers remain. (Before the fix it printed `leftover conflict marker` for every marker line.)

4. **Tell Git** the file is resolved by staging it:

```bash
git add main.rs
git status
```

```text
On branch main
All conflicts fixed but you are still merging.
  (use "git commit" to conclude merge)

Changes to be committed:
	modified:   main.rs

```

5. **Finish** the merge with a commit:

```bash
git commit --no-edit
git log --oneline --graph
```

```text
[main b37cecc] Merge branch 'polite'
*   b37cecc Merge branch 'polite'
|\
| * a27ff71 Make the welcome polite
* | 8b15433 Mention Rust in the welcome
|/
* 65e17d3 Add main.rs
```

**What happens:** `git add` means "this file is now resolved". `git commit` creates the merge commit with two parents. `--no-edit` keeps the default message `Merge branch 'polite'`. (`git merge --continue` does the same.) Done.

> ⚠️ **Common mistake:** committing a file that still has markers. Git does **not** check the content: if you run `git add` with markers inside, they get committed. The Rust compiler will then complain, as here:

```text
error: encountered diff marker
 --> main.rs:2:1
  |
2 | <<<<<<< HEAD
  | ^^^^^^^ between this marker and `=======` is the code that you are merging into
3 |     println!("Welcome to Rust");
4 | =======
  | ------- between this marker and `>>>>>>>` is the incoming code
5 |     println!("Welcome, dear guest");
6 | >>>>>>> polite
  | ^^^^^^^ this marker concludes the conflict region
  |
  = note: conflict markers indicate that a merge was started but could not be completed due to merge conflicts
          to resolve a conflict, keep only the code you want and then delete the lines containing conflict markers
…
error: aborting due to 1 previous error
```

This is the real message from the compiler (`rustc`). It means: you forgot to remove the markers. Before `git add`, run `git diff --check`, or search for `<<<<<<<` in the project.

## 5. Abort: go back to before the merge

If the conflict is too confusing, or you started the merge by mistake, abort. Everything returns to the state before `git merge`.

> 💡 **Tip:** to try sections 5 and 6 you need a merge that is *in progress*. If you already finished the merge in section 4, undo the merge commit with `git reset --hard HEAD~1` (safe here, because the merge commit is the last commit and nothing else is uncommitted), and run `git merge polite` again.

Now abort:

```bash
git merge --abort
git status -s
```

(Nothing is printed: the working tree is clean.) You can start the merge again later. This is the "undo" button for a merge that has conflicts.

## 6. Take one whole side

Sometimes you want one version completely. During a merge:

- **ours** = the branch you are on (`main`);
- **theirs** = the branch you are merging in (`polite`).

```bash
git merge polite
git checkout --ours main.rs
cat main.rs
git checkout --theirs main.rs
cat main.rs
git status -s
```

```text
Updated 1 path from the index
fn main() {
    println!("Welcome to Rust");
    println!("Have a nice day");
}
Updated 1 path from the index
fn main() {
    println!("Welcome, dear guest");
    println!("Have a nice day");
}
UU main.rs
```

**What happens:** `git checkout --ours <file>` replaces the file with your side, `--theirs` with the other side, and both **remove the markers**. The status is still `UU`, because you have not told Git yet. After choosing, run `git add main.rs` and `git commit --no-edit` as before. (`git restore --ours main.rs` is the newer spelling of the same idea.)

> ⚠️ **Common mistake:** in a *rebase* (Lesson 01.9) the meanings of "ours" and "theirs" are **reversed** compared to a merge. If you use them, double-check the result with `cat`.

## 7. Other kinds of conflict: modified here, deleted there

If one branch changes a file and the other branch deletes it, Git cannot choose either:

```text
CONFLICT (modify/delete): main.rs deleted in cleanup and modified in HEAD.  Version HEAD of main.rs left in tree.
Automatic merge failed; fix conflicts and then commit the result.
```

```text
$ git status -s
UD main.rs
```

`UD` means "unmerged: deleted by them". The file is left in your folder. You decide:

- **Keep the file:** `git add main.rs`.
- **Delete the file:** `git rm main.rs`.

Then `git commit --no-edit` finishes the merge. There are similar messages for a file that two branches both **added** with different content (`add/add`) and for renames. The recipe is the same: understand, decide, `git add` or `git rm`, commit.

## 8. Conflicts in generated files, such as `Cargo.lock`

`Cargo.lock` is a file that Cargo writes. Two branches that add dependencies can both change it, and a conflict inside it is hard to read. Do **not** edit it by hand. A common safe recipe:

1. First resolve `Cargo.toml` (the human-written file) by hand.
2. Take either version of the lock file (`git checkout --ours Cargo.lock`).
3. Run `cargo build`. Cargo updates `Cargo.lock` to match the merged `Cargo.toml`.
4. `git add Cargo.lock Cargo.toml` and commit.

The same idea works for any generated file: resolve the **source**, regenerate the generated file, never hand-merge it.

## 9. The graphical way: RustRover

RustRover has a merge tool. When a merge has conflicts, a dialog lists the files (or open *Git → Resolve Conflicts*). For each file you choose *Merge…* and see three columns: your version (left), the result (middle) and their version (right). Little arrows `»` and `«` accept a change from one side; `✕` ignores it. When the result is right, click *Apply*. RustRover stages the file for you. You still finish with a commit.

It does the same thing as the text steps above. Learn the text version first: it works everywhere, including over SSH (Secure Shell, a way to work in the terminal of a remote computer) on a server.

## 10. Make conflicts rarer

Conflicts cannot be removed, but you can reduce them:

- **Small, short-lived branches.** The less time two branches live apart, the less chance they touch the same lines.
- **Integrate often.** Merge `main` into your branch regularly, so differences stay small.
- **Small, focused commits and files.** Do not mix reformatting with real changes in one commit. If you must reformat, do it alone and quickly.
- **Use the formatter.** `cargo fmt` gives everyone the same layout, so conflicts about spaces and line breaks disappear.
- **Talk to the team.** If two people must edit the same function, agree who goes first.

## Common mistakes

> ⚠️ **Common mistake:** deleting the wrong lines and losing the other person's work. Read the block: **both** sides may contain code you need. When unsure, keep both, build and test.

> ⚠️ **Common mistake:** forgetting to run `git add` after fixing the file. `git commit` then says `error: Committing is not possible because you have unmerged files.` Stage the file first.

> ⚠️ **Common mistake:** panicking and running `git reset --hard`. It would work for a merge in progress, but `git merge --abort` is the safe tool for this job.

> ⚠️ **Common mistake:** resolving the conflict textually, but the program no longer compiles or the tests fail. A resolved file must still be correct. Build and test (`cargo build`, `cargo test`) **before** you commit the merge.

> 🧪 **Testing:** after resolving a conflict, run the project's tests. A merge that applies "cleanly" can still be logically wrong; a resolved conflict is even more likely to be.

## Try it yourself

1. Run the setup block from section 2 in a new folder. Use the name `conflict-demo2` instead of `conflict-demo`, so the old practice folder stays untouched.
2. Run `git merge polite` and look at `git status`.
3. Open `main.rs` and identify the four marker lines.
4. Run `git merge --abort` and check `cat main.rs` (it is back to the `main` version).
5. Merge again. This time resolve it to `Welcome to Rust, dear guest`, run `git diff --check`, `git add`, `git commit --no-edit`.
6. Run `git log --oneline --graph` and find the merge commit.

## Exercises

### Exercise 1 — Read the block (easy)

This block is in `README.md` after a merge. What was in the original text, what did "ours" write and what did "theirs" write? Write the final file if you want to keep **both** sentences.

```text
# Project
<<<<<<< HEAD
This tool is fast.
||||||| a1b2c3d
This tool works.
=======
This tool is safe.
>>>>>>> feature/safety
```

<details>
<summary>Hint</summary>

The part between `|||||||` and `=======` is the base: the text before either side changed it.

</details>

<details>
<summary>Answer</summary>

- Original (base): `This tool works.`
- Ours (`HEAD`): `This tool is fast.`
- Theirs (`feature/safety`): `This tool is safe.`

A final file that keeps both ideas:

```text
# Project
This tool is fast and safe.
```

(Two lines, `This tool is fast.` and `This tool is safe.`, are also correct. What matters is that the markers and the base are gone and the text is what you want.)

Why it works: both authors replaced the same line, so Git cannot choose. You combine their intentions by hand.

</details>

### Exercise 2 — Resolve and finish (easy)

Create the conflict from section 2 and resolve it by keeping the text `Welcome to Rust, dear guest`. End with a clean `git status`.

<details>
<summary>Hint</summary>

Edit the file (or rewrite it with `printf`), then `git add`, then `git commit --no-edit`.

</details>

<details>
<summary>Answer</summary>

```bash
git merge polite
printf 'fn main() {\n    println!("Welcome to Rust, dear guest");\n    println!("Have a nice day");\n}\n' > main.rs
git diff --check
git add main.rs
git commit --no-edit
git status
```

```text
On branch main
nothing to commit, working tree clean
```

Why it works: after the merge command the repository is in the "merging" state. Editing the file, `git add` and `git commit` complete the merge.

</details>

### Exercise 3 — Abort and retry (medium)

Start the conflicting merge, then change your mind and abort. Prove that nothing changed: the branch, the file and `git status`. Then run the merge again. Before you start, make sure that `main` has not merged `polite` yet and that no merge is in progress: if `git log --oneline -1` shows `Merge branch 'polite'`, run `git reset --hard HEAD~1`; if `git status` says `You have unmerged paths`, run `git merge --abort`.

<details>
<summary>Hint</summary>

`git merge --abort`. Compare `cat main.rs` and `git log --oneline` before and after.

</details>

<details>
<summary>Answer</summary>

```bash
git merge polite
git merge --abort
git status
cat main.rs
git merge polite
```

```text
On branch main
nothing to commit, working tree clean
```

`cat main.rs` shows the `Welcome to Rust` version, exactly as before the first merge.

Why it works: `--abort` restores the state from before the merge started, so you can retry as often as you like.

</details>

### Exercise 4 — Take the other side wholesale (medium)

Merge `polite` into `main` again, but instead of editing the file, keep **their** version of the whole file. Finish the merge and print the result. Before you start, make sure that `main` has not merged `polite` yet and that no merge is in progress: if `git log --oneline -1` shows `Merge branch 'polite'`, run `git reset --hard HEAD~1`; if `git status` says `You have unmerged paths`, run `git merge --abort`.

<details>
<summary>Hint</summary>

`git checkout --theirs <file>`, then `git add`, then `git commit --no-edit`.

</details>

<details>
<summary>Answer</summary>

```bash
git merge polite
git checkout --theirs main.rs
git add main.rs
git commit --no-edit
cat main.rs
```

```text
fn main() {
    println!("Welcome, dear guest");
    println!("Have a nice day");
}
```

Why it works: `--theirs` takes the file as it is on the branch that you merged in. Because the file is complete and has no markers, you can stage it right away.

</details>

### Exercise 5 — Modified here, deleted there (hard)

In a fresh repository with `main.rs`, create a branch `cleanup` that deletes `main.rs`, and a commit on `main` that edits it. Merge, and resolve in two different ways: first keep the file, then (after an abort and retry) delete it. Create the new repository next to `conflict-demo`, not inside it: a repository inside another repository only causes confusion.

<details>
<summary>Hint</summary>

The status will be `UD`. Use `git add main.rs` to keep it, or `git rm main.rs` to delete it.

</details>

<details>
<summary>Answer</summary>

```bash
cd ~/projects/git-playground
git init -b main modify-delete
cd modify-delete
printf 'fn main() {}\n' > main.rs
git add main.rs
git commit -m "Add main.rs"
git switch -c cleanup
git rm main.rs
git commit -m "Remove main.rs"
git switch main
echo "// edited" >> main.rs
git commit -am "Edit main.rs"
git merge cleanup
git status -s
git add main.rs
git commit --no-edit
ls
```

The merge prints `CONFLICT (modify/delete)` and `git status -s` shows `UD main.rs`. After `git add`, the merge commit keeps the file, and `ls` lists `main.rs`.

To delete instead, run `git merge --abort`, repeat `git merge cleanup`, then use `git rm main.rs` instead of `git add main.rs`, and `git commit --no-edit`. Then `ls` no longer shows the file.

Why it works: Git cannot tell whether the edit or the deletion should win, so you choose by staging (`git add`, keep) or removing (`git rm`, delete).

</details>

## Quiz

1. When does Git report a merge conflict?

<details>
<summary>Answer</summary>

When both branches changed the same lines differently, or when one branch changed a file that the other deleted. Git cannot choose, so it asks you.

</details>

2. What do `<<<<<<<`, `=======` and `>>>>>>>` mark?

<details>
<summary>Answer</summary>

`<<<<<<<` starts the block and shows your version (ours), `=======` divides your version from theirs, `>>>>>>>` ends the block and names the other branch.

</details>

3. What is the extra `|||||||` section?

<details>
<summary>Answer</summary>

The base: the text in the common ancestor, before either side changed it. It helps you see what each side really changed. It appears with `merge.conflictStyle zdiff3`.

</details>

4. After editing a conflicted file, what two commands finish a merge?

<details>
<summary>Answer</summary>

`git add <file>` (mark it as resolved) and `git commit` (or `git merge --continue`).

</details>

5. How do you cancel a merge that has conflicts?

<details>
<summary>Answer</summary>

`git merge --abort`.

</details>

6. How do you deal with a conflict in `Cargo.lock`?

<details>
<summary>Answer</summary>

Do not edit it by hand. Resolve `Cargo.toml`, take either version of `Cargo.lock`, run `cargo build` to regenerate it, then stage both files.

</details>

7. Name two ways to reduce the number of conflicts.

<details>
<summary>Answer</summary>

Short-lived small branches; merging `main` into your branch often; using `cargo fmt`; not mixing reformatting with other changes; talking to the team.

</details>

## Summary

- A conflict is Git asking you to choose when two sides changed the same place; it is normal and fully reversible.
- Markers: `<<<<<<< HEAD` (ours), `|||||||` (base, with `zdiff3`), `=======`, `>>>>>>> branch` (theirs).
- To resolve: understand, decide, edit out the markers, `git add`, `git commit`. Check with `git diff --check`.
- `git merge --abort` goes back to before the merge.
- `git checkout --ours/--theirs <file>` takes one whole side. In a rebase the meanings swap.
- Modify/delete conflicts are solved with `git add` (keep) or `git rm` (delete).
- Never hand-merge generated files such as `Cargo.lock`: resolve the source and regenerate.
- After resolving, build and test before you finish.

## Sources and further reading

- [Pro Git — 3.2 Basic Branching and Merging (Basic Merge Conflicts)](https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging) — the "Basic Merge Conflicts" part of the official chapter.
- [Pro Git — 7.8 Advanced Merging](https://git-scm.com/book/en/v2/Git-Tools-Advanced-Merging) — `--ours`/`--theirs`, the base view, `git diff` during a merge.
- [git-merge manual](https://git-scm.com/docs/git-merge) — the section "How conflicts are presented".
- [RustRover help — Resolve Git conflicts](https://www.jetbrains.com/help/rust/resolving-conflicts.html) — the visual merge tool.
- [The Cargo Book — Cargo.toml vs Cargo.lock](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html) — what `Cargo.lock` is.

---

[🏠 Start](../START-HERE.md) · [← 01.6 Branches and merging](06-branches-and-merging.md) · [📋 Phase 01](README.md) · [01.8 Remote repositories: GitHub and GitLab →](08-remotes-github-and-gitlab.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
