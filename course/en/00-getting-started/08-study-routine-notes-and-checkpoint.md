<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 00.7 Reading errors and getting help](07-reading-errors-and-getting-help.md) · [📋 Phase 00](README.md) · [Phase 01 — Git and version control →](../01-git-and-version-control/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/00-getting-started/08-study-routine-notes-and-checkpoint.md) <!-- nav -->

# Lesson 00.8 — Study routine, notes in Markdown and checkpoint

> **Phase 00 — Getting started** · Lesson 8 of 8 · ⏱ about 135 minutes

## What you will learn

- A weekly study routine that fits about 10 hours and uses how memory really works.
- How to keep a learning journal and review old lessons (spaced repetition).
- How to write notes in Markdown: headings, lists, code, tables and links.
- How to keep your work safe: version control, a backup and a remote copy (a first look at what Phase 01 builds).
- A cheat sheet of everything from Phase 00, and a checkpoint to prove that you are ready for Phase 01.

## Before you start

- You finished [Lesson 00.7 — Reading errors and getting help](07-reading-errors-and-getting-help.md), and so all of Lessons 00.1–00.7.
- The folder `~/projects/rust/notes` exists (Lesson 00.4). Rust and RustRover are installed (Lesson 00.5).

## New words

| Word | Meaning |
|---|---|
| **learning journal** | A short written log of what you learned and what is still unclear, written after every lesson. |
| **active recall** | Trying to remember something *without looking*. It trains memory much better than re-reading. |
| **spaced repetition** | Reviewing a topic again after growing gaps of time (1 day, 3 days, 1 week, 1 month). |
| **Pomodoro technique** | Working in blocks of 25 minutes with 5-minute breaks. |
| **Markdown** | A way to write formatted text in a plain file, using simple marks such as `#` for a heading and `-` for a list item. The files of this course are Markdown files (`.md`). |
| **heading** | A title line in Markdown, starting with one or more `#`. |
| **code block** | A group of lines shown in a fixed-width font. In Markdown it is fenced by three backticks. |
| **backup** | A second copy of your files kept in another place, so that a broken disk or a mistake does not destroy your work. |
| **Time Machine** | The backup program that comes with macOS. |
| **VC** (Version Control) | Keeping the history of your files so you can compare and restore old versions. Phase 01 teaches **Git**, the VC tool. |
| **repository** | A folder that Git tracks, together with its full history. |
| **commit** | One saved snapshot of the tracked files in a repository, with a message. |
| **remote** | A copy of a repository on another computer, such as GitHub or GitLab. |
| **checkpoint** | A test at the end of a phase to confirm that you are ready to move on. |
| **CPU** (Central Processing Unit) | The processor chip that runs instructions (Lesson 00.3). |
| **RAM** (Random Access Memory) | The fast working memory of a computer (Lesson 00.3). |
| **OS** (Operating System) | The main program of a computer that runs other programs; here macOS (Lesson 00.3). |
| **AI** (Artificial Intelligence) assistant | A program such as Claude that answers questions in natural language (Lesson 00.1). |

## 1. A study routine that works

### How memory works, in four lines

1. Re-reading feels productive but works poorly. **Active recall** (closing the page and trying to
   write down or say what you remember) works much better.
2. Reviewing after growing gaps (**spaced repetition**) moves knowledge into long-term memory.
3. **Sleep** stores what you studied. A late-night marathon is worse than two shorter sessions on two days.
4. **Doing** teaches more than reading. Every lesson has code to type and exercises to solve for this reason.

### Your week (about 10 hours)

| Block | Time | What | Rule |
|---|---|---|---|
| 4 study sessions | 4 × 2 h | new lessons and "Try it yourself" | one fixed time of the day |
| 1 practice session | 1 × 1.5 h | exercises and small programs without the lesson open | no hints for the first 15 minutes |
| 1 review session | 1 × 0.5 h | the journal, the quizzes of earlier weeks | active recall |

### One 2-hour session

| Minutes | Activity |
|---|---|
| 0–10 | **Warm-up:** answer from memory 3 questions from the last lesson (quiz at the end of the lesson). |
| 10–55 | **Focus block 1:** read and type the new lesson. |
| 55–65 | **Break:** stand up, water, no phone. |
| 65–105 | **Focus block 2:** the rest of the lesson; start the exercises. |
| 105–120 | **Journal:** write the journal entry (section 2). Decide the next step. |

