<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 24.1 Roles for systems engineers](01-roles-for-systems-engineers.md) · [📋 Phase 24](README.md) · [24.3 Reading large codebases →](03-reading-large-codebases.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/24-career-and-open-source/02-building-your-portfolio.md) <!-- nav -->

# Lesson 24.2 — Building your portfolio

> **Phase 24 — Career and open source** · Lesson 2 of 9 · ⏱ about 120 minutes

## What you will learn

- What a portfolio is, who reads it, and how much time they spend on it.
- How to choose two or three projects from this course and make them easy to understand.
- The sections of a project README that answers a reader's questions, and how to show benchmark numbers honestly.
- How to write a technical write-up and prepare a short talk.
- How to check your READMEs automatically with a small Rust program and its tests.

## Before you start

- You finished [Lesson 24.1 — Roles for systems engineers](01-roles-for-systems-engineers.md) and created
  `code/p24-career`.
- You have at least one finished project from the course, for example a capstone from
  [Phase 23](../23-capstone-projects/README.md) or the project of your specialization track
  ([Phase 22](../22-specializations/README.md)).
- You know how to keep secrets out of Git ([Lesson 10.8 — Secrets management](../10-security-fundamentals/08-secrets-management.md)).

## New words

| Word | Meaning |
|---|---|
| **portfolio** | A small, public collection of your best work, with short explanations, that other people can look at. |
| **README** | The file `README.md` at the top of a repository. Code hosting sites show it on the project's front page. The name means "read me first". |
| **profile page** | Your personal page on GitHub or GitLab. Both sites can show a README from a special repository there. |
| **write-up** | An article that explains one thing you built or learned: the problem, the design, the numbers and the mistakes. |
| **lightning talk** | A very short talk, usually five minutes, about one idea. |
| **reproducible** | Another person can follow your steps and get the same result. |
| **licence** | The legal text that says what other people may do with your code. Without one, nobody may legally reuse it. |
| **MIT licence** | A short, permissive open-source licence, named after the Massachusetts Institute of Technology, where it was written. |
| **Apache-2.0** | The Apache License, version 2.0: a permissive licence from the Apache Software Foundation that also covers patents. |
| **status line** | One line at the top of a README that says how finished the project is, for example "Status: milestone 3 of 4". |
| **CI** (Continuous Integration) | A service that builds and tests every change automatically (Lesson 09.12). A *CI badge* is a small picture in the README that shows whether the last run passed. |
| **TCP** (Transmission Control Protocol) | The protocol that gives programs a reliable stream of bytes over the network (Lesson 15.3). It appears in the README template below. |
| **SRE** (Site Reliability Engineering) | Running production systems with software engineering methods; also the name of the role (Lesson 24.1). |
| **ASCII** (American Standard Code for Information Interchange) | The basic set of 128 text characters (English letters, digits, punctuation). An *ASCII diagram* is a picture drawn with these characters. |
| **exit code** | The number a program returns to the shell when it ends: 0 means success, other values mean failure (Lesson 11.5). |

## 1. What a portfolio is, and who reads it

A **portfolio** is public proof of your skills. Lesson 24.1 called such proof *evidence*. When you
apply for a job or offer a pull request to a project, the people on the other side cannot see your
learning journal or your private folders. They can see a repository, a README, an article or a video.

Who reads it, and how?

- **An engineer who reviews applications.** They often have a few minutes per person. They open your
  profile, click one project, read the first screen of the README and maybe one source file.
- **An interviewer.** They read more deeply before the interview and ask "why did you choose this?".
- **A maintainer of an open-source project.** They look at your earlier work to decide how carefully to
  review your first pull request.

This gives the main rule: **the first screen must explain the project.** What it is, why it exists, how to
run it, and whether it works. Everything else can come later in the file.

The second rule: **quality beats quantity.** Two finished, tested, well-explained projects say more than
twenty half-finished ones. A reader who opens a repository with no README and a failing build stops there.

## 2. Choose two or three projects

