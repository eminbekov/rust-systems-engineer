<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 24.4 Your first open-source contribution](04-your-first-open-source-contribution.md) · [📋 Phase 24](README.md) · [24.6 Resume and job search →](06-resume-and-job-search.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/24-career-and-open-source/05-working-with-maintainers-and-communities.md) <!-- nav -->

# Lesson 24.5 — Working with maintainers and communities

> **Phase 24 — Career and open source** · Lesson 5 of 9 · ⏱ about 120 minutes

## What you will learn

- How a project looks from the maintainer's side, and why that changes how you write to them.
- How to write a bug report with a minimal reproducible example, and collect your environment with a small
  Rust program.
- How to ask good questions, answer review comments, wait patiently and handle a closed pull request.
- Where the Rust community talks, how to behave there, and how to grow from a first-time contributor into a
  regular one without burning out.

## Before you start

- You finished [Lesson 24.4 — Your first open-source contribution](04-your-first-open-source-contribution.md).
- You know the author's and the reviewer's side of code review from
  [Lesson 08.13 — Refactoring and code review](../08-software-design/13-refactoring-and-code-review.md).
- You know `std::process::Command` from [Lesson 13.5](../13-operating-systems/05-fork-exec-and-wait.md).

## New words

| Word | Meaning |
|---|---|
| **community** | The people around a project: users, contributors, maintainers, and the places where they talk. |
| **bug report** | An issue that describes a problem so precisely that someone else can see it happen. |
| **MRE** (Minimal Reproducible Example) | The smallest complete program that still shows the problem. Also called a "minimal repro". |
| **environment** | The facts about where the code ran: versions of Rust and the library, operating system, processor type. |
| **review comment** | A remark that a reviewer attaches to a line or to the whole pull request. |
| **nit** | A small, optional review remark ("nitpick"), for example about a name. Reviewers often start it with `nit:`. |
| **ping** | A short, polite reminder in a discussion that has been quiet for a while. |
| **triage** | Sorting new issues: checking that they can be reproduced, adding labels, asking for missing facts. |
| **burnout** | Exhaustion from too much work or pressure for too long. It is common among maintainers. |
| **ID** (identifier) | A number or name that identifies one thing, for example a thread ID in a panic message. |
| **AI** (Artificial Intelligence) | Tools that generate text or code (Lesson 24.3). |
| **OS** (Operating System) | The program that manages the hardware and runs other programs; `bug-report` prints its name, for example `macos`. |

## 1. The maintainer's side

Imagine you maintain a library that a few thousand projects use. You work on it in the evenings. Every week
brings thirty new notifications: bug reports, questions, pull requests, and people asking "any news?".

From this side, three facts follow:

1. **Time is the scarce resource.** A report that can be understood in two minutes gets attention. One that
   needs five questions back and forth waits.
2. **A merged change is a promise.** The maintainer will keep that code working for years, answer questions
   about it and fix its bugs. That is why they are careful about *what* they accept, not only *how* it is written.
3. **Most maintainers are kind but tired.** A short answer is usually not rudeness; it is the tenth answer that
   evening.

So everything in this lesson has one goal: **make the maintainer's next step easy.** A precise report, a small
pull request, a clear reply to a review comment.

## 2. A bug report that gets fixed

A good bug report answers four questions. Many projects have an issue template with exactly these parts:

| Part | Question | Example |
|---|---|---|
| What I did | Which steps, which code, which input? | "I called `first_word("hello")`." |
| What I expected | What should have happened, and why do you think so? | "The whole text, `"hello"`, as for any single word." |
| What happened | The exact output, error or panic, copied, not retold. | the panic message |
| Environment | Versions and platform. | Rust version, crate version, operating system |

The most valuable part is the **MRE**, the minimal reproducible example. Suppose a large program of yours
crashes, and you suspect a helper function from a library. You remove everything that is not needed for the
crash, step by step, and check after each step that the crash still happens. You end with this:

```rust
fn first_word(text: &str) -> &str {
    let end = text.find(' ').unwrap();
    &text[..end]
}

fn main() {
    println!("{}", first_word("hello world"));
    println!("{}", first_word("hello"));
}
```