If you like the **Pomodoro technique**, use 25 minutes of work and 5 minutes of rest instead.

### Rules that save you

- **Same time, same place.** The habit is stronger than motivation.
- **Phone away** during focus blocks.
- **Stop in the middle of something.** Finish the session at a moment when you know exactly what the
  next step is, and write that step in your journal. The next session starts without friction.
- **Use the stuck ladder** from Lesson 00.1 (error message, 15 minutes, hint, answer, ask).
- **Do not skip the quiz.** It is the cheapest active recall there is.
- **A missed day is fine; a missed week is not.** If you miss a session, do a short 30-minute one the next
  day instead of silently dropping the plan.
- **Be kind to yourself.** Frustration is the normal feeling of learning something hard. Take a walk; the answer often comes during it.

## 2. The learning journal and spaced repetition

After each lesson write 5 short lines in `~/projects/rust/notes/`. Use one file per phase, for example
`phase-00.md`. A template:

```text
## Lesson 00.4 — The terminal (2026-11-18)
- Learned: cd, ls, mkdir, redirect (> and >>), pipe (|).
- Confusing: the difference between ~ and /.
- Remember: rm has no Trash. ls first!
- Question: what does the "@" mean in ls -l?
- Next: install Rust.
```

The "Question" line is important. In the next session, search for the answer (docs, the lesson, a
forum, an assistant) and write it down. A list of answered questions is a very valuable personal reference.

### The review schedule

Mark in your calendar a short review after **1 day, 3 days, 1 week and 1 month** after each lesson. In each review
spend 5 minutes:

1. Close the lesson. Read only the heading list of your journal.
2. Try to answer the lesson's quiz questions from memory.
3. Open the quiz answers and mark what you forgot.
4. Retype one command or one small program without looking.

