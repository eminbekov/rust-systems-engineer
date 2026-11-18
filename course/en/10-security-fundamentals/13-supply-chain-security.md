<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.12 Memory safety and reviewing unsafe code](12-memory-safety-and-reviewing-unsafe.md) · [📋 Phase 10](README.md) · [10.14 Project: an encrypted secrets vault →](14-project-encrypted-secrets-vault.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/13-supply-chain-security.md) <!-- nav -->

# Lesson 10.13 — Supply-chain security: cargo-audit and cargo-deny

> **Phase 10 — Security fundamentals** · Lesson 13 of 15 · ⏱ about 120 minutes

## What you will learn

- What a software supply chain is and which attacks go through your dependencies.
- How to see what your project really depends on (`cargo tree`) and why `Cargo.lock` matters.
- How to find known vulnerabilities and unmaintained crates with `cargo-audit`, and how to fix them.
- How to enforce rules on licenses, banned crates and sources with `cargo-deny`.
- Habits for adding a new dependency, and how to run both tools automatically.

## Before you start

- You finished [Lesson 10.12 — Memory safety and reviewing unsafe code](12-memory-safety-and-reviewing-unsafe.md).
- You know dependencies, version requirements and `Cargo.lock` ([Lesson 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md)).
- Install the two tools. They are normal Rust programs; `--locked` makes Cargo use the exact dependency versions the authors tested. This compiles a lot of code and takes a few minutes:

```bash
cargo install --locked cargo-audit cargo-deny
cargo audit --version
cargo deny --version
```

```text
cargo-audit-audit 0.23.0
cargo-deny 0.21.0
```

(The doubled word in `cargo-audit-audit` is just how this version of the tool prints its own name; the number is what matters.)

- Create a scratch package, which you will delete at the end. It deliberately uses an **old** version of a crate that has a published vulnerability:

```bash
cd ~/projects/rust
cargo new code/p10-audit-demo
cd code/p10-audit-demo
cargo add regex@1.5 atty@0.2
cargo update -p regex --precise 1.5.4
```

```text
    Updating crates.io index
 Downgrading aho-corasick v1.1.5 -> v0.7.20
 Downgrading regex v1.13.2 -> v1.5.4
    Removing regex-automata v0.4.19
 Downgrading regex-syntax v0.8.12 -> v0.6.29
```

Replace `src/main.rs` with a tiny program that uses both crates:

```rust
use regex::Regex;

fn main() {
    let re = Regex::new(r"^\d{4}-\d{2}-\d{2}$").unwrap();
    println!("date? {}", re.is_match("2026-11-18"));
    println!("terminal? {}", atty::is(atty::Stream::Stdout));
}
```

```text
date? true
terminal? false
```

(`terminal? false` appears because the output of `cargo run` is not connected to a terminal in this listing; on your screen it may say `true`.)

> 🔒 **Ethics:** this old `regex` is a vulnerability only when a program compiles regular expressions sent by an untrusted user. The demo program is never exposed to anyone. Never deploy a project that uses a known-vulnerable version.

## New words

| Word | Meaning |
|---|---|
| **supply chain** | Everything that goes into your program that you did not write yourself: dependencies, build tools, the registry, the compiler. |
| **dependency tree** | All crates your project needs, including the dependencies of your dependencies (*transitive* dependencies). |
| **advisory** | A published security notice about a crate version: what is wrong, which versions are affected, and which version fixes it. |
| **RustSec** | The Rust Security Advisory Database, with advisories for crates published on crates.io. Its entries have IDs such as `RUSTSEC-2022-0013`. |
| **CVE** (Common Vulnerabilities and Exposures) | A worldwide numbering system for vulnerabilities, for example `CVE-2022-24713`. |
| **unmaintained** | A crate whose authors say they stopped working on it. It may keep working but will not get security fixes. |
| **unsound** | A crate that has a safe function which can cause undefined behaviour (Lesson 10.12). |
| **yanked** | A version that its author withdrew from the registry because it is broken or dangerous. Projects that already locked it still work, but new projects will not pick it. |
| **license** | The legal terms for using code. Some licenses (for example GPL, General Public License) place conditions on your own code. |
| **SPDX** (Software Package Data Exchange) | A standard list of short license names such as `MIT` and `Apache-2.0`. |
| **typosquatting** | Publishing a malicious crate with a name close to a popular one (`serde_jsonn`) and waiting for typing mistakes. |
| **`build.rs` / procedural macro** | Code that runs on your computer **while you compile**. A dependency can use it. |