```text
hello

thread 'main' (6656315) panicked at src/main.rs:2:30:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

**What happens:**

1. The first call works and prints `hello`: there is a space, so `find` returns `Some(5)`.
2. In the second call there is no space; `find` returns `None`, and `unwrap()` panics (Lesson 04.10).
3. The program is nine lines, has no dependencies, and shows the expected result (`hello`) next to the
   failure. A maintainer can paste it into the Rust Playground or a new project and see the bug in seconds.
   (The number after `'main'` is a thread ID, an identifier of the thread; it changes on every run.)

Shrinking a program like this often shows you the bug yourself. That is fine: then you can say in the report
"the cause seems to be the `unwrap` in line 2", or even send the fix.

> 💡 **Tip:** Search the issue tracker for the error message before you write a new issue. If an issue already
> exists, add your MRE and environment there instead of a "+1" comment; the extra facts help, a "+1" does not.

## 3. Collect the environment with a program

The environment part is easy to forget and boring to type. Add a small helper to `code/p24-career`,
`src/bin/bug-report.rs`:

```rust
// Prints the facts that every bug report needs, ready to paste into an issue.
use std::process::Command;

/// Runs `program --version` and returns the first line it prints,
/// or a short note if the program cannot be run.
fn version_of(program: &str) -> String {
    match Command::new(program).arg("--version").output() {
        Ok(out) if out.status.success() => String::from_utf8_lossy(&out.stdout)
            .lines()
            .next()
            .unwrap_or("")
            .to_string(),
        _ => format!("{program}: not found"),
    }
}

/// Formats the environment part of a bug report as Markdown.
fn environment(rustc: &str, cargo: &str, os: &str, arch: &str) -> String {
    format!("### Environment\n\n- {rustc}\n- {cargo}\n- OS: {os} ({arch})\n")
}

fn main() {
    let report = environment(
        &version_of("rustc"),
        &version_of("cargo"),
        std::env::consts::OS,
        std::env::consts::ARCH,
    );
    print!("{report}");
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn environment_is_a_markdown_list() {
        let text = environment("rustc 1.0.0", "cargo 1.0.0", "linux", "x86_64");
        assert_eq!(
            text,
            "### Environment\n\n- rustc 1.0.0\n- cargo 1.0.0\n- OS: linux (x86_64)\n"
        );
    }

    #[test]
    fn a_missing_program_is_reported_not_panicked() {
        assert_eq!(
            version_of("no-such-program-24-5"),
            "no-such-program-24-5: not found"
        );
    }

    #[test]
    fn rustc_reports_its_version() {
        assert!(version_of("rustc").starts_with("rustc "));
    }
}
```

**What happens:**

1. `Command::new(program).arg("--version").output()` starts the program, waits for it and collects what it
   printed (Lesson 13.5). It returns `Err` if the program does not exist.
2. The `match` has a **guard**, `if out.status.success()`: the first arm is used only when the program ran *and*
   exited with code 0. Every other case falls into `_` and gives a note instead of a panic.
3. `String::from_utf8_lossy` turns the output bytes into text, replacing any invalid bytes instead of failing;
   `.lines().next()` keeps the first line.
4. `std::env::consts::OS` and `ARCH` are constants that Rust fills in at compile time with the target operating
   system and processor type.
5. `environment` only formats text, so it is easy to test exactly. The second test checks the failure path; the
   third checks the real `rustc` on your machine.

```bash
cargo run -q --bin bug-report
cargo test --bin bug-report
```

```text
### Environment