For vocabulary (words, commands, error codes) you can also use **flashcards**: paper cards, or a free app such as
[Anki](https://apps.ankiweb.net/) that schedules the repetitions for you. A good card has one question on the front
("What does `cargo check` do?") and a short answer on the back.

> 💡 **Tip:** After the month of Phase 00 you will have about 8 journal entries. Look at the
> questions you wrote in the first lessons. Most of them you will now be able to answer. This is the best proof of progress.

## 3. Notes in Markdown

All lessons are Markdown files, and your notes should be too. Markdown is plain text with a few marks, so any editor can open it
and Git (Phase 01) can show exactly what changed. Here is a complete example:

~~~markdown
# Phase 00 notes

## Commands I use every day

- `pwd` shows where I am
- `ls -la` lists files, including hidden ones
- `cargo run` builds and runs

### Steps to create a project

1. `cd ~/projects/rust`
2. `cargo new code/p00-hello`
3. `cargo run`

**Important:** `rm` has *no* Trash.

| Command | What it does |
|---|---|
| `cargo check` | checks the code, no executable |
| `cargo fmt` | formats the code |

```rust
fn main() {
    println!("Hello, Rust!");
}
```

See [the Rust book](https://doc.rust-lang.org/book/) for more.

> Note to self: ask what `@` means in `ls -l`.
~~~

The marks you need:

| You write | You get |
|---|---|
| `# Title` | a heading of level 1 (the file title; use only one per file) |
| `## Section`, `### Part` | headings of level 2 and 3 |
| `- item` | a bullet list item |
| `1. step` | a numbered list item |
| `` `code` `` | inline code (commands, file names, keywords) |
| three backticks, then an optional language name (`rust`, `text`), the code lines, and three backticks | a code block with a language |
| `**bold**`, `*italic*` | **bold**, *italic* |
| `[text](https://example.com)` | a link |
| `> note` | a quotation or note |
| `\| a \| b \|` rows with a line `\|---\|---\|` below the first row | a table |

Rules that prevent most problems:

1. Put an **empty line** before and after headings, lists, tables and code blocks.
2. Put a language name after the opening three backticks (` ```rust `): the editor will colour the code.
3. A numbered list may use `1.` for every item; the viewer counts for you.
4. To show three backticks inside a code block (as above), open and close the outer block with `~~~`.
5. Keep lines short and write one idea per paragraph. A blank line starts a new paragraph.

To see the result: open the `.md` file in RustRover, which has a built-in preview (the buttons at the top right of the editor
switch between source, split and preview). In a browser, GitHub and GitLab render Markdown automatically.

> 📏 **Convention:** One file per topic, a descriptive lowercase name with dashes (`phase-00-notes.md`,
> `terminal-cheatsheet.md`), and one `#` title at the top. In a few weeks you will want to search your own notes.

## 4. Keeping your work safe

You will soon have a lot of code and notes. Losing them would hurt, so build three layers of protection.

| Layer | What it protects against | Tool | When |
|---|---|---|---|
| **1. Version control** | mistakes, "what did this look like yesterday?" | **Git** | Phase 01 |
| **2. A local backup** | a broken or stolen laptop | **Time Machine** | today |
| **3. A remote copy** | the same, and a lost external disk | GitHub or GitLab | Phase 01 (Lesson 01.8) |

**Version control** (Git) keeps every saved snapshot (**commit**) of your project in a **repository**. You can
look at the history, compare versions, return to an old one, and work on risky ideas without fear. Phase 01 starts
from zero. From there on, every project in the course ends each milestone with a commit. At the
end of Phase 01 you will turn `~/projects/rust` into your own **learning repository** (Lesson 01.10).

> ⚠️ **Common mistake:** Thinking that Git alone is a backup. A repository lives on *your* disk, next to
> your files. If the disk dies, history dies too. Only a **remote** (a copy on GitHub or GitLab) or a separate backup
> protects you. You need both: Git for history, a backup for disasters.

**Time Machine** is built into macOS and needs an external disk. Open **System Settings → General →
Time Machine**, add the disk and turn it on; macOS then makes automatic hourly backups. Apple's page about it is in the
sources below. It protects not only the course folder but everything else on your Mac.

Some folder rules, which also prepare you for Git:

- **Code in `code/`, notes in `notes/`.** Do not mix them with the course folders.
- **No secrets in notes or code:** passwords, tokens, private keys. Whatever lands in a repository on GitHub can be read by others, and history keeps it even after you delete the line.
- **Do not store build output.** The `target/` folders of Cargo can be recreated; Cargo already gives each new project a `.gitignore` for that (Lesson 00.6).
- **Name files clearly.** `notes-final-2-new.md` is a sign that you need version control. (This is why Phase 01 comes first.)

Check what is ready already:

```text
% git --version
git version 2.54.0 (Apple Git-157)
```

If this works (Lesson 00.5), Phase 01 will only ask you to set your name and e-mail in Git (Lesson 01.2).

## 5. The Phase 00 cheat sheet

Terminal (Lesson 00.4):

| Command | Meaning |
|---|---|
| `pwd` | where am I |
| `ls`, `ls -la` | list files (long, with hidden) |
| `cd dir`, `cd ..`, `cd ~` | go into, up, home |
| `mkdir -p a/b` | make folders |
| `touch f`, `echo text > f`, `echo text >> f`, `cat f` | create, write, append, read |
| `cp a b`, `mv a b`, `rm f`, `rm -r d` | copy, move/rename, delete (final!) |
| `man x`, `which x`, `type x` | help, where is a command |
| `a \| b`, `grep`, `wc -l`, `head`, `find` | pipe and filter |
| Ctrl + C, Tab, ↑ | stop, complete, history |

Rust and Cargo (Lessons 00.5 and 00.6):

| Command | Meaning |
|---|---|
| `rustc --version`, `cargo --version` | versions |
| `rustup update`, `rustup doc` | update; open the offline docs |
| `cargo new code/pNN-name` | new project |
| `cargo run`, `cargo build`, `cargo check` | run, build, fast check |
| `cargo build --release` | optimised build |
| `cargo fmt`, `cargo clippy` | format, lint |

Errors (Lesson 00.7):

1. Read the **first** error. 2. `file:line:column` shows where. 3. Fix one thing, run again.
4. `rustc --explain E....` explains the code. 5. Warnings are for fixing. 6. A panic gives exit code 101.
7. A good question: goal, minimal code, exact error, version. Never paste secrets.

## Common mistakes

1. **Copying the notes of others instead of writing your own.** You do not remember what you did not put into words. *Fix:* write the journal in your own words, even if it is short.
2. **Skipping reviews.** The topic feels clear on the day and is gone in two weeks. *Fix:* put the 1-3-7-30 day reviews in the calendar.
3. **Notes in many places** (paper, phone, five files). *Fix:* one folder, `notes/`, in Markdown.
4. **No blank line before a Markdown list or table.** It renders as one ugly paragraph. *Fix:* add an empty line.
5. **Forgetting the backup.** "It will not happen to me" is how data gets lost. *Fix:* turn on Time Machine this week.

## Try it yourself

1. Make a new file `~/projects/rust/notes/phase-00.md` in RustRover. Copy the journal template from section 2 and fill in **Lesson 00.1** as a trial entry: what you learned, what was confusing, one thing to remember, one question, the next step.
2. Open the Markdown preview. Does the heading look like a heading? Does the list look like a list?
3. Add a table with three commands you want to remember and a code block with your `Hello, world!` program.
4. Write the dates of your four reviews (1, 3, 7 and 30 days from today) into your calendar or into the file.
5. Open System Settings and check whether Time Machine is turned on. If not, decide today when you will set it up.
6. Run the toolchain report from Practical task 2 below.

## Checkpoint — are you ready for Phase 01?

### Self-test quiz

Answer each question from memory first, then open the answer. If you miss a question, the table at the end
tells you which lesson to repeat.

1. What is a systems engineer, in one sentence?

<details>
<summary>Answer</summary>

A programmer who builds and looks after the lower layers of software (servers, databases, tools, parts of operating systems), where speed, memory use and reliability matter.

</details>

2. Why does the course teach Git (Phase 01) before the Rust language?

<details>
<summary>Answer</summary>

From Phase 01 on, every project is saved with a commit per milestone. Version control keeps the history of your files and protects your work.

</details>

3. Name the six building blocks of a program.

<details>
<summary>Answer</summary>

Input, output, storing (variables), decisions (conditions), repetition (loops), reuse (functions).

</details>

4. What is the difference between a compile error and a logic bug?

<details>
<summary>Answer</summary>

A compile error stops the build and the compiler explains where. A logic bug lets the program run but gives a wrong result, with no message.

</details>

5. Which part of the computer runs instructions, which holds the running program, and which keeps files when the power is off?

<details>
<summary>Answer</summary>

The CPU runs instructions, RAM holds the running program and its data, the storage (SSD) keeps files permanently.

</details>

6. Convert `00101010` to decimal.

<details>
<summary>Answer</summary>

32 + 8 + 2 = 42.

</details>

7. What is machine code, and why does a program built on your M4 Mac not run on an x86-64 PC?

<details>
<summary>Answer</summary>

Machine code is the instructions as numbers that the CPU reads directly. Each CPU family (arm64, x86-64) has its own instruction set, so the numbers differ.

</details>

8. What are the steps from source file to output on the screen?

<details>
<summary>Answer</summary>

The compiler (`rustc`) translates the source into an executable file; the OS loads it into RAM as a process; the CPU executes its instructions; the output appears.

</details>

9. What is the difference between `>` and `>>`, and between `rm` and `rmdir`?

<details>
<summary>Answer</summary>

`>` replaces the content of a file with the output; `>>` appends. `rm` deletes files (and directories with `-r`); `rmdir` deletes only an empty directory.

</details>

10. What is `PATH`?

<details>
<summary>Answer</summary>

An environment variable with the list of directories where the shell looks for commands. The Rust installer adds `~/.cargo/bin` to it.

</details>

11. What must `rustc -vV` show in the `host` line on your Mac?

<details>
<summary>Answer</summary>

`host: aarch64-apple-darwin`.

</details>

12. What are the files `Cargo.toml`, `Cargo.lock`, `src/main.rs` and the folder `target` for?

<details>
<summary>Answer</summary>

`Cargo.toml` holds the project settings; `Cargo.lock` records the exact versions used (Cargo writes it); `src/main.rs` is the source code where the program starts; `target` holds the build results (ignored by Git).

</details>

13. What is the difference between `cargo run`, `cargo build`, `cargo check` and `cargo build --release`?

<details>
<summary>Answer</summary>

`run` builds and starts; `build` only builds (debug profile); `check` only analyses the code, no executable; `build --release` builds the optimised version.

</details>

14. What does the `!` mean in `println!`?

<details>
<summary>Answer</summary>

`println` is a macro, not an ordinary function.

</details>

15. You get seven errors. Where do you start, and how?

<details>
<summary>Answer</summary>

At the first error. Fix one thing, run again, and repeat; later errors are often caused by earlier ones.

</details>

16. What exit code does a panic produce, and how do you see it?

<details>
<summary>Answer</summary>

101; run `echo $?` right after the program.

</details>

17. Why are 1, 3, 7 and 30 days a good review schedule?

<details>
<summary>Answer</summary>

Reviewing just as you begin to forget (spaced repetition) strengthens memory far more than re-reading everything at once.

</details>

18. Is a Git repository on your laptop a backup? Why?

<details>
<summary>Answer</summary>

No. It lives on the same disk as your files. You also need a remote copy (GitHub/GitLab) or a separate backup such as Time Machine.

</details>

### Practical tasks

Do these on your Mac. Each has a hint and an answer.

#### Task 1 — Build the folder structure (terminal)

Using only the terminal, from `~/projects/rust`: make sure `code` and `notes` exist, create
`notes/checkpoint/` and in it a file `a.md` containing the line `Checkpoint 00`, then show the file, count its lines and finally
delete the folder `notes/checkpoint` together with the file.

<details>
<summary>Hint</summary>

`mkdir -p`, `echo ... >`, `cat`, `wc -l`, and `rm -r` (check with `ls` first).

</details>

<details>
<summary>Answer</summary>

```text
% cd ~/projects/rust
% mkdir -p code notes/checkpoint
% echo 'Checkpoint 00' > notes/checkpoint/a.md
% cat notes/checkpoint/a.md
Checkpoint 00
% wc -l notes/checkpoint/a.md
       1 notes/checkpoint/a.md
% ls notes/checkpoint
a.md
% rm -r notes/checkpoint
```

Why it works: `-p` creates the missing chain of folders; `>` writes the file; `ls` before `rm -r` confirms what will be deleted.

</details>

#### Task 2 — A toolchain report

Create `notes/phase-00-report.md` that contains a title, an empty line and the output of `rustc --version`,
`cargo --version`, the `host` line of `rustc -vV`, and `git --version`, using redirects.

<details>
<summary>Hint</summary>

The first command uses `>` (it creates the file), the next ones use `>>` (they append). A pipe with `grep host` selects the right line.

</details>

<details>
<summary>Answer</summary>

```text
% cd ~/projects/rust
% echo '# Phase 00 report' > notes/phase-00-report.md
% echo '' >> notes/phase-00-report.md
% rustc --version >> notes/phase-00-report.md
% cargo --version >> notes/phase-00-report.md
% rustc -vV | grep host >> notes/phase-00-report.md
% git --version >> notes/phase-00-report.md
% cat notes/phase-00-report.md
# Phase 00 report

rustc 1.100.0 (0ef1bc386 2026-11-09)
cargo 1.100.0 (3acc79c0c 2026-11-05)
host: aarch64-apple-darwin
git version 2.54.0 (Apple Git-157)
```

Why it works: each command prints to the screen, and `>>` sends that output to the end of the file instead. Your version numbers may be newer.

</details>

#### Task 3 — A new project with formatting and lint

Create `code/p00-checkpoint`. It must print this card, and `cargo fmt --check` and `cargo clippy` must show
no problems:

```text
Phase 00 study card
-------------------
Compiler: rustc
Build tool: cargo
Shell: zsh
```

<details>
<summary>Hint</summary>

`cargo new code/p00-checkpoint`, then five `println!` lines in `src/main.rs`.

</details>

<details>
<summary>Answer</summary>

```rust
fn main() {
    println!("Phase 00 study card");
    println!("-------------------");
    println!("Compiler: rustc");
    println!("Build tool: cargo");
    println!("Shell: zsh");
}
```

Then `cargo run`, `cargo fmt --check` and `cargo clippy`: the first prints the card, the other two print nothing wrong.

Why it works: every line is a separate `println!` with text in double quotes and a `;` at the end.

</details>

#### Task 4 — Repair a program, step by step

This program has three mistakes. Follow the routine from Lesson 00.7 and record how many errors you saw in each round.

```rust
fn main() {
    let tool = "cargo"
    println("Tool: {}", tol);
}
```

<details>
<summary>Hint</summary>

Fix the error that is reported first. The mistakes are: a missing character at the end of a line, a missing character in a macro name, and a misspelled variable name.

</details>

<details>
<summary>Answer</summary>

Round 1: 3 errors (`expected ;`, `cannot find value tol`, `cannot find function println`). Fix: add `;` after `"cargo"`.
Round 2: 2 errors. Fix: `println` → `println!`.
Round 3: 1 error (`E0425`, `cannot find value tol`). Fix: `tol` → `tool`.

```rust
fn main() {
    let tool = "cargo";
    println!("Tool: {}", tool);
}
```

Output:

```text
Tool: cargo
```

Why it works: each round removes one cause, and the next run shows what remains. The compiler's suggestion for `tol` (use `tool`) was right here because the typo was in the *use*, not in the declaration.

</details>

#### Task 5 — Your notes (Markdown)

In `notes/phase-00.md` write a summary of Phase 00 with: a `#` title, at least two `##` sections, a bullet list, a numbered
list, a table, a code block with a language, and a link. Open the preview and check it.

<details>
<summary>Hint</summary>

Use the example in section 3 as a model. Remember the empty lines around the blocks.

</details>

<details>
<summary>Answer</summary>

There is no single correct answer. Check your file against this list: one `#` title; two or more `##` headings; a list with `-`; a list with `1.`; a table with a `|---|---|` line under the first row; a code block that starts with three backticks and `rust` or `text`; a link in the form `[text](https://...)`; blank lines around all blocks; the preview shows no raw `#` or `|` characters.

Why it works: Markdown is only a set of marks, and the preview shows at once if one of them is wrong.

</details>

### If you got something wrong

| Topic | Review this lesson |
|---|---|
| Goals, structure, folders, AI assistant | [Lesson 00.1 — How to use this course](01-how-to-use-this-course.md) |
| Program, algorithm, bug types, tools | [Lesson 00.2 — What programming is](02-what-is-programming.md) |
| CPU, RAM, binary, machine code, compiler, OS | [Lesson 00.3 — How a computer runs a program](03-how-a-computer-runs-a-program.md) |
| Terminal commands, paths, `PATH`, permissions | [Lesson 00.4 — The terminal: your first commands](04-terminal-basics.md) |
| `rustup`, versions, `host`, RustRover, Git check | [Lesson 00.5 — Installing Rust and RustRover](05-install-rust-and-rustrover.md) |
| Cargo, `println!`, `cargo fmt`, `cargo clippy` | [Lesson 00.6 — Your first Rust program](06-your-first-rust-program.md) |
| Compiler errors, panics, asking for help | [Lesson 00.7 — Reading errors and getting help](07-reading-errors-and-getting-help.md) |
| Study routine, Markdown, backups | Section 1–4 of this lesson |

If you can do all the practical tasks and answer at least 15 of the 18 questions, you are ready.

### What's next

[Phase 01 — Git and version control](../01-git-and-version-control/01-what-is-version-control.md) begins with the idea that you met in this phase: a tool
that remembers every version of your files. You will install and configure Git, make your first commits, look at the history, undo
mistakes safely, work with branches, publish to GitHub or GitLab, and turn `~/projects/rust` into your own
learning repository. After that, in [Phase 02](../02-programming-basics/01-variables-and-mutability.md),
the Rust language begins for real.

## Summary

- Study in fixed sessions of about 2 hours, 4–5 times a week; warm up with active recall, stop at a clear next step.
- Write 5 lines in your learning journal after each lesson; review after 1, 3, 7 and 30 days.
- Markdown is plain text with marks: `#` headings, `-` lists, backticks for code, `|` tables, `[text](https://example.com)` links; leave empty lines around blocks.
- Protect your work in three layers: version control (Git, Phase 01), a local backup (Time Machine) and a remote copy (GitHub or GitLab).
- Git is not a backup by itself; a copy on another machine is.
- Keep code in `code/`, notes in `notes/`, and never put secrets into either.
- The checkpoint covers the terminal, the toolchain, a first Cargo project, reading errors and good habits.
- Next: Phase 01, Git and version control.

## Sources and further reading

- [CommonMark — Markdown help](https://commonmark.org/help/) — a quick reference and a 10-minute interactive tutorial for Markdown.
- [Markdown Guide — Basic Syntax](https://www.markdownguide.org/basic-syntax/) — the basic Markdown marks with examples.
- [GitHub Flavored Markdown spec](https://github.github.com/gfm/) — the exact rules for tables, task lists and other extensions that GitHub and GitLab render.
- [Back up your Mac with Time Machine](https://support.apple.com/guide/mac-help/mh35860/mac) — Apple's official guide to setting up backups.
- [Anki](https://apps.ankiweb.net/) — a free flashcard application with spaced repetition.
- [Pro Git](https://git-scm.com/book/en/v2) — the free official Git book; the main reference of Phase 01.

---

[🏠 Start](../START-HERE.md) · [← 00.7 Reading errors and getting help](07-reading-errors-and-getting-help.md) · [📋 Phase 00](README.md) · [Phase 01 — Git and version control →](../01-git-and-version-control/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