## 1. What is a supply chain, and how does it get attacked?

A typical Rust program has your 500 lines and the code of tens or hundreds of crates written by other people. You trust all of it with your users' data. The attacks go through that trust:

| Attack | Example | Defence |
|---|---|---|
| **Known vulnerability** in a dependency | An old `regex` version that can be made to spend a long time on a crafted pattern | `cargo audit`, `cargo deny check advisories`, regular updates |
| **Unmaintained or unsound** crate | `atty`: it works, but nobody fixes problems | The same tools, then replace the crate |
| **Malicious crate** | Typosquatting; a stolen maintainer account that publishes a bad update | Careful review of new dependencies, `Cargo.lock`, a short dependency list |
| **Code that runs at build time** | `build.rs` or a procedural macro reads your files or secrets | Review, build in a clean CI (Continuous Integration: a server that runs checks on every push) environment without secrets |
| **License problem** | A dependency with terms your company cannot accept | `cargo deny check licenses` |
| **Unexpected source** | A dependency taken from a random Git URL (Uniform Resource Locator, a web address) | `cargo deny check sources` |

You cannot remove the risk completely, but you can make it small and **visible**.

## 2. Know what you depend on

```bash
cargo tree
```

```text
p10-audit-demo v0.1.0 (/Users/ada/projects/rust/code/p10-audit-demo)
├── atty v0.2.14
│   └── libc v0.2.191
└── regex v1.5.4
    ├── aho-corasick v0.7.20
    │   └── memchr v2.8.3
    ├── memchr v2.8.3
    └── regex-syntax v0.6.29
```

Two direct dependencies became **six crates** in the tree. (`Cargo.lock` lists 11 packages in total: these six, the demo itself, and four more that other systems need: `hermit-abi` and three `winapi` crates, which `atty` uses on Windows and other platforms.) Two more `cargo tree` commands answer common questions:

```bash
cargo tree -i libc    # inverse: who depends on libc?
cargo tree -d         # duplicates: the same crate in two versions
```

```text
libc v0.2.191
└── atty v0.2.14
    └── p10-audit-demo v0.1.0 (/Users/ada/projects/rust/code/p10-audit-demo)
```

`cargo tree -d` printed `warning: nothing to print.`: there are no duplicates here. Duplicates grow the build and the attack surface.

`Cargo.lock` records the exact version (and checksum) of every crate in the tree. For **programs** you commit it to Git, so everybody builds the same code. In CI, `cargo build --locked` fails if the lock file would change, so a dependency cannot change silently. The registry also stores a hash for every download, and Cargo checks it against the lock file (Lesson 10.9: a hash detects accidental or hostile changes of the file).

## 3. cargo-audit: known vulnerabilities

`cargo-audit` reads `Cargo.lock` and looks every `name + version` up in the **RustSec advisory database**.

```bash
cargo audit
```

The first run downloads the database (the first lines tell you from where and how many advisories it loaded; both change over time). The rest looks like this:

```text
    Scanning Cargo.lock for vulnerabilities (11 crate dependencies)
Crate:     regex
Version:   1.5.4
Title:     Regexes with large repetitions on empty sub-expressions take a very long time to parse
Date:      2022-03-08
ID:        RUSTSEC-2022-0013
URL:       https://rustsec.org/advisories/RUSTSEC-2022-0013
Severity:  7.5 (high)
Solution:  Upgrade to >=1.5.5 (patched)

Crate:     atty
Version:   0.2.14
Warning:   unmaintained
Title:     `atty` is unmaintained
Date:      2024-09-25
ID:        RUSTSEC-2024-0375
URL:       https://rustsec.org/advisories/RUSTSEC-2024-0375
Solution:  No fixed upgrade is available!

Crate:     atty
Version:   0.2.14
Warning:   unsound
Title:     Potential unaligned read
Date:      2021-07-04
ID:        RUSTSEC-2021-0145
URL:       https://rustsec.org/advisories/RUSTSEC-2021-0145
Solution:  No fixed upgrade is available!

error: 1 vulnerability found!
warning: 2 allowed warnings found
```

**What happens:**