- rustc 1.100.0 (0ef1bc386 2026-11-09)
- cargo 1.100.0 (3acc79c0c 2026-11-05)
- OS: macos (aarch64)
```

```text
running 3 tests
test tests::environment_is_a_markdown_list ... ok
test tests::a_missing_program_is_reported_not_panicked ... ok
test tests::rustc_reports_its_version ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.04s
```

`aarch64` is Rust's name for the arm64 processors of Apple M-series Macs (Lesson 12.6). For a bug in a library,
add the library's version too: `cargo tree -p <crate> --depth 0` prints it.

> 🔒 **Security:** Before you paste logs, configuration or a backtrace into a public issue, remove passwords,
> tokens, internal host names and personal data. If you found a **security** problem (for example a crash that an
> attacker could trigger), do not open a public issue: look for `SECURITY.md` or a "security policy" in the
> repository and report it privately, as described there. After the fix, such problems are published as security
> advisories (Lesson 10.13).

## 4. Asking questions

When you are stuck in a contribution, ask; but make the question easy to answer:

1. **Search first**: the documentation, the issue tracker, the chat history.
2. **Say what you want to achieve**, not only the step that fails. Sometimes there is a better way.
3. **Show what you tried** and the exact error, in a code block.
4. **Ask in the right place.** Questions about *using* Rust belong in the Rust Users Forum. Questions about *one
   project* belong in its chat or discussion forum, or in the issue you work on. Do not send private messages to
   maintainers unless they invite you.

A good question, in an issue you are working on:

```text
I am working on this issue. I added the check in `parse_pair` (commit 12cc175),
but `cargo test --doc` now fails in `README.md` line 14, because the example
there uses an empty key on purpose. Should I change the example, or should
empty keys stay allowed in that case?
```

It says what was done, what fails, where, and offers two concrete options. The maintainer can answer in one line.

## 5. Code review: answering comments

Your pull request will get comments. That is normal, and it is the most valuable part of contributing: an
experienced engineer is reading your code for free.

- **Answer every comment.** Either change the code and reply "Done in <commit>", or explain why you think
  otherwise. Silence looks like you did not see it.
- **Ask when a comment is unclear.** "Do you mean the check should move into `split_once`'s caller, or the
  whole function should return a `Result`?" is a good reply.
- **Disagree with reasons, once.** Give your argument briefly. If the maintainer still prefers their way, follow
  it: they will maintain the code, not you.
- **Separate nits from real problems.** A comment starting with `nit:` is optional; follow it if it is cheap.
- **Follow the project's style for updates.** Some projects want new commits during review and squash at the end;
  others want you to rewrite your commits with an interactive rebase (Lesson 01.9). `CONTRIBUTING.md` or the
  reviewer will say.
- **Do not take it personally.** A comment is about the code. "This allocates on every call" is not "you are a bad
  engineer".

Short replies that work well:

```text
Good catch, done in 4f2a9c1.
I kept the `unwrap` here because `split_once` already checked for `=`; is a comment enough?
I am not sure I understand. Do you mean X or Y?
Thanks for the review! I pushed the changes; ready for another look.
```

> 📏 **Convention:** Be specific and kind, in both directions. Write "This loop is quadratic for long input; a
> `HashSet` would make it linear" instead of "This is slow". The same rule applies when you review others.

## 6. Waiting, pinging and closed pull requests

**Waiting.** Reviews in volunteer projects can take days or weeks. If nothing happens for about two weeks, a
single polite ping is fine: "Friendly ping: is there anything I should change here?" Do not ping daily, and do not
tag many people.

**When there is no answer at all.** Sometimes a project is quiet for months. It is fine to move on to another
project. If you claimed an issue and stop working on it, say so in a short comment, so that someone else can take
it.

**When your pull request is closed.** It happens to everyone. Common reasons: the maintainers want a different
design, the problem was solved in another way, or the change does not fit the project's goals. Read the reason,
thank them for their time, and keep what you learned. A polite reaction to a closed pull request makes your next
one in the same project more likely to succeed.

## 7. Communities and how to behave there

The official Rust places to talk, listed on the community page of rust-lang.org:

- the **Rust Users Forum** (`users.rust-lang.org`) for questions about using Rust;
- the **Rust Zulip** chat, where the teams that develop Rust itself work and discuss;
- project chats (many projects run a Zulip, Discord or Matrix space; the README links to it);
- **This Week in Rust**, the weekly newsletter, and local or online **meetups**.

Every one of them has a code of conduct. The basic rules are simple: be kind, assume good intent, keep
discussions technical, and do not insult anyone, also when you are frustrated.

Writing in a language that is not your first language is normal in these spaces. Short, clear sentences are easy
to read for everybody. If you use an AI tool to translate or polish your text, follow the project's AI policy
(Lesson 24.3): some projects ask you to write in your own words and mark any machine translation as such.

## 8. From first-timer to regular

After the first merged change, the best next step is usually **the same project again**. You already know how to
build and test it, and the maintainers know you. Typical growth:

1. a second and third small fix;
2. **triage**: reproducing new bug reports, asking reporters for missing facts, writing MREs for them;
3. **reviewing** other people's pull requests: "I tried this branch and it fixes the issue for me" helps the
   maintainers a lot;
4. a larger change, discussed in an issue first.

Keep it sustainable. Contributing is voluntary; pick a pace you can keep for a year, for example two hours a week.
It is fine to say "I do not have time for this now". Maintainers burn out, contributors do too.

## Common mistakes

**Retelling an error instead of copying it.** "It says something about unwrap" cannot be searched or checked.
Copy the exact text into a code block.

**An MRE that does not compile.** If the maintainer must guess the missing `use` lines or types, the example
is not reproducible. Run it yourself in a new project before you post it.

**Arguing every review comment.** Explain once; then accept the maintainer's decision or close the pull request
politely.

**Pinging too often, or in too many places.** One ping after about two weeks, in the pull request itself.

**Posting secrets.** Logs and configuration often contain tokens or internal names. Read every line you paste.

## Try it yourself

1. Add `bug-report` to `code/p24-career`, run it and its tests.
2. Take a panic or a wrong result from one of your own course projects, shrink it to an MRE of at most fifteen
   lines, and write a complete bug report (the four parts of Section 2) in `notes/oss-log.md`.
3. Read ten recent issues in a project you use. Mark which ones are easy to act on and which ones are not, and
   write down why.
4. Read the code of conduct of that project.

## Exercises

### Exercise 1 — Template for the whole report (medium)

Extend `bug-report` so that it prints empty sections "What I did", "What I expected" and "What happened instead"
(each followed by `(fill in)`) before the environment. Write a test that checks that the four headings come in
this order.

<details>
<summary>Hint</summary>

Write a function `report_template(environment: &str) -> String` that builds the text in a loop over the three
headings. In the test, use `text.find("### …")` for each heading and compare the positions.

</details>

<details>
<summary>Answer</summary>

```rust
/// The empty sections that the reporter fills in by hand, followed by the environment.
fn report_template(environment: &str) -> String {
    let mut text = String::new();
    for heading in ["What I did", "What I expected", "What happened instead"] {
        text.push_str(&format!("### {heading}\n\n(fill in)\n\n"));
    }
    text.push_str(environment);
    text
}

