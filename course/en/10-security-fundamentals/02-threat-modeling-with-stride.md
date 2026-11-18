<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.1 The security mindset and the CIA triad](01-security-mindset-and-the-cia-triad.md) · [📋 Phase 10](README.md) · [10.3 The OWASP Top 10 →](03-the-owasp-top-10.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/02-threat-modeling-with-stride.md) <!-- nav -->

# Lesson 10.2 — Threat modeling with STRIDE

> **Phase 10 — Security fundamentals** · Lesson 2 of 15 · ⏱ about 135 minutes

## What you will learn

- What threat modeling is and why it is cheaper to do it before you write the code.
- How to draw a simple data-flow diagram with trust boundaries.
- The six STRIDE categories of threats and how to apply them to each part of a system.
- How to rate and rank threats, and how to choose what to do with each one.
- How to keep a small threat model as data in a Rust program, with tests.

## Before you start

- You finished [Lesson 10.1 — The security mindset and the CIA triad](01-security-mindset-and-the-cia-triad.md).
- You know enums and `match` ([Lesson 04.3](../04-structs-enums-collections-errors/03-enums.md)) and traits such as `Display` ([Lesson 05.2](../05-rust-intermediate/02-traits.md)).
- Create the package for this lesson: `cargo new --lib code/p10-threats`, then `mkdir src/bin`.

## New words

| Word | Meaning |
|---|---|
| **threat modeling** | A structured way to find what can go wrong with a system, **before** it goes wrong, and to decide what to do about it. |
| **STRIDE** | A checklist of six kinds of threats: **S**poofing, **T**ampering, **R**epudiation, **I**nformation disclosure, **D**enial of service, **E**levation of privilege. |
| **DFD** (Data-Flow Diagram) | A simple picture of a system: who talks to it, which programs run, where data is stored, and how data moves. |
| **external entity** | Something outside your system that talks to it: a user, another service. |
| **process** | A part of your system that runs code and handles data. |
| **data store** | A place where data rests: a file, a database. |
| **data flow** | Data moving between the parts: a network request, a file read, a command-line argument. |
| **spoofing** | Pretending to be someone or something else (another user, another server). |
| **tampering** | Changing data or code without permission. |
| **repudiation** | Denying that you did something ("I never deleted that"), when the system cannot prove otherwise. |
| **information disclosure** | Letting data reach someone who must not see it. |
| **elevation of privilege** | Getting more rights than you were given, for example a normal user acting as an administrator. |
| **mitigation** | An action that reduces a threat: a check, an encryption, a limit. |
| **attack tree** | A diagram that starts with the attacker's goal and splits it into the different ways to reach it. |

## 1. Why think about threats before coding

Fixing a design problem after the program is finished is expensive: the problem is built into every
part. Finding it on paper takes an hour. **Threat modeling** is that hour on paper. It answers four
questions, which you can remember as a loop:

1. **What are we building?** Draw it.
2. **What can go wrong?** Use a checklist (STRIDE) so you do not depend on your imagination alone.
3. **What will we do about it?** For each threat choose an action.
4. **Did we do a good job?** Review the result, and repeat it when the design changes.

You do not need special software. A sheet of paper or a text file in the repository is enough, and a
small model that exists is much better than a perfect model that never gets written.

**Our example in this lesson** is the program you will build at the end of the phase: a **secrets
vault**. It is a command-line program. The user starts it and types a *master password*; the program
opens an encrypted file and prints or stores small secrets such as API keys (an API, Application Programming Interface, is the way programs call a service; its key is a password for programs). (Encryption is taught in
[Lesson 10.10](10-cryptography-encryption-and-signatures.md); here we only need to know that the file
is meant to be unreadable without the password.)

## 2. Step 1: draw the system

A data-flow diagram uses only four kinds of shapes. Here is the vault:

```text
                         trust boundary: the user's computer
   ┌────────────────────────────────────────────────────────────────────┐
   │                                                                    │
   │  [ User ]  ── master password, commands ──►  ( vault program )     │
   │  external                                       process            │
   │  entity    ◄── secrets, messages ───────────                       │
   │                                                   │      ▲         │
   │                                        write file │      │ read    │
   │                                                   ▼      │         │
   │                                              ═══ vault file ═══    │
   │                                                  data store        │
   └────────────────────────────────────────────────────────────────────┘
        ▲
        │ copies of the file: backups, a stolen laptop, a shared folder
     [ Other people and programs ]  external entities
```