1. The tool found three problems. The first is a real **vulnerability** (an attacker who can send a regular expression to your program can make it spend a very long time compiling it; a denial of service, Lesson 10.12). The solution field says which version fixes it.
2. The other two are *warnings* about `atty`: unmaintained, and unsound (it can read memory incorrectly on Windows). There is nothing to upgrade to, so the fix is to stop using the crate.
3. The command ends with a non-zero exit code (`1`) because it found a vulnerability. That is how CI notices. Warnings alone do not fail it unless you ask: `cargo audit --deny warnings`.

**Fix 1: update** (the vulnerability had a patched version). `cargo update -p` updates one package within the limits of `Cargo.toml` (`"1.5"` allows every `1.x` from `1.5`):

```bash
cargo update -p regex
```

```text
    Updating crates.io index
     Locking 4 packages to highest Rust 1.100.0 compatible versions
    Updating aho-corasick v0.7.20 -> v1.1.5
    Updating regex v1.5.4 -> v1.13.2
      Adding regex-automata v0.4.19
    Updating regex-syntax v0.6.29 -> v0.8.12
```

**Fix 2: replace** the crate with no fix. The standard library has had `IsTerminal` since Rust 1.70, so `atty` is not needed. Change `main.rs`, then remove the dependency:

```rust
use regex::Regex;
use std::io::IsTerminal;

fn main() {
    let re = Regex::new(r"^\d{4}-\d{2}-\d{2}$").unwrap();
    println!("date? {}", re.is_match("2026-11-18"));
    println!("terminal? {}", std::io::stdout().is_terminal());
}
```

```bash
cargo remove atty
cargo audit
```

```text
    Scanning Cargo.lock for vulnerabilities (6 crate dependencies)
```

No findings, and the exit code is `0`. The dependency tree shrank from 11 packages to 6.

> 📏 **Convention:** run `cargo audit` in CI on every push **and on a schedule**. New advisories are published every week for code that has not changed. When a finding cannot be fixed at once, you can ignore it with `cargo audit --ignore RUSTSEC-xxxx-xxxx` or in `.cargo/audit.toml`, but write down why, and set a date to look again.

## 4. cargo-deny: policies for the whole tree

`cargo-audit` answers one question. `cargo-deny` checks four kinds of rules, so a team can decide once and let a tool enforce it:

| Check | What it enforces |
|---|---|
| `advisories` | The RustSec check again (vulnerabilities, unmaintained, unsound, **yanked** versions). |
| `licenses` | Only the license names you allow. |
| `bans` | Crates you forbid, and whether the tree may contain the same crate in several versions. |
| `sources` | Only the registries and Git repositories you list. |

Create the starting configuration and look at it:

```bash
cargo deny init
```

It writes `deny.toml`, a long file where every option is explained in comments. Replace it with this short, working version:

```toml
[graph]
all-features = false

[advisories]
yanked = "deny"

[licenses]
allow = ["MIT", "Apache-2.0"]

[licenses.private]
ignore = true # our own, unpublished crate needs no license

[bans]
multiple-versions = "warn"
wildcards = "deny"

[sources]
unknown-registry = "deny"
unknown-git = "deny"
allow-registry = ["https://github.com/rust-lang/crates.io-index"]
```

Also add `publish = false` under `[package]` in `Cargo.toml`, so `cargo-deny` knows the demo crate is private.

**What happens:**

1. `yanked = "deny"` makes a yanked version an error. Vulnerabilities are errors by default.
2. `licenses.allow` is an **allow-list**: a crate is accepted if its license expression can be satisfied with allowed names. `"MIT OR Apache-2.0"` (very common) is accepted, because it needs one of them.
3. `multiple-versions = "warn"` reports the same crate twice in the tree; `wildcards = "deny"` forbids dependencies written as `version = "*"` (any version), which turn every update into a surprise.
4. `sources` allows only the official crates.io index; a dependency from any Git URL is an error.
5. The `bans` table can also forbid a crate by name, with a reason. You will add that rule in a moment.

To see the checks fail, reproduce the old state: put `atty` back (`cargo add atty@0.2`, and restore the `atty::is` line in `main.rs`) and downgrade `regex` (`cargo update -p regex --precise 1.5.4`). Then run:

```bash
cargo deny check
```

The full output is long: every finding comes with the advisory text. This is the shortened version, one message per problem:

```text
error[unsound]: Potential unaligned read
  ├ ID: RUSTSEC-2021-0145
  ├ Solution: No safe upgrade is available!
  ├ atty v0.2.14
    └── p10-audit-demo v0.1.0

error[vulnerability]: Regexes with large repetitions on empty sub-expressions take a very long time to parse
  ├ ID: RUSTSEC-2022-0013
  ├ Solution: Upgrade to >=1.5.5 (try `cargo update -p regex`)
  ├ regex v1.5.4
    └── p10-audit-demo v0.1.0

error[unmaintained]: `atty` is unmaintained
  ├ ID: RUSTSEC-2024-0375
  ├ Solution: No safe upgrade is available!
  ├ atty v0.2.14
    └── p10-audit-demo v0.1.0

advisories FAILED, bans ok, licenses ok, sources ok
```

The last line is the summary of the four checks. Now make the other checks fail on purpose. Change `allow` to `["Apache-2.0"]` and run `cargo deny check licenses`:

```text
error[rejected]: failed to satisfy license requirements
   ┌─ ~/.cargo/registry/src/index.crates.io-…/atty-0.2.14/Cargo.toml:23:12
   │
23 │ license = "MIT"
   │            ━━━
   │            │
   │            rejected: license is not explicitly allowed
   │
   ├ MIT - MIT License:
   ├   - OSI approved
   ├   - FSF Free/Libre
   ├ atty v0.2.14
     └── p10-audit-demo v0.1.0
```

(Paths are shortened; there is one such error per crate whose license you did not allow.) Restore `allow = ["MIT", "Apache-2.0"]`. Now ban a crate by name: add this line to the `[bans]` table, right after `wildcards = "deny"`, and run `cargo deny check bans`:

```toml
deny = [{ crate = "atty", reason = "unmaintained, use std::io::IsTerminal" }]
```

The ban produces its own error, with your reason:

```text
error[banned]: crate 'atty = 0.2.14' is explicitly banned
   ┌─ deny.toml:16:20
   │
16 │ deny = [{ crate = "atty", reason = "unmaintained, use std::io::IsTerminal" }]
   │                    ━━━━             ───────────────────────────────────── reason
   │                    │                 
   │                    banned here
   │
   ├ atty v0.2.14
     └── p10-audit-demo v0.1.0

bans FAILED
```

Remove the `deny` line again (or keep it: once `atty` is gone it does no harm), remove `atty` from the project and update `regex`. Now all four checks pass:

```text
advisories ok, bans ok, licenses ok, sources ok
```

> 💡 **Tip:** `cargo deny check advisories` can ignore a finding with a **reason**, written in the file, in code review, where everybody sees it:
> `ignore = [{ id = "RUSTSEC-2021-0145", reason = "atty only checks for a terminal; replace by 2027-01" }]`. Use it as a last resort, and always with a reason and a date.

## 5. Automate it

A check that nobody runs does not protect anything. Create `.github/workflows/security.yml` in the repository root:

```yaml
name: security

on:
  push:
  pull_request:
  schedule:
    - cron: "17 4 * * 1" # every Monday: new advisories appear even when your code does not change

permissions:
  contents: read

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - name: Install the tools
        run: cargo install --locked cargo-audit cargo-deny
      - name: Known vulnerabilities (RustSec)
        run: cargo audit
      - name: Licenses, bans, sources and advisories
        run: cargo deny check
```

Real GitHub Actions runners cannot be started from your Mac, but the syntax can be checked locally with `actionlint` (Lesson 09.12): the command `actionlint .github/workflows/security.yml` prints nothing and exits with `0`. The three triggers mean: on every push, on every pull request, and weekly. `permissions: contents: read` gives the job's token only read rights, as Lesson 09.12 recommends for every workflow. (Faster alternatives exist: the actions `EmbarkStudios/cargo-deny-action` and `rustsec/audit-check` install ready-made binaries instead of compiling the tools.)

## 6. Before you add a dependency

A short checklist. It takes five minutes and prevents most trouble:

| Question | Where to look |
|---|---|
| Do I need it? Can I write ten lines or use `std`? | Your own code. Every crate is attack surface. |
| Is the name exactly right? | Compare the name and the repository link with the documentation you used (typosquatting). |
| Is it maintained and used? | crates.io page: recent release, number of downloads, the repository activity. |
| Who maintains it? | One person or a team? Is it an organisation crate (for example `rust-lang`, `tokio-rs`)? |
| What does it pull in? | `cargo tree -p <crate>` after adding; fewer transitive crates is better. |
| Does it use `unsafe`, `build.rs` or a procedural macro? | Skim the repository. Build scripts and macros run during compilation. |
| Are the features minimal? | `default-features = false` and enable only what you use. |