You built many projects in this course. Pick two or three that are **finished**, **tested**, **yours** and
**related to the role** you chose in Lesson 24.1.

| Candidate | Where it comes from | Good for these roles |
|---|---|---|
| Your capstone project | Phase 23 | All of them; this is your strongest piece |
| Your specialization project | Phase 22 (one of the track projects) | The role of that track |
| Mini container runtime | Lesson 13.17 | Infrastructure, platform, kernel |
| TCP load balancer | Lesson 15.15 | Networking, infrastructure, SRE |
| Your own key-value store | Lesson 17.14 | Database, systems software |
| Replicated key-value store | Lesson 18.14 | Database, distributed systems |
| Service on Kubernetes | Lesson 21.13 | Platform, infrastructure, SRE |

For each chosen project, make a short list of what is missing: a README section, a test, a benchmark, a
clean-up of warnings. Fix those things before you add anything new. A project you already have, made
excellent, is worth more than a new one.

> 📏 **Convention:** One project, one repository. Give it a clear name (`minikv`, not `p17-project-final2`)
> and a one-sentence description in the repository settings. Keep the course repository as your learning
> log; copy the finished project into its own repository when it is ready.

## 3. A README that answers the reader's questions

A reader comes with questions in a fixed order. A good README answers them in the same order:

1. *What is this?* — one paragraph.
2. *Is it finished?* — the status line.
3. *How does it work?* — a small diagram.
4. *How do I run it?* — exact commands.
5. *Does it work?* — how it is tested.
6. *Is it fast?* — numbers, with the machine and the commands.
7. *What does it not do?* — limitations.
8. *What did the author learn?* — two or three honest sentences.

Here is a template. Copy it into each portfolio project and fill it in:

````markdown
# minikv

A small key-value store with a write-ahead log and crash recovery, written in Rust.

Status: complete for the goals below; not used in production.

## What it is

Two or three sentences: the problem, who would use it, what is special about it.

## Architecture

```text
client --TCP--> server --> memtable --> SSTables on disk
                   \--> write-ahead log (fsync before reply)
```

## How to run

```bash
cargo run --release -- --data ./data --port 7000
```

## How it is tested

- Unit tests: `cargo test`
- A crash test that kills the server during writes and checks that no
  acknowledged write is lost: `cargo test --test crash -- --ignored`

## Benchmarks (M4 Mac, release build)

Machine: Apple M4, 24 GB memory, macOS, Rust 1.100.0, `--release`.
Command: `cargo bench --bench put_get`

| Operation | Median latency |
|---|---|
| put (with fsync) | (your number) |
| get (from memtable) | (your number) |

## Limitations

- One node only; no replication.
- Keys and values must fit in memory during compaction.

## What I learned

Two or three sentences about the hardest problem and how you solved it.

## License

MIT OR Apache-2.0
````

A few details matter more than they seem:

- **Commands must be exact.** A reader copies them. Test them yourself from a fresh clone.
- **The diagram can be ASCII.** It does not need to be pretty; it needs to show the parts and the data flow.
- **The limitations section is a strength, not a weakness.** It shows that you know where your design ends.
- **The licence.** Most Rust projects use two licences at once, `MIT OR Apache-2.0`: the user may choose
  either. Put the two licence texts in `LICENSE-MIT` and `LICENSE-APACHE` and write
  `license = "MIT OR Apache-2.0"` in `Cargo.toml`. You can copy the texts from choosealicense.com (see the
  sources).
- **Your capstone READMEs.** Lesson 23.9 gave capstones a README with other headings ("Quick start",
  "Guarantees and non-goals", "How it works", "Results", "Limits and next steps", "Licence"). Both structures
  answer the same questions. Choose one set of headings for all your public repositories; if you keep the
  Lesson 23.9 headings, change the list `REQUIRED` of the checker in Section 4 to match them (Exercise 1 shows
  how to change it).

> ⚡ **Performance:** A benchmark number without its context means nothing. Always write the machine, the
> Rust version, the build profile and the exact command, and prefer the median or percentiles to the
> average (Lesson 19.1). Never claim that your project is "faster than X" from one micro-benchmark;
> describe what you measured and its limits.