Reading the picture:

- **Square brackets** mark an external entity: the user, and "other people and programs" that can
  reach the file.
- **Round brackets** mark a process: our program.
- **Double lines** mark a data store: the encrypted file.
- **Arrows** are data flows. The big box is a trust boundary.

The most useful part is the arrows that **cross** a trust boundary or leave the box: the password and
commands entering the program, and the file that can be copied away. Attacks happen at those places.

> 💡 **Tip:** when you draw, put a name on every arrow ("master password", "encrypted bytes") and a
> name for the data. Many threats are discovered only because you had to write the name down.

## 3. Step 2: the STRIDE checklist

STRIDE was created at Microsoft. Each letter is one kind of threat, and each kind attacks one property
that a secure system should have:

| Letter | Threat | Property it breaks | Question to ask | Typical mitigation |
|---|---|---|---|---|
| **S** | Spoofing | Authenticity | Can someone pretend to be someone else? | Authentication: passwords, keys, certificates ([Lesson 10.6](06-authentication-and-password-hashing.md)) |
| **T** | Tampering | Integrity | Can someone change data or code? | Permissions, signatures, authenticated encryption ([10.10](10-cryptography-encryption-and-signatures.md)) |
| **R** | Repudiation | Non-repudiation (accountability) | Can someone deny an action and we cannot prove it? | Audit logs that record who did what and when |
| **I** | Information disclosure | Confidentiality | Can data reach someone who must not see it? | Encryption, access control, not logging secrets |
| **D** | Denial of service | Availability | Can someone make it stop or slow down? | Limits, timeouts, validation ([10.4](04-input-validation.md)) |
| **E** | Elevation of privilege | Authorization | Can someone do more than they should? | Authorization checks, least privilege ([10.7](07-authorization-sessions-and-tokens.md)) |

Compare with the CIA triad from Lesson 10.1: **T** is integrity, **I** is confidentiality, **D** is
availability. STRIDE adds three questions about *who* is acting and *what they may do*.

## 4. Step 3: apply STRIDE to each element

You do not ask all six questions about every shape. Some questions make no sense for some shapes. This
table shows which letters to check for each kind of element:

| Element | S | T | R | I | D | E |
|---|---|---|---|---|---|---|
| External entity (user) | yes | | yes | | | |
| Process (our program) | yes | yes | yes | yes | yes | yes |
| Data store (file) | | yes | sometimes | yes | yes | |
| Data flow (arrow) | | yes | | yes | yes | |

Now go through the vault. Each row is a threat, with the element where it lives:

| # | Element | STRIDE | What can go wrong | What we do |
|---|---|---|---|---|
| 1 | vault file | I | A thief copies the file and reads the secrets | Encrypt it with a key made from the master password |
| 2 | vault file | T | Someone edits the file to change or remove a secret | Authenticated encryption: a changed file refuses to open |
| 3 | master password | S | An attacker guesses the password offline, with unlimited tries | Slow key derivation (Argon2id), minimum password length |
| 4 | vault program | I | A secret is printed in a log, an error message or a crash report | Never log values; wrapper types that hide secrets (Lesson 10.8) |
| 5 | vault file | D | A crash while saving leaves a half-written file and all secrets are lost | Write to a new file, then rename it over the old one |
| 6 | vault program | E | A secret named `../../x` makes the program write outside its folder | Validate names with an allow-list ([10.4](04-input-validation.md), [10.5](05-injection-attacks.md)) |
| 7 | user | R | A user says "I never deleted that secret" | A single-user tool: accept this risk (see Section 6) |

Notice the pattern: **one element can have several threats, and one threat can fit two letters.**
For example, threat 3 is also a kind of information disclosure. Do not spend time arguing about the
"right" letter. The letters are there to make you ask questions, and the goal is a good list of
mitigations.

## 5. Step 4: rate and rank

You cannot fix everything first. Rate every threat with the two numbers from Lesson 10.1 and multiply:

```text
risk = likelihood (1 to 3) × impact (1 to 3)       from 1 (ignore) to 9 (fix first)
```

| Score | Meaning |
|---|---|
| 7–9 | Fix before release. |
| 4–6 | Fix soon, or add a second layer of defense. |
| 1–3 | Accept it, or fix when it is cheap. |

Then choose one of four **responses** for each threat:

| Response | Meaning | Example |
|---|---|---|
| **Mitigate** | Reduce the risk with a control | Encrypt the file |
| **Eliminate** | Remove the feature that causes the threat | Do not offer "show all passwords" |
| **Transfer** | Let someone else carry it | Use a cloud key service, or buy insurance |
| **Accept** | Decide to live with it, and write that down | Threat 7: a one-person tool needs no audit log |

> 📏 **Convention:** the numbers are only for **ordering** the list. Do not argue whether a threat is a
> 5 or a 6. If two people disagree, take the higher one and move on.

## 6. A threat model as data in Rust

A threat model written once and forgotten is useless. A model that lives next to the code, and that
a test can check, stays alive. We will keep it as plain Rust data. The library part goes in
`src/lib.rs`. First the six categories:

```rust
use std::fmt;

/// The six STRIDE categories.
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum Stride {
    Spoofing,
    Tampering,
    Repudiation,
    InformationDisclosure,
    DenialOfService,
    ElevationOfPrivilege,
}

impl Stride {
    pub const ALL: [Stride; 6] = [
        Stride::Spoofing,
        Stride::Tampering,
        Stride::Repudiation,
        Stride::InformationDisclosure,
        Stride::DenialOfService,
        Stride::ElevationOfPrivilege,
    ];
}

impl fmt::Display for Stride {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        let name = match self {
            Stride::Spoofing => "Spoofing",
            Stride::Tampering => "Tampering",
            Stride::Repudiation => "Repudiation",
            Stride::InformationDisclosure => "Information disclosure",
            Stride::DenialOfService => "Denial of service",
            Stride::ElevationOfPrivilege => "Elevation of privilege",
        };
        write!(f, "{name}")
    }
}
```

**What happens:**

1. `enum Stride` has one variant for each letter. `Copy` lets us pass a category around cheaply, and
   `PartialEq` lets us compare two categories with `==`.
2. `Stride::ALL` is a constant array that lists all six. We need it to ask "which category is missing?".
3. `impl fmt::Display` gives each category a readable name for output.

Now the threat itself, the ordering, and the check for gaps:

```rust
/// One entry of the threat model.
#[derive(Debug, Clone)]
pub struct Threat {
    pub id: u32,
    pub target: &'static str,
    pub category: Stride,
    pub description: &'static str,
    pub likelihood: u8, // 1 = low, 3 = high
    pub impact: u8,     // 1 = low, 3 = high
    pub mitigation: &'static str,
}

impl Threat {
    pub fn new(
        id: u32,
        target: &'static str,
        category: Stride,
        description: &'static str,
        likelihood: u8,
        impact: u8,
        mitigation: &'static str,
    ) -> Self {
        Threat {
            id,
            target,
            category,
            description,
            likelihood,
            impact,
            mitigation,
        }
    }

    /// Risk is likelihood times impact: from 1 to 9.
    pub fn risk(&self) -> u8 {
        self.likelihood * self.impact
    }
}

/// Returns the threats sorted from the highest risk to the lowest.
/// Threats with the same risk keep their order of `id`.
pub fn prioritize(mut threats: Vec<Threat>) -> Vec<Threat> {
    threats.sort_by(|a, b| b.risk().cmp(&a.risk()).then(a.id.cmp(&b.id)));
    threats
}

/// Which STRIDE categories have no threat yet? A gap in the model.
pub fn missing_categories(threats: &[Threat]) -> Vec<Stride> {
    Stride::ALL
        .into_iter()
        .filter(|c| !threats.iter().any(|t| t.category == *c))
        .collect()
}
```

**What happens:**

1. `Threat` has the fields from our table: where it is, which letter, what can go wrong, the two scores
   and the mitigation. `&'static str` is a text that lives for the whole program, such as a string
   literal.