For critical projects there are tools that record "a trusted person reviewed this version of this crate", such as `cargo vet`; they build on the same ideas. Also keep updates **regular**: a project that updates a little every week is easier to fix than one that has been frozen for two years. After `cargo update`, run the tests.

## Common mistakes

**1. Not committing `Cargo.lock` for a program.** Then two builds of the same commit can contain different dependency versions and `cargo audit` has no exact list to check.

**2. Running the tools once.** The advisory database changes every week. Put them in CI on a schedule.

**3. Ignoring warnings.** "Unmaintained" is a warning today and a vulnerability nobody fixes tomorrow. Plan the replacement.

**4. Using `cargo update` without testing.** Updates can change behaviour even inside the allowed version range. Update in a branch, run the tests, and read the changelog of important crates.

**5. Dependencies written as `*`.** `serde = "*"` means "any version, even a future incompatible or hostile one". `cargo-deny` forbids it with `wildcards = "deny"`.

**6. `cargo install` without `--locked`** for tools: the tool is built with the newest versions of its own dependencies, not the tested ones.

## Try it yourself

1. Run `cargo tree` in one of your own earlier packages (for example `code/p10-hashing`). How many crates did the few dependencies bring in?
2. Run `cargo audit` there. Does it find anything?
3. In the demo package, change `regex = "1.5"` to `regex = "*"` in `Cargo.toml` and run `cargo deny check bans`. What happens? Restore it afterwards.

When you are done, delete the demo: `rm -rf ~/projects/rust/code/p10-audit-demo`.

## Exercises

### Exercise 1 — Who needs it? (easy)

After fixing the demo (`regex` updated, `atty` removed), find out **which crates** depend on `memchr`.

<details>
<summary>Hint</summary>

The inverse tree: `cargo tree -i <crate>`.

</details>

<details>
<summary>Answer</summary>

```bash
cargo tree -i memchr
```

```text
memchr v2.8.3
├── aho-corasick v1.1.5
│   ├── regex v1.13.2
│   │   └── p10-audit-demo v0.1.0 (/Users/ada/projects/rust/code/p10-audit-demo)
│   └── regex-automata v0.4.19
│       └── regex v1.13.2 (*)
├── regex v1.13.2 (*)
└── regex-automata v0.4.19 (*)
```

Why it works: `-i` (inverse) draws the tree upside down from the chosen crate, so you see every path from your package to it. `(*)` marks a branch that was already shown above.

</details>

### Exercise 2 — Ignore with a reason (medium)

Put `atty` back (you may keep `regex` updated) and make `cargo deny check advisories` pass **without** removing `atty`: ignore the two `atty` advisories, with a reason. Then explain why this is a last resort.

<details>
<summary>Hint</summary>

In the `[advisories]` table use `ignore = [{ id = "RUSTSEC-…", reason = "…" }]`. The IDs are in the output above: `RUSTSEC-2021-0145` and `RUSTSEC-2024-0375`. Remember that the `[bans]` rule also bans `atty`; this exercise is only about `advisories`.

</details>

<details>
<summary>Answer</summary>

```toml
[advisories]
yanked = "deny"
ignore = [
    { id = "RUSTSEC-2021-0145", reason = "atty is only used to check for a terminal; replace with IsTerminal by 2027-01" },
    { id = "RUSTSEC-2024-0375", reason = "same: atty, tracked in the backlog" },
]
```

```bash
cargo deny check advisories
```

```text
advisories ok
```

Why it works: ignored advisories are no longer reported. It is a last resort, because the problem is still there: the crate is still unmaintained and unsound. The `reason` and the date make the decision visible in code review, so it is not forgotten.

</details>

### Exercise 3 — A policy test (hard)

Write an integration test `tests/policy.rs` that fails if a crate you decided never to use appears in `Cargo.lock`. The list of banned names is a constant in the test (start with `atty` and `time`). Check that the test passes on the fixed demo and fails when `atty` is in the project.

<details>
<summary>Hint</summary>

A lock file contains one `name = "crate"` line for every package. `env!("CARGO_MANIFEST_DIR")` is the path of the folder with `Cargo.toml`, known at compile time. Read the file with `std::fs::read_to_string`.