fn main() {
    let env = environment(
        &version_of("rustc"),
        &version_of("cargo"),
        std::env::consts::OS,
        std::env::consts::ARCH,
    );
    print!("{}", report_template(&env));
}

#[test]
fn template_has_the_sections_in_order() {
    let text = report_template("### Environment\n");
    let did = text.find("### What I did").unwrap();
    let expected = text.find("### What I expected").unwrap();
    let happened = text.find("### What happened instead").unwrap();
    let env = text.find("### Environment").unwrap();
    assert!(did < expected && expected < happened && happened < env);
}
```

The test goes into `mod tests`. Output of `cargo run -q --bin bug-report`:

```text
### What I did

(fill in)

### What I expected

(fill in)

### What happened instead

(fill in)

### Environment

- rustc 1.100.0 (0ef1bc386 2026-11-09)
- cargo 1.100.0 (3acc79c0c 2026-11-05)
- OS: macos (aarch64)
```

Why it works: `report_template` gets the environment as a parameter instead of calling `rustc` itself, so the
test can pass a fixed text and does not depend on the machine.

</details>

### Exercise 2 — Shrink it (medium)

This program should print a summary with one warning for the line `verbose` (a setting without a value), but it
panics. Shrink it to an MRE of at most ten lines that shows the same panic, and say in one sentence what the cause
is.

```rust
use std::collections::HashMap;

struct Config {
    values: HashMap<String, String>,
    warnings: Vec<String>,
}

fn command_name(line: &str) -> &str {
    let end = line.find(' ').unwrap();
    &line[..end]
}

fn load(text: &str) -> Config {
    let mut config = Config {
        values: HashMap::new(),
        warnings: Vec::new(),
    };
    for (number, line) in text.lines().enumerate() {
        let line = line.trim();
        if line.is_empty() || line.starts_with('#') {
            continue;
        }
        let name = command_name(line);
        let rest = line[name.len()..].trim();
        if rest.is_empty() {
            config
                .warnings
                .push(format!("line {}: {name} has no value", number + 1));
        }
        config.values.insert(name.to_string(), rest.to_string());
    }
    config
}

fn main() {
    let text = "# server settings\nport 8080\nhost localhost\nverbose\n";
    let config = load(text);
    println!(
        "{} values, {} warnings",
        config.values.len(),
        config.warnings.len()
    );
}
```

```text

thread 'main' (6656547) panicked at src/main.rs:9:30:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

<details>
<summary>Hint</summary>

The panic is in line 9, inside `command_name`. Which input line reaches it without a space? Keep only that
function and one call that works next to one that fails.

</details>

<details>
<summary>Answer</summary>