2. `risk()` multiplies the two scores. `u8` is enough: the largest result is `3 × 3 = 9`.
3. `prioritize` sorts a list so that the highest risk comes first. `b.risk().cmp(&a.risk())` compares in
   reverse (b before a) to get a descending order; `.then(a.id.cmp(&b.id))` breaks ties by `id`,
   so the order is always the same.
4. `missing_categories` keeps the categories for which **no** threat exists. A letter with no threat
   is not proof that the system is safe; it is a reminder to think harder about that letter.

Now the tests. Add them at the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn threat(id: u32, category: Stride, likelihood: u8, impact: u8) -> Threat {
        Threat::new(
            id,
            "test",
            category,
            "test threat",
            likelihood,
            impact,
            "none",
        )
    }

    #[test]
    fn risk_is_likelihood_times_impact() {
        assert_eq!(threat(1, Stride::Spoofing, 2, 3).risk(), 6);
    }

    #[test]
    fn highest_risk_comes_first() {
        let sorted = prioritize(vec![
            threat(1, Stride::Spoofing, 1, 1),
            threat(2, Stride::Tampering, 3, 3),
            threat(3, Stride::Repudiation, 2, 2),
        ]);
        let ids: Vec<u32> = sorted.iter().map(|t| t.id).collect();
        assert_eq!(ids, vec![2, 3, 1]);
    }

    #[test]
    fn gaps_are_reported() {
        let threats = vec![threat(1, Stride::Spoofing, 1, 1)];
        let missing = missing_categories(&threats);
        assert_eq!(missing.len(), 5);
        assert!(!missing.contains(&Stride::Spoofing));
    }
}
```

**What happens:**

1. The helper `threat(...)` builds a test threat. The tests check the multiplication (2 × 3 = 6), the
   order (ids `[2, 3, 1]` mean risks 9, 4, 1) and that one threat leaves five categories missing.

```bash
cargo test
```

```text
running 3 tests
test tests::risk_is_likelihood_times_impact ... ok
test tests::gaps_are_reported ... ok
test tests::highest_risk_comes_first ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

Finally a small program that holds three of the vault threats and prints the ranked list. Save it as
`src/bin/report.rs` and run `cargo run --bin report`:

```rust
use p10_threats::{Stride, Threat, missing_categories, prioritize};

fn main() {
    let threats = prioritize(vec![
        Threat::new(
            1,
            "vault file",
            Stride::InformationDisclosure,
            "A thief copies the vault file and reads the secrets",
            2,
            3,
            "Encrypt the file with a key made from the master password",
        ),
        Threat::new(
            2,
            "master password",
            Stride::Spoofing,
            "An attacker guesses the master password offline",
            3,
            3,
            "Slow key derivation (Argon2id) and a minimum length",
        ),
        Threat::new(
            3,
            "secret name",
            Stride::ElevationOfPrivilege,
            "A name such as ../../x makes the program touch other files",
            1,
            3,
            "Validate names with an allow-list; never use them as paths",
        ),
    ]);

    println!("{:<3} {:<5} {:<22} description", "id", "risk", "category");
    for t in &threats {
        println!(
            "{:<3} {:<5} {:<22} {}",
            t.id,
            t.risk(),
            t.category.to_string(),
            t.description
        );
    }

    let missing = missing_categories(&threats);
    let names: Vec<String> = missing.iter().map(|c| c.to_string()).collect();
    println!("categories without a threat yet: {}", names.join(", "));
}
```

```text
id  risk  category               description
2   9     Spoofing               An attacker guesses the master password offline
1   6     Information disclosure A thief copies the vault file and reads the secrets
3   3     Elevation of privilege A name such as ../../x makes the program touch other files
categories without a threat yet: Tampering, Repudiation, Denial of service
```

**What happens:**

1. `use p10_threats::{…}` imports our library. The package is named `p10-threats`; in code dashes become
   underscores.
2. `Threat::new(...)` fills in one threat each. `prioritize(vec![...])` returns them sorted: the
   password-guessing threat (risk 9) is first.
3. `{:<22}` pads a text to 22 characters, left-aligned, so the columns line up. `t.category.to_string()` is
   needed because `Display` values do not accept padding on their own unless the `Display` code
   uses `f.pad`; converting to a `String` first makes the padding work.