</details>

<details>
<summary>Answer</summary>

```rust
//! A policy test: fails if a crate we decided never to use appears in Cargo.lock.

const BANNED: &[&str] = &["atty", "time"];

#[test]
fn no_banned_crates_in_cargo_lock() {
    let lock = std::fs::read_to_string(concat!(env!("CARGO_MANIFEST_DIR"), "/Cargo.lock")).unwrap();
    for banned in BANNED {
        let line = format!("name = \"{banned}\"");
        assert!(!lock.lines().any(|l| l == line), "banned crate `{banned}` is in Cargo.lock");
    }
}
```

With `atty` in the project, `cargo test --test policy` shows:

```text
test no_banned_crates_in_cargo_lock ... FAILED
thread 'no_banned_crates_in_cargo_lock' (5319383) panicked at tests/policy.rs:10:9:
banned crate `atty` is in Cargo.lock
```

and after `cargo remove atty` it shows `test no_banned_crates_in_cargo_lock ... ok`.

Why it works: the exact line comparison avoids false matches such as a crate called `atty-extra`. This is a small, homemade version of the `bans` check, and it shows that a policy can be a normal test. In real projects prefer `cargo deny`, which also understands the dependency graph.

</details>

## Quiz

1. What is a transitive dependency?

<details>
<summary>Answer</summary>

A dependency of one of your dependencies. You did not choose it, but it is part of your program and your attack surface. `cargo tree` shows all of them.

</details>

2. `cargo audit` reports `Warning: unmaintained` and "No fixed upgrade is available!". What are your options?

<details>
<summary>Answer</summary>

Replace the crate (best: with `std` or a maintained crate), or ignore the warning with a written reason and a date to review. There is no version to update to.

</details>

3. Why run `cargo audit` on a schedule, not only when the code changes?

<details>
<summary>Answer</summary>

Because new advisories are published for old code. A project that was clean last month can have a known vulnerability this week without any change in your repository.

</details>

4. Name the four checks of `cargo-deny`.

<details>
<summary>Answer</summary>

`advisories`, `licenses`, `bans` and `sources`.

</details>

5. What is typosquatting, and what is one defence?

<details>
<summary>Answer</summary>

Publishing a malicious crate with a name very close to a popular one, hoping people mistype it. Defences: copy the dependency line from the official documentation, check the repository and download numbers on crates.io, and review the diff of `Cargo.toml` and `Cargo.lock` in every pull request.

</details>

## Summary

- Your program contains much code written by others; the supply chain is part of your attack surface.
- `cargo tree` shows what you depend on; `Cargo.lock` pins exact versions and should be committed for programs.
- `cargo audit` finds known vulnerabilities, unmaintained and unsound crates in `Cargo.lock`; fix by `cargo update -p`, or by replacing the crate.
- `cargo deny check` enforces policy: advisories (including yanked), licenses, bans and sources, configured in `deny.toml`.
- Run both in CI on every push and on a weekly schedule; ignore findings only with a reason and a date.
- Before adding a crate: do you need it, is the name right, is it maintained, what does it pull in, and does it run code at build time.

## Sources and further reading

- [RustSec advisory database](https://rustsec.org/) — the advisories behind `cargo-audit`, searchable by crate.
- [cargo-audit on GitHub](https://github.com/rustsec/rustsec/tree/main/cargo-audit) — options, configuration (`audit.toml`) and CI use.
- [cargo-deny book](https://embarkstudios.github.io/cargo-deny/) — every option of `deny.toml` and the four checks.
- [The Cargo Book: Cargo.toml vs Cargo.lock](https://doc.rust-lang.org/cargo/guide/cargo-toml-vs-cargo-lock.html) — why and when to commit `Cargo.lock`.
- [The Cargo Book: cargo tree](https://doc.rust-lang.org/cargo/commands/cargo-tree.html) — all options of `cargo tree`.
- [cargo-vet book](https://mozilla.github.io/cargo-vet/) — recording who reviewed which dependency version.

---

[🏠 Start](../START-HERE.md) · [← 10.12 Memory safety and reviewing unsafe code](12-memory-safety-and-reviewing-unsafe.md) · [📋 Phase 10](README.md) · [10.14 Project: an encrypted secrets vault →](14-project-encrypted-secrets-vault.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