> 🔒 **Security:** Before you make a repository public, read its whole history with `git log -p` and look
> for passwords, tokens, private keys and internal host names. Deleting a file in a new commit does **not**
> remove it from history. If a real secret was ever committed, treat it as leaked: revoke it and create a
> new one (Lesson 10.8). Never publish code, configuration or data from an employer.

## 4. Check your READMEs with a program

You will have several repositories, and READMEs drift over time. A small checker finds missing sections
in a second. Create `src/bin/readme-check.rs` in `code/p24-career`:

```rust
// Checks that a project README has the sections a reader expects.
use std::process::ExitCode;

const REQUIRED: &[&str] = &[
    "what it is",
    "architecture",
    "how to run",
    "how it is tested",
    "benchmarks",
    "limitations",
];

/// Returns the level-2 headings (`## ...`) of a Markdown text, in lowercase.
/// Lines inside fenced code blocks are ignored.
fn headings(markdown: &str) -> Vec<String> {
    let mut in_code = false;
    let mut found = Vec::new();
    for line in markdown.lines() {
        if line.starts_with("```") {
            in_code = !in_code;
        } else if !in_code && let Some(title) = line.strip_prefix("## ") {
            found.push(title.trim().to_lowercase());
        }
    }
    found
}

/// Returns the required sections that the README does not have.
/// A heading counts if it starts with the required words, e.g. "Benchmarks (M4 Mac)".
fn missing_sections(markdown: &str) -> Vec<&'static str> {
    let found = headings(markdown);
    REQUIRED
        .iter()
        .copied()
        .filter(|required| !found.iter().any(|h| h.starts_with(required)))
        .collect()
}

fn main() -> ExitCode {
    let Some(path) = std::env::args().nth(1) else {
        eprintln!("usage: readme-check <path/to/README.md>");
        return ExitCode::from(2);
    };
    let text = match std::fs::read_to_string(&path) {
        Ok(text) => text,
        Err(err) => {
            eprintln!("cannot read {path}: {err}");
            return ExitCode::from(2);
        }
    };

    let missing = missing_sections(&text);
    if missing.is_empty() {
        println!("{path}: all {} sections present", REQUIRED.len());
        return ExitCode::SUCCESS;
    }
    println!(
        "{path}: missing {} of {} sections:",
        missing.len(),
        REQUIRED.len()
    );
    for section in missing {
        println!("  - ## {section}");
    }
    ExitCode::FAILURE
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn finds_level_two_headings_in_lowercase() {
        let md = "# Title\n## What it is\ntext\n### Detail\n## How To Run\n";
        assert_eq!(headings(md), vec!["what it is", "how to run"]);
    }

    #[test]
    fn ignores_headings_inside_code_blocks() {
        let md = "```text\n## not a heading\n```\n## Limitations\n";
        assert_eq!(headings(md), vec!["limitations"]);
    }

    #[test]
    fn heading_may_have_extra_words() {
        let md = "## Benchmarks (M4 Mac, release build)\n";
        assert!(!missing_sections(md).contains(&"benchmarks"));
    }

    #[test]
    fn complete_readme_has_nothing_missing() {
        let md: String = REQUIRED.iter().map(|s| format!("## {s}\n")).collect();
        assert!(missing_sections(&md).is_empty());
    }
}
```

**What happens:**

1. `REQUIRED` lists the section names in lowercase. Comparing lowercase text means `## How To Run` and
   `## How to run` both count.
2. `headings` walks over the lines. A line that starts with three backticks opens or closes a code block,
   so `in_code` flips. Outside code blocks, `strip_prefix("## ")` returns the rest of the line for a
   level-2 heading and `None` for anything else. The `&& let` part is a *let chain* (Lesson 04.5).
3. `missing_sections` keeps every required name that no heading starts with. `starts_with` allows extra
   words, so `## Benchmarks (M4 Mac, release build)` counts as "benchmarks".