4. The last line lists the categories we have not thought about yet: **Tampering**, **Repudiation** and **Denial of service**.
   This is the model telling us it is incomplete.

> 🧪 **Testing:** a test such as `a_complete_model_has_no_gaps` (Exercise 2) turns "did we think about
> every letter?" into something that fails when someone forgets.

## 7. Attack trees: thinking from the goal

STRIDE starts from the parts of the system. An **attack tree** starts from the attacker's **goal** and
asks "how could they get there?" Each branch is a way; branches joined by OR are alternatives, and
branches joined by AND all have to happen.

```text
Goal: read the secrets in the vault
├── OR  get the vault file
│   ├── steal or copy the laptop
│   ├── find an old backup copy
│   └── read it from a shared folder with wrong permissions
└── OR  get the master password
    ├── guess it (offline, many tries per second)
    ├── watch the user type it
    └── find it in a shell history or a log
AND the attacker needs BOTH the file and the password
```

The attacker needs the file **and** the password, so you can make each part harder separately (defense
in depth). The tree also shows a weak place that you fully control: a password left in a shell history or a log.

## 8. When to do it, and how to keep it alive

- Do it at the **start** of a feature, and again when the design changes: a new input, a new
  network connection, a new kind of user.
- Keep the model in the repository, for example in a file `THREATS.md`, and review it in the same pull
  request that changes the design ([Lesson 08.13](../08-software-design/13-refactoring-and-code-review.md)).
- Turn each mitigation into a test, a check in the code, or a line in a checklist. A mitigation that
  nobody implemented is only a wish.

> 🔒 **Security:** threat modeling finds the problems you can imagine. It does not replace testing and
> review. The next lessons teach the common attacks, so your imagination has more material.

## Common mistakes

> ⚠️ **Common mistake:** making the model too big. If the diagram does not fit on one page, split it:
> one model for the program, another for its deployment.

> ⚠️ **Common mistake:** skipping the trust boundaries. Without them every arrow looks the same, and you
> forget that data crossing from outside needs checking.

> ⚠️ **Common mistake:** using the model once. A model that is older than the design is lying to you.

## Try it yourself

1. Type the library, the tests and `report.rs` and run `cargo test` and `cargo run --bin report`.
2. Add threats 2 (tampering) and 7 (repudiation) from the table in Section 4 to the report. What does the
   last line of the output say now?

## Exercises

### Exercise 1 — Name the letter (easy)

Which STRIDE category fits best? (a) Someone sends a request that pretends to come from an
administrator. (b) A program writes the user's password into a debug log. (c) A single huge upload
fills the disk. (d) A user clicks "delete account" and later says they never did, and there is no
record. (e) A normal user finds a URL (web address) that only administrators should open and it works. (f) A man in the
middle changes the amount in a payment request on the way.

<details>
<summary>Hint</summary>

Match each case to the question in the table of Section 3.

</details>

<details>
<summary>Answer</summary>

(a) Spoofing. (b) Information disclosure. (c) Denial of service. (d) Repudiation. (e) Elevation of
privilege. (f) Tampering (of a data flow).

</details>

### Exercise 2 — Close the gaps (medium)

Add a **test** that builds one threat for each of the six categories and checks that
`missing_categories` returns an empty list.

<details>
<summary>Hint</summary>

`Stride::ALL.into_iter()` gives every category. Use `enumerate()` to make a different `id` for each, and
the helper `threat(id, category, 1, 1)` from the tests module.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn a_complete_model_has_no_gaps() {
        let all: Vec<Threat> = Stride::ALL
            .into_iter()
            .enumerate()
            .map(|(i, c)| threat(i as u32, c, 1, 1))
            .collect();
        assert!(missing_categories(&all).is_empty());
    }