```rust
fn command_name(line: &str) -> &str {
    let end = line.find(' ').unwrap();
    &line[..end]
}

fn main() {
    println!("{}", command_name("port 8080"));
    println!("{}", command_name("verbose"));
}
```

```text
port

thread 'main' (6656974) panicked at src/main.rs:2:30:
called `Option::unwrap()` on a `None` value
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

Cause: `command_name` assumes every line has a space after the name, so the line `verbose` panics before the
warning code is ever reached. Why it works as an MRE: the `HashMap`, the `Config` struct, comments and line numbers
had nothing to do with the panic; removing them leaves only the faulty assumption, with one passing call to show
what the function should do.

</details>

### Exercise 3 — Answer a review (easy, no code)

A maintainer left three comments on your pull request: (a) "nit: rename k to key". (b) "This adds a new public
function. Could it be private instead?" (c) "I think this belongs in a separate pull request." Write a reply to each.

<details>
<summary>Hint</summary>

For each comment decide: do it and confirm, ask a question, or explain once. Keep every reply to one or two
sentences.

</details>

<details>
<summary>Answer</summary>

Example replies:

```text
(a) Done in 7c1e0b2.
(b) Good point. Nothing outside the module uses it, so I made it private in 7c1e0b2.
(c) Agreed. I removed that part from this pull request and opened a separate issue for it.
```

Each reply confirms an action with a commit, or shows that you understood the reason. None of them argues about a
nit, and the scope comment is handled by making the pull request smaller, which is what the maintainer asked for.

</details>

## Quiz

1. Why is a minimal reproducible example the most valuable part of a bug report?

<details>
<summary>Answer</summary>

The maintainer can run it in seconds and see the problem without guessing your setup. Shrinking it often also
shows where the bug is.

</details>

2. Where do you report a security problem that you found in a library?

<details>
<summary>Answer</summary>

Privately, following the project's security policy (often in `SECURITY.md`), not in a public issue.

</details>

3. A reviewer asks for a change you disagree with. What do you do?

<details>
<summary>Answer</summary>

Explain your reasons once, briefly. If they still prefer their way, follow it, because they maintain the code;
or close the pull request politely if you cannot accept the change.

</details>

4. Your pull request has had no reaction for two weeks. What is a good next step?

<details>
<summary>Answer</summary>

One short, polite ping in the pull request ("Friendly ping: is there anything I should change?"). Not daily, not
in many places.

</details>

5. In `version_of`, why is there a guard `if out.status.success()` in the `match`?

<details>
<summary>Answer</summary>

`output()` returns `Ok` even when the program ran but failed. The guard makes the first arm match only a
successful run, so a failing program also gives the "not found" note instead of printing its error output as a
version.

</details>

## Summary

- Maintainers have little time and must maintain what they merge: make their next step easy.
- A good bug report has four parts (what I did, what I expected, what happened, environment) and an MRE that
  compiles and shows the problem in a few lines.
- Collect the environment automatically, and remove secrets before you paste anything.
- Report security problems privately, following the project's security policy.
- Answer every review comment, disagree once with reasons, and do not take comments personally.
- Wait patiently, ping once, accept a closed pull request gracefully, and grow in one project at a sustainable pace.

## Sources and further reading

- [Stack Overflow — How to create a Minimal, Reproducible Example](https://stackoverflow.com/help/minimal-reproducible-example) —
  the classic description of an MRE.
- [Rust Playground](https://play.rust-lang.org/) — run and share small Rust programs, ideal for MREs.
- [Rust — Community](https://www.rust-lang.org/community), [Rust Users Forum](https://users.rust-lang.org/) and
  [Rust Zulip](https://rust-lang.zulipchat.com/) — where the Rust community talks.
- [Google Engineering Practices — The author's guide to code review](https://google.github.io/eng-practices/review/developer/) —
  including how to handle reviewer comments.
- [Open Source Guides — Maintaining Balance for Open Source Maintainers](https://opensource.guide/maintaining-balance-for-open-source-maintainers/) —
  the maintainer's side, and how to avoid burnout.
- [std::process::Command](https://doc.rust-lang.org/std/process/struct.Command.html) and
  [std::env::consts](https://doc.rust-lang.org/std/env/consts/index.html) — the standard library items used by `bug-report`.

---

[🏠 Start](../START-HERE.md) · [← 24.4 Your first open-source contribution](04-your-first-open-source-contribution.md) · [📋 Phase 24](README.md) · [24.6 Resume and job search →](06-resume-and-job-search.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