4. `main` returns `ExitCode`. `let ... else` handles a missing argument; exit code 2 means "wrong usage",
   1 means "the README is incomplete", 0 means "all good". Scripts and CI can use these codes.
5. The tests check the parsing rules on tiny strings. The last one builds a README from `REQUIRED`
   itself, so it stays correct if you change the list.

Try it on a draft README. Save the template from Section 3 as `demo/README.md` in the package folder,
then delete three of its sections: "Architecture", "How it is tested" and "Limitations".

Run the checker, then print the exit code with `echo $?` (the shell keeps the last exit code in `$?`):

```bash
cargo run -q --bin readme-check -- demo/README.md; echo "exit code: $?"
```

```text
demo/README.md: missing 3 of 6 sections:
  - ## architecture
  - ## how it is tested
  - ## limitations
exit code: 1
```

`-q` hides Cargo's own "Compiling…" lines, and `--` separates Cargo's options from the program's
arguments. Without an argument the program prints its usage line and returns exit code 2; you will see
the "cannot read" case in the common mistakes below.

And the tests:

```bash
cargo test --bin readme-check
```

```text
running 4 tests
test tests::complete_readme_has_nothing_missing ... ok
test tests::finds_level_two_headings_in_lowercase ... ok
test tests::ignores_headings_inside_code_blocks ... ok
test tests::heading_may_have_extra_words ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Testing:** Each rule of the parser has its own test with the smallest input that shows it. When a
> real README confuses the checker, first copy the confusing lines into a new test, watch it fail, then
> fix the code (the regression-test habit from Lesson 05.6).

## 5. Write-ups: explain one thing well

A **write-up** shows that you understand what you built. It is also the best way to find out whether you
really do: gaps in understanding become visible when you try to explain them in writing.

A structure that works for systems topics:

| Part | Content | Size |
|---|---|---|
| Problem | What you wanted to build or find out, in plain words. | 1 paragraph |
| Background | Every term the reader needs, explained simply. | 2–4 paragraphs |
| Design | The parts, a diagram, and why you chose this design over another. | the main part |
| Measurements | A table with real numbers, the machine, the commands. | 1 table |
| Mistakes | What went wrong and what it taught you. | 1–3 paragraphs |
| Next | What you would do with more time. | a short list |

Good topics come from your journal: "What `fsync` really guarantees, tested by killing my key-value store",
"How my shell runs a pipeline: `fork`, `exec` and `pipe` step by step", "Three bugs my Raft simulation
found". Choose a topic where you have **numbers or a surprising result**.

Where to publish: a small static site you control is enough. **mdBook** (the tool that builds the Rust
Book) and **Zola** (a static site generator) are both written in Rust and can be hosted for free with
GitHub Pages or GitLab Pages. A Markdown file in the project repository also works for a start.

> 💡 **Tip:** Write for yourself one year ago. If that person could follow the article from start to end,
> the level is right.

## 6. Talks: five minutes, one idea

A **lightning talk** is the easiest way to start speaking: five minutes, one idea, three to six slides.
Local meetups and online meetups often welcome short talks from new speakers.

A simple shape: **the problem** (one slide), **what you built** (one or two slides with a diagram),
**one number or one surprise** (one slide), **what you learned** (one slide). Practise it out loud three
times with a timer. Record one practice run and watch it; you will notice what to cut.
Speaking in a language that is not your first language is normal at technical events.

## 7. The profile page

Your GitHub or GitLab profile is the entry point. Keep it short:

1. **Three sentences** about you: what you work on, what you are learning, what kind of role you look for.
2. **Links to your two or three projects**, each with one line of description.
3. **Links to merged pull requests** in open-source projects (Lessons 24.4 and 24.5) and to your write-ups.

Pin the portfolio repositories on the profile so they appear first. Remove or archive experiments that
would confuse a reader, or give them a status line that says "experiment".

## Common mistakes

**A README that starts with installation details.** The reader does not yet know what the project is. Start
with one paragraph that says what it is and why it exists.

**A red CI badge.** A badge that shows a failing build tells every visitor that the project is broken. Fix
the build or remove the badge.

**Running the checker from another folder.** A relative path such as `demo/README.md` is looked up from
the folder where you start the command, not from the package folder. From `src/` it fails:

```bash
cd src
cargo run -q --bin readme-check -- demo/README.md; echo "exit code: $?"
```

```text
cannot read demo/README.md: No such file or directory (os error 2)
exit code: 2
```

Go back with `cd ..`, or pass a full path such as `~/projects/rust/code/p24-career/demo/README.md`.

**Copying company work.** Even "cleaned" code or configuration from a job can break your contract and leak
internal details. Your portfolio contains only what you built in your own time with your own data.

## Try it yourself

1. Pick your two or three portfolio projects and write the list in `notes/career.md` with one missing thing
   for each.
2. Copy the README template into the first project and fill in every section with real content.
3. Run the checker on it: `cargo run -q --bin readme-check -- <path>/README.md`. Repeat until it prints
   "all 6 sections present".
4. Ask a friend who does not know the project to read only the first screen of the README and tell you what
   the project does. If they cannot, rewrite the first paragraph.
5. Plan one write-up in `notes/writeup-plan.md`: a title, one sentence for each of the six parts of
   Section 5, and the exact command that produces your main number.

## Exercises

### Exercise 1 — One more required section (easy)

Make the checker also require a `## License` section. What does it print for `demo/README.md` now? Why?