```

Put it inside `mod tests`. Why it works: the iterator yields each of the six categories once, so every
category has a threat and nothing is reported as missing.

</details>

### Exercise 3 — Model a small web notes service (hard)

A notes service has a browser (user), a web server process, and a database. Users log in with a
password and can read only their own notes. Draw the data-flow diagram in text, mark the trust
boundaries, and write at least one threat for **each** STRIDE letter with a mitigation and a risk score.

<details>
<summary>Hint</summary>

The boundaries are between the browser and the server (the internet) and between the server and the
database. For E, think about "reading someone else's note by changing the number in the address".

</details>

<details>
<summary>Answer</summary>

One possible answer (yours will differ, and that is fine):

```text
[Browser] -- login, note text --> (Web server) -- queries --> [[Database]]
          <-- pages ----------                <-- rows -----
   ^ boundary 1: the internet        ^ boundary 2: server to database
```

| STRIDE | Threat | Mitigation | L×I |
|---|---|---|---|
| S | Someone logs in as another user by guessing a password | Argon2id hashes, rate limit on logins | 3×3=9 |
| T | The note text is changed on the network | HTTPS (TLS, Transport Layer Security, Lesson 10.11) | 2×3=6 |
| R | A user denies deleting a note | Audit log with user and time | 1×1=1 (accept) |
| I | Opening `/notes/42` shows another user's note | Check that the note belongs to the logged-in user | 3×3=9 |
| D | One request uploads a 10 GB (gigabyte) note | Size limit, timeout | 2×2=4 |
| E | A normal user calls `/admin/delete` | Authorization check in every handler | 2×3=6 |

Why it works: every letter produced a concrete question about one element, and each answer is a check
that you can later write as code or as a test.

</details>

## Quiz

1. What are the four questions of threat modeling?

<details>
<summary>Answer</summary>

What are we building? What can go wrong? What will we do about it? Did we do a good job?

</details>

2. Name the six STRIDE letters and the property each one breaks.

<details>
<summary>Answer</summary>

Spoofing (authenticity), Tampering (integrity), Repudiation (non-repudiation), Information disclosure
(confidentiality), Denial of service (availability), Elevation of privilege (authorization).

</details>

3. Why are trust boundaries the most important lines in a data-flow diagram?

<details>
<summary>Answer</summary>

Data that crosses a boundary comes from a place you do not control, so these are the places where attacks
enter and where checks are needed.

</details>

4. What are the four possible responses to a threat?

<details>
<summary>Answer</summary>

Mitigate, eliminate, transfer, accept.

</details>

5. What does it mean when `missing_categories` returns `[Repudiation]`?

<details>
<summary>Answer</summary>

No threat of that category is in the model yet. It does not prove that the system is safe; it asks you to
think about that letter and either add a threat or write down why it does not apply.

</details>

6. Why are the risk numbers only used for ordering?

<details>
<summary>Answer</summary>

They are rough guesses. Their purpose is to decide what to work on first, not to be exact.

</details>

## Summary

- Threat modeling is a short, structured look at a design before things go wrong.
- Draw the system with four shapes and mark the trust boundaries.
- STRIDE: Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege.
- Check the letters that make sense for each element type.
- Rate with likelihood × impact, then mitigate, eliminate, transfer or accept, and write the decision down.
- Keep the model in the repository (and as data and tests), and update it when the design changes.
- Attack trees start from the attacker's goal and show alternative and combined paths.

## Sources and further reading

- [OWASP Threat Modeling Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Threat_Modeling_Cheat_Sheet.html) — a practical guide to the four questions, data-flow diagrams and STRIDE.
- [Threat Modeling Manifesto](https://www.threatmodelingmanifesto.org/) — the values and principles of threat modeling, written by practitioners.
- [Microsoft — Threats in the Threat Modeling Tool](https://learn.microsoft.com/en-us/azure/security/develop/threat-modeling-tool-threats) — the STRIDE categories with detailed examples and mitigations.
- [OWASP — Threat Modeling](https://owasp.org/www-community/Threat_Modeling) — the process overview, including ranking threats.
- [The Rust standard library — `slice::sort_by`](https://doc.rust-lang.org/std/primitive.slice.html#method.sort_by) — the sorting method used in `prioritize`.

---

[🏠 Start](../START-HERE.md) · [← 10.1 The security mindset and the CIA triad](01-security-mindset-and-the-cia-triad.md) · [📋 Phase 10](README.md) · [10.3 The OWASP Top 10 →](03-the-owasp-top-10.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