<details>
<summary>Hint</summary>

Add one line to `REQUIRED`, in lowercase. The test `complete_readme_has_nothing_missing` does not need a
change. Why?

</details>

<details>
<summary>Answer</summary>

Add `"license",` after `"limitations",` in `REQUIRED`. The output becomes:

```text
demo/README.md: missing 3 of 7 sections:
  - ## architecture
  - ## how it is tested
  - ## limitations
```

The template already has a `## License` section, so only the total changes from 6 to 7. Delete that
section from `demo/README.md` and `- ## license` appears as a fourth line.

Why it works: the program and the test both read the list from `REQUIRED`, so the test builds a README with
seven headings now and still passes.

</details>

### Exercise 2 — "How to run" needs a command (medium)

A "How to run" section without a command is not useful. Write
`section_has_code(markdown: &str, name: &str) -> bool`, which returns `true` when the section whose heading
starts with `name` contains a fenced code block. Write the test first. Then let `main` print a warning
line when the "how to run" section exists but has no code block.

<details>
<summary>Hint</summary>

Walk the lines and keep a flag `inside`. Every `## ` heading sets the flag: true if this heading starts
with `name`, false otherwise. While `inside` is true, a line starting with three backticks means "found".

</details>

<details>
<summary>Answer</summary>

```rust
/// Returns true if the section whose heading starts with `name` contains a fenced code block.
fn section_has_code(markdown: &str, name: &str) -> bool {
    let mut inside = false;
    for line in markdown.lines() {
        if let Some(title) = line.strip_prefix("## ") {
            inside = title.trim().to_lowercase().starts_with(name);
        } else if inside && line.starts_with("```") {
            return true;
        }
    }
    false
}

#[test]
fn finds_code_only_in_the_named_section() {
    let md = "## How to run\n```bash\ncargo run\n```\n## Limitations\nnone\n";
    assert!(section_has_code(md, "how to run"));
    assert!(!section_has_code(md, "limitations"));
    assert!(!section_has_code("## How to run\njust run it\n", "how to run"));
}
```

The test goes into `mod tests`. In `main`, right after `let missing = missing_sections(&text);`:

```rust
    if !missing.contains(&"how to run") && !section_has_code(&text, "how to run") {
        println!("{path}: the \"how to run\" section has no code block");
    }
```

For a file `demo/bad.md` with the text `## What it is`, `x`, `## How to run`, `Just run it.` (four lines):

```text
demo/bad.md: the "how to run" section has no code block
demo/bad.md: missing 4 of 6 sections:
  - ## architecture
  - ## how it is tested
  - ## benchmarks
  - ## limitations
```

Why it works: the flag changes only at `## ` headings, so a code block counts only for the section it is in.
The `!missing.contains(...)` check avoids a second message when the section is missing completely.

</details>

### Exercise 3 — Pin down a rule with a test (easy)

In Markdown, `##Architecture` (no space after the hashes) is **not** a heading; it is a normal line of text.
Write a test that documents how the checker treats such a line. Predict the result before you run it.

<details>
<summary>Hint</summary>

`strip_prefix("## ")` needs the space. So `headings` returns nothing, and every required section is
missing.

</details>

<details>
<summary>Answer</summary>

```rust
#[test]
fn a_heading_needs_a_space_after_the_hashes() {
    assert!(headings("##Architecture\n").is_empty());
    assert_eq!(missing_sections("##Architecture\n").len(), REQUIRED.len());
}
```

Why it works: the test states the rule in code. If somebody later changes the parser so that `##Architecture`
counts as a heading, this test fails and makes them decide on purpose. The README author sees the section
as missing, which is correct, because most Markdown viewers would not show it as a heading either.

</details>

## Quiz

1. What should the first screen of a README contain?

<details>
<summary>Answer</summary>

What the project is and why it exists, its status, and ideally how it works and how to run it. A reader
decides on the first screen whether to continue.

</details>

2. What must you write next to a benchmark number?

<details>
<summary>Answer</summary>

The machine (processor, memory, operating system), the Rust version, the build profile and the exact command, and which
statistic it is (for example the median).

</details>

3. You committed an access token a year ago and deleted it in the next commit. Is it safe now?

<details>
<summary>Answer</summary>

No. It is still in the Git history. Revoke the token and create a new one; then you may also clean the
history, but the revocation is what makes it safe.

</details>

4. What does `MIT OR Apache-2.0` in `Cargo.toml` mean?

<details>
<summary>Answer</summary>

The code is offered under two licences and each user may choose either of them. It is the most common
licence choice in the Rust ecosystem.

</details>

5. Which exit codes does `readme-check` use, and why do they matter?

<details>
<summary>Answer</summary>

0 when all sections are present, 1 when some are missing, 2 for wrong usage or an unreadable file.
Scripts and CI jobs decide success or failure by the exit code, so a checker that always returned 0
could not stop a pipeline.

</details>

## Summary

- A portfolio is public evidence: two or three finished, tested, explained projects, a few write-ups and
  maybe a talk.
- The first screen of a README decides whether someone reads on: what it is, status, diagram, how to run.
- Benchmarks always come with the machine, the Rust version, the build profile and the command.
- Limitations and "what I learned" sections show maturity.
- Keep secrets and employer material out of public repositories; check the history before publishing.
- A small checker with tests (`readme-check`) keeps your READMEs complete as projects change.

## Sources and further reading

- [GitHub Docs — Managing your profile README](https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-github-profile/customizing-your-profile/managing-your-profile-readme) —
  how the special profile repository works.
- [Make a README](https://www.makeareadme.com/) — a short guide to README sections.
- [Choose a License](https://choosealicense.com/) — the licence texts and what each licence allows.
- [The Cargo Book — The manifest format](https://doc.rust-lang.org/cargo/reference/manifest.html) — the
  `license` and `description` fields of `Cargo.toml`.
- [mdBook](https://rust-lang.github.io/mdBook/) and [Zola](https://www.getzola.org/) — two static site
  tools for write-ups; [GitHub Pages](https://docs.github.com/en/pages) and
  [GitLab Pages](https://docs.gitlab.com/user/project/pages/) host them for free.
- [std::process::ExitCode](https://doc.rust-lang.org/std/process/struct.ExitCode.html) — returning exit
  codes from `main`.

---

[🏠 Start](../START-HERE.md) · [← 24.1 Roles for systems engineers](01-roles-for-systems-engineers.md) · [📋 Phase 24](README.md) · [24.3 Reading large codebases →](03-reading-large-codebases.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
