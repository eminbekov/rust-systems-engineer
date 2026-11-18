<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.2 Threat modeling with STRIDE](02-threat-modeling-with-stride.md) · [📋 Phase 10](README.md) · [10.4 Input validation: never trust input →](04-input-validation.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/03-the-owasp-top-10.md) <!-- nav -->

# Lesson 10.3 — The OWASP Top 10

> **Phase 10 — Security fundamentals** · Lesson 3 of 15 · ⏱ about 135 minutes

## What you will learn

- What OWASP is and what the "Top 10" list is (and is not).
- The ten risk categories of the 2025 edition, in plain words, and which lesson of this course covers each one.
- Four categories in practice with small Rust programs: broken access control, security misconfiguration, mishandled errors (fail open) and weak logging.
- How to turn a category into a checklist question for your own code.

## Before you start

- You finished [Lesson 10.2 — Threat modeling with STRIDE](02-threat-modeling-with-stride.md).
- You know error handling ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)) and `Option` ([Lesson 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)).
- Create the package: `cargo new code/p10-owasp`, then `mkdir src/bin` and `rm src/main.rs`. Every program below is one file in `src/bin/`.

## New words

| Word | Meaning |
|---|---|
| **OWASP** (Open Worldwide Application Security Project) | A non-profit community that publishes free guides, tools and lists about application security. |
| **OWASP Top 10** | A list of the ten most serious kinds of risk for web applications, made from real data and updated every few years. This lesson uses the 2025 edition. |
| **access control** | The rules that decide who may do what (read this note, delete that user). Also called authorization (Lesson 10.7). |
| **IDOR** (Insecure Direct Object Reference) | A flaw where the program takes an identifier from the request (`/notes/42`) and uses it without checking that this user may see that object. |
| **SSRF** (Server-Side Request Forgery) | An attack where the attacker makes your server send a request to an address the attacker chose, often an internal one that the attacker cannot reach directly. |
| **misconfiguration** | A wrong or unsafe setting: a default password, debug mode in production, a service open to the whole network. |
| **fail open / fail closed** | What a check does when it cannot decide. *Fail open* allows the action; *fail closed* refuses it. Security checks must fail closed. |
| **audit log** | A record of important events (logins, denied requests, changes), kept so that attacks and mistakes can be found later. |
| **log injection** | An attack where text typed by the attacker contains a newline and so creates fake lines in a log file. |
| **credential stuffing** | Trying passwords stolen from one website on many other websites, because many people reuse passwords. |
| **SQL** (Structured Query Language) | The language used to talk to most databases (Lesson 10.5). |

## 1. What the OWASP Top 10 is

OWASP is a community that publishes free security knowledge. Its best-known document is the **Top
10**: a list of the ten categories of weakness that cause the most serious damage in web applications.
OWASP collects data from many real applications and from security experts, and publishes a new
edition every few years. The edition used here is **OWASP Top 10:2025**.

Three things to know before you read the list:

1. It is a list of **categories**, not of single bugs. "Injection" contains dozens of different attacks.
2. It is an **awareness** document. It tells you where to look; it is not a complete checklist of everything
   that can go wrong. A program that avoids all ten can still have other bugs.
3. It is about web applications, but nearly everything applies to any program that takes input from
   outside: command-line tools, services, libraries.

## 2. The ten categories of 2025

| ID | Category | In plain words | Where this course teaches it |
|---|---|---|---|
| A01 | **Broken Access Control** | Users can do things they should not (read another user's data, open admin pages). This category also includes SSRF. | Section 3 here; [10.7](07-authorization-sessions-and-tokens.md) |
| A02 | **Security Misconfiguration** | Unsafe settings: default passwords, debug mode on, too many open ports, detailed error pages. | Section 4 here; [10.8](08-secrets-management.md) |
| A03 | **Software Supply Chain Failures** | Problems in the code and tools you depend on: a vulnerable or malicious library, a compromised build or update system. | [10.13](13-supply-chain-security.md) |
| A04 | **Cryptographic Failures** | No encryption where it is needed, weak algorithms, bad key handling, passwords stored badly. | [10.6](06-authentication-and-password-hashing.md), [10.9](09-cryptography-hashes-macs-and-randomness.md), [10.10](10-cryptography-encryption-and-signatures.md), [10.11](11-tls-and-certificates.md) |
| A05 | **Injection** | Input that is taken as part of a command or query (SQL, shell, file path) and changes its meaning. | [10.4](04-input-validation.md), [10.5](05-injection-attacks.md) |
| A06 | **Insecure Design** | The design itself is missing a protection, so no amount of careful coding can fix it. | [10.2](02-threat-modeling-with-stride.md) |
| A07 | **Authentication Failures** | Weak passwords allowed, no protection against guessing, broken sessions. | [10.6](06-authentication-and-password-hashing.md), [10.7](07-authorization-sessions-and-tokens.md) |
| A08 | **Software or Data Integrity Failures** | Trusting code or data without checking that it is genuine: unsigned updates, unsafe deserialization. | [10.10](10-cryptography-encryption-and-signatures.md), [10.13](13-supply-chain-security.md) |
| A09 | **Security Logging and Alerting Failures** | Important events are not recorded, or nobody is told, so attacks go unnoticed. | Section 6 here |
| A10 | **Mishandling of Exceptional Conditions** | Errors and unusual situations are handled badly: crashes, fail-open checks, leaking error details. | Section 5 here; [10.4](04-input-validation.md) |

Compared with the 2021 edition, Security Misconfiguration moved up to second place; the old
"Vulnerable and Outdated Components" grew into the wider Supply Chain category; Server-Side Request
Forgery became part of Broken Access Control; and Mishandling of Exceptional Conditions is new.

> 💡 **Tip:** use the list as questions you ask during code review: "Who is allowed to call this?"
> (A01), "What is the default?" (A02), "What do we depend on?" (A03), "What happens when this fails?" (A10).

## 3. A01 Broken Access Control: the number in the address

Imagine a notes application. You are logged in as `ana`, and your note is at `/notes/1`. You change
the `1` to `2` in the address bar. If the program returns the note, it has an **IDOR**: it checked that
you are logged in, but not that note 2 is yours. This is the most common serious flaw on the web, and
no clever attack is needed.

Save this as `src/bin/access.rs`. We simulate the application with a list in memory:

```rust
struct Note {
    id: u32,
    owner: &'static str,
    text: &'static str,
}

/// Broken: anyone who knows (or guesses) an id gets the note.
fn get_note_unchecked(notes: &[Note], id: u32) -> Option<&Note> {
    notes.iter().find(|n| n.id == id)
}

/// Fixed: the note must exist AND belong to the logged-in user.
/// Both failures give the same answer, so an attacker cannot learn which ids exist.
fn get_note<'a>(notes: &'a [Note], user: &str, id: u32) -> Option<&'a Note> {
    notes.iter().find(|n| n.id == id && n.owner == user)
}

fn main() {
    let notes = [
        Note {
            id: 1,
            owner: "ana",
            text: "my shopping list",
        },
        Note {
            id: 2,
            owner: "bob",
            text: "bob's private diary",
        },
    ];

    // "ana" is logged in and changes the number 1 to 2, like editing /notes/2 in a web address.
    let found = get_note_unchecked(&notes, 2);
    println!("unchecked: {:?}", found.map(|n| n.text));

    let found = get_note(&notes, "ana", 2);
    println!("checked:   {:?}", found.map(|n| n.text));
    let found = get_note(&notes, "ana", 1);
    println!("own note:  {:?}", found.map(|n| n.text));
}
```

**What happens:**

1. `Note` has an `id`, an `owner` and some `text`. The fields are `&'static str`, text that lives for
   the whole program.
2. `get_note_unchecked` looks only at the id. This is the broken version.
3. `get_note` finds a note only if the id matches **and** the owner is the current user. If either part
   fails it returns `None`, so "does not exist" and "exists but is not yours" look exactly the same.
   The attacker cannot even learn which ids are in use.
4. In `main`, `ana` asks for note 2, first with the broken function and then with the fixed one, and
   at last for her own note 1.

```text
unchecked: Some("bob's private diary")
checked:   None
own note:  Some("my shopping list")
```

The broken function gives `ana` Bob's diary. The tests prove the fix and keep it fixed. Add them at
the end of the same file:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn a_user_cannot_read_a_note_of_another_user() {
        let notes = [Note {
            id: 2,
            owner: "bob",
            text: "private",
        }];
        assert!(get_note(&notes, "ana", 2).is_none());
        assert!(get_note(&notes, "bob", 2).is_some());
    }

    #[test]
    fn missing_and_forbidden_look_the_same() {
        let notes = [Note {
            id: 2,
            owner: "bob",
            text: "private",
        }];
        assert_eq!(
            get_note(&notes, "ana", 2).map(|n| n.id),
            get_note(&notes, "ana", 99).map(|n| n.id)
        );
    }
}
```

```bash
cargo test --bin access
```

```text
running 2 tests
test tests::missing_and_forbidden_look_the_same ... ok
test tests::a_user_cannot_read_a_note_of_another_user ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🔒 **Security:** check access on **the server side, on every request, for every object**. Do not rely
> on hiding links, on random-looking ids (they help, but are not a check), or on the user interface.
> Deny by default: access is allowed only when a rule says yes.

**SSRF in one paragraph.** Some programs fetch a URL (Uniform Resource Locator, a web address) given by the user (a link preview, an image
import). An attacker supplies an internal address such as `http://127.0.0.1:8080/admin`, and your server
dutifully requests it from the inside, where the firewall trusts it. The defense is the same idea:
allow only the hosts you expect (an allow-list, [Lesson 10.4](04-input-validation.md)), and never
follow redirects blindly.

## 4. A02 Security Misconfiguration: safe by default

Many breaches use no bug at all: the service ran with a default password, with debug mode on, or
listened on every network interface. The best defense is a configuration that is **safe by default**
and that **refuses to start** when something dangerous is set. Save `src/bin/config.rs`:

```rust
use std::net::{IpAddr, Ipv4Addr};

#[derive(Debug)]
struct Config {
    bind: IpAddr,
    debug: bool,
    admin_password: Option<String>,
}

/// Secure by default: local only, no debug output, and NO default password.
impl Default for Config {
    fn default() -> Self {
        Config {
            bind: IpAddr::V4(Ipv4Addr::LOCALHOST),
            debug: false,
            admin_password: None,
        }
    }
}

impl Config {
    /// Refuse to start with a configuration that is known to be unsafe.
    fn validate(&self) -> Result<(), String> {
        match &self.admin_password {
            None => return Err(String::from("admin password is not set")),
            Some(p) if p.len() < 12 => {
                return Err(String::from(
                    "admin password must have at least 12 characters",
                ));
            }
            Some(_) => {}
        }
        if self.debug && !self.bind.is_loopback() {
            return Err(String::from(
                "debug mode is allowed only on a local address",
            ));
        }
        Ok(())
    }
}

fn main() {
    let config = Config::default();
    println!(
        "default config: bind={} debug={}",
        config.bind, config.debug
    );
    println!("validate: {:?}", config.validate());

    let risky = Config {
        bind: IpAddr::V4(Ipv4Addr::UNSPECIFIED), // 0.0.0.0: reachable from the whole network
        debug: true,
        admin_password: Some(String::from("a-long-enough-password")),
    };
    println!("risky config: {:?}", risky.validate());
}
```

**What happens:**

1. `Default` gives the safest values: listen only on `127.0.0.1` (`LOCALHOST`, this computer only),
   debug off, and **no admin password at all**. There is no built-in "admin/admin" to forget.
2. `validate` returns an error text for each unsafe case: no password, a short password, or debug mode on a
   non-local address. `is_loopback()` is true for local-only addresses.
3. `Some(p) if p.len() < 12` is a *match guard*: this arm matches only when the extra `if` is true. (`len()` counts bytes, which is the same as characters for plain English letters; [Lesson 10.4](04-input-validation.md) treats text more carefully.)
4. `main` tries the default configuration and a risky one (`0.0.0.0` means "every network
   interface", so the debug mode would be reachable from outside). Both are refused.

```text
default config: bind=127.0.0.1 debug=false
validate: Err("admin password is not set")
risky config: Err("debug mode is allowed only on a local address")
```

> 📏 **Convention:** a program that starts with an unsafe configuration is a bug waiting for its
> time. Validate configuration at startup and fail with a clear message ("fail fast"). The
> twelve-factor approach to configuration is in [Lesson 08.15](../08-software-design/15-the-twelve-factor-app.md).

## 5. A10 Mishandling of Exceptional Conditions: fail closed

Every security check can itself fail: the database is down, the file is missing, the network times
out. What the check does **then** decides whether it is secure. Save `src/bin/failopen.rs`:

```rust
#[derive(Debug, PartialEq)]
enum Role {
    Admin,
    User,
}

#[derive(Debug)]
struct LookupError;

/// Pretend database: it is "down" for the user "carol".
fn lookup_role(user: &str) -> Result<Role, LookupError> {
    match user {
        "ana" => Ok(Role::Admin),
        "bob" => Ok(Role::User),
        _ => Err(LookupError),
    }
}

/// Broken: when the check itself fails, it lets the user in (fails open).
fn is_admin_open(user: &str) -> bool {
    match lookup_role(user) {
        Ok(role) => role == Role::Admin,
        Err(_) => true, // "do not block people because of our own error"
    }
}

/// Fixed: when the check fails, the answer is "no" (fails closed).
fn is_admin(user: &str) -> bool {
    match lookup_role(user) {
        Ok(role) => role == Role::Admin,
        Err(_) => false,
    }
}

fn main() {
    for user in ["ana", "bob", "carol"] {
        println!(
            "{user:<6} open: {:<5} closed: {}",
            is_admin_open(user),
            is_admin(user)
        );
    }
}
```

**What happens:**

1. `lookup_role` plays a role database. It knows `ana` and `bob`; for anyone else it returns an
   `Err`, as if the database could not answer.
2. `is_admin_open` handles the `Err` case with `true`. The developer thought "do not block people
   because of our own error". Now any failure of the lookup makes the user an administrator.
3. `is_admin` returns `false` on `Err`. The check **fails closed**.
4. `main` shows the three users with both functions.

```text
ana    open: true  closed: true
bob    open: false closed: false
carol  open: true  closed: false
```

For `carol` the two versions disagree: `open: true`. An attacker only has to make the lookup fail,
for example with a strange user name or by overloading the database. Add the test that protects the fix:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn unknown_user_is_not_admin() {
        assert!(!is_admin("carol"));
        assert!(is_admin("ana"));
        assert!(!is_admin("bob"));
    }
}
```

A second mistake in this category is the careless `unwrap()`. Save `src/bin/quantity_panic.rs`:

```rust
fn main() {
    let text = "twelve";
    let quantity: u32 = text.parse().unwrap();
    println!("quantity: {quantity}");
}
```

```text
thread 'main' (4685023) panicked at src/bin/quantity_panic.rs:3:38:
called `Result::unwrap()` on an `Err` value: ParseIntError { kind: InvalidDigit }
```

**What happens:** `"twelve"` is not a number, so `parse` returns an `Err`, and `unwrap()` turns the `Err`
into a panic. If a request handler runs this code with text from the network, one bad request stops the
program (or the thread), which is a denial of service ([Lesson 10.1](01-security-mindset-and-the-cia-triad.md)).
Use `match`, `?` or `map_err` and return an error to the caller, as you learned in [Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md).

> 🔒 **Security:** two more rules for errors. First, show the user a short, general message ("could
> not process the request") and write the details (file paths, SQL text, stack traces) only to the log. Details in
> an error page help an attacker. Second, always release resources (files, locks) on the error path too; Rust's
> drop rules do most of this for you.

## 6. A09 Security Logging and Alerting Failures

If nobody records the failed logins, the denied requests and the changes of permissions, an attack can
last for months unnoticed. Good logging answers: **who** did **what**, **when**, from **where**, and
with what **result**. But logs bring their own risks:

- **Never log secrets** or full personal data (passwords, tokens, card numbers). See Lesson 10.1, Section 3.1.
- **Never write raw user input into a log line.** A user name that contains a newline can forge a
  line. Save `src/bin/logging.rs`:

```rust
fn log_failed_login_bad(user: &str) {
    println!("WARN login failed for user {user}");
}

fn log_failed_login(user: &str) {
    // `{:?}` prints the text in quotes and turns a newline into the two characters \n
    println!("WARN login failed for user {user:?}");
}

fn main() {
    // The "user name" that an attacker typed into a login form:
    let attacker = "mallory\nINFO login succeeded for user admin";

    log_failed_login_bad(attacker);
    println!("---");
    log_failed_login(attacker);
}
```

```text
WARN login failed for user mallory
INFO login succeeded for user admin
---
WARN login failed for user "mallory\nINFO login succeeded for user admin"
```

**What happens:**

1. The "user name" contains `\n`, a newline, followed by text that looks like a second log entry.
2. `log_failed_login_bad` prints it as it is, so the output has **two** lines, and the second one
   falsely says that the admin logged in.
3. `log_failed_login` uses `{user:?}`, the `Debug` format: it puts the text in quotes and writes the
   newline as the two characters `\n`. One event stays one line.

Logging is not only writing; somebody (or some alert) must read the signals, for example many failed logins
from one address. Tools for structured logs and tracing are the topic of
[Lesson 09.10](../09-testing-debugging-benchmarking/10-logging-and-tracing.md).

## 7. The remaining categories, in one place

- **A03 Supply Chain:** every crate you add is code that runs with your rights. Lesson 10.13 teaches
  `cargo audit` and `cargo deny`; you already know `Cargo.lock` from [Lesson 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md).
- **A04 Cryptographic Failures:** use audited libraries and high-level APIs (Application Programming Interfaces, the functions a library offers). Never invent your own.
- **A05 Injection:** keep data and commands apart. Lesson 10.5 shows SQL, shell and path injection.
- **A06 Insecure Design:** a missing rate limit or a missing check is not an implementation slip. Model
  threats early (Lesson 10.2).
- **A07 Authentication Failures:** hash passwords with a slow algorithm, limit attempts, and do not
  reveal which part of a login was wrong. *Credential stuffing* works because people reuse passwords.
- **A08 Integrity Failures:** check signatures and checksums of what you download or load. Rust's `serde`
  does not run code from data, but untrusted data can still be huge or deeply nested, so limit sizes.

## Common mistakes

> ⚠️ **Common mistake:** learning the list by heart and thinking you are done. The Top 10 is a starting
> point. Use it together with threat modeling for your own program.

> ⚠️ **Common mistake:** protecting only the page in the user interface. Hiding the "Delete" button from
> normal users does not stop a direct request. The check must be on the server.

> ⚠️ **Common mistake:** writing `Err(_) => true` (or `unwrap_or(true)`) in a permission check "to be
> safe for the user". It is safe for the attacker. Fail closed.

## Try it yourself

1. Create the package, type the five programs and run them with `cargo run --bin <name>`.
2. In `access.rs`, add a third user `eve` with no notes. What does `get_note(&notes, "eve", 1)` return?
   Does the answer give her any information?

## Exercises

### Exercise 1 — Name the category (easy)

Which OWASP 2025 category fits best? (a) A website still uses the password `admin` that came with
the software. (b) The server returns the full database error, with the table names, to the browser.
(c) A user changes `?invoice=100` to `?invoice=101` and sees a stranger's invoice. (d) A library with a
known vulnerability has been in `Cargo.lock` for two years. (e) Hundreds of failed logins happened, and
nobody noticed.

<details>
<summary>Hint</summary>

Use the "In plain words" column of the table in Section 2.

</details>

<details>
<summary>Answer</summary>

(a) A02 Security Misconfiguration (a default password; it is also an authentication weakness).
(b) A10 Mishandling of Exceptional Conditions (also A02, verbose errors). (c) A01 Broken Access Control (IDOR).
(d) A03 Software Supply Chain Failures. (e) A09 Security Logging and Alerting Failures.

</details>

### Exercise 2 — Test a configuration (medium)

Add a **test** to `config.rs` that proves a configuration with a strong password on the default local
address is accepted, and another that proves that a configuration with a password of 11 characters is
refused.

<details>
<summary>Hint</summary>

Use struct update syntax: `Config { admin_password: Some(…), ..Config::default() }` copies all other fields
from the default.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn the_default_config_does_not_start() {
        assert!(Config::default().validate().is_err());
    }

    #[test]
    fn a_strong_local_config_is_accepted() {
        let config = Config {
            admin_password: Some(String::from("a-long-enough-password")),
            ..Config::default()
        };
        assert!(config.validate().is_ok());
    }

    #[test]
    fn an_eleven_character_password_is_refused() {
        let config = Config {
            admin_password: Some(String::from("elevenchars")),
            ..Config::default()
        };
        assert!(config.validate().is_err());
    }
}
```

Why it works: `elevenchars` has exactly 11 characters, so `p.len() < 12` is true and `validate` returns an
`Err`. Testing the border value (11 and 12) is the best way to catch an off-by-one mistake in a security rule.

</details>

### Exercise 3 — Log the denied access (hard)

Change `get_note` so that it prints one log line when access is not granted, in the form
`WARN note not available: user="ana" id=2`. Use a format that makes log injection impossible.

<details>
<summary>Hint</summary>

Put the result of `find` in a variable, check `is_none()`, print, and return the variable. Use `{user:?}`.

</details>

<details>
<summary>Answer</summary>

```rust
fn get_note<'a>(notes: &'a [Note], user: &str, id: u32) -> Option<&'a Note> {
    let found = notes.iter().find(|n| n.id == id && n.owner == user);
    if found.is_none() {
        println!("WARN note not available: user={user:?} id={id}");
    }
    found
}
```

The output of `main` then contains `WARN note not available: user="ana" id=2` before the line `checked:   None`.
Why it works: the user name goes through `{:?}`, so even a name with a newline stays on one line; the id is a
number and cannot contain a newline.

</details>

## Quiz

1. What is the OWASP Top 10, and what is it not?

<details>
<summary>Answer</summary>

A list of the ten most serious categories of web application risk, from real data, used for awareness. It
is not a complete list of all bugs and not a complete checklist.

</details>

2. What is an IDOR and how do you prevent it?

<details>
<summary>Answer</summary>

The program uses an identifier from the request without checking that the current user may access that
object. Check ownership or permission on the server for every request and every object.

</details>

3. What does "fail closed" mean?

<details>
<summary>Answer</summary>

When a security check cannot decide (because of an error), it refuses the action.

</details>

4. How can a user name forge a log line, and what stops it?

<details>
<summary>Answer</summary>

A newline in the name starts a new line in the log. Printing the value with `{:?}` (or another escaping
step) writes the newline as `\n`, so one event stays on one line.

</details>

## Summary

- The OWASP Top 10 lists categories of risk; it is a map, not a full checklist.
- 2025 order: Broken Access Control, Security Misconfiguration, Supply Chain Failures, Cryptographic Failures, Injection, Insecure Design, Authentication Failures, Integrity Failures, Logging and Alerting Failures, Mishandling of Exceptional Conditions.
- Check access on the server for every request and every object, and deny by default.
- Make the default configuration safe and refuse to start with an unsafe one.
- Security checks fail closed; never `unwrap` outside input.
- Log security events, but never secrets and never raw input.
- Each category points to a lesson of this phase where you learn the defense.

## Sources and further reading

- [OWASP Top 10:2025](https://top10.owasp.org/2025) — the official list with a page for each category.
- [OWASP Top 10:2025 — Introduction](https://top10.owasp.org/2025/0x00_2025-Introduction/) — how the list is made and what changed since 2021.
- [OWASP Cheat Sheet: Insecure Direct Object Reference Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Insecure_Direct_Object_Reference_Prevention_Cheat_Sheet.html) — IDOR in detail.
- [OWASP Cheat Sheet: Authorization](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) — the rules for access control, including deny by default.
- [OWASP Cheat Sheet: Error Handling](https://cheatsheetseries.owasp.org/cheatsheets/Error_Handling_Cheat_Sheet.html) — what to show users and what to log.
- [OWASP Cheat Sheet: Logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html) — what to record and how to avoid log injection.
- [OWASP Cheat Sheet: SSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html) — defenses against Server-Side Request Forgery.

---

[🏠 Start](../START-HERE.md) · [← 10.2 Threat modeling with STRIDE](02-threat-modeling-with-stride.md) · [📋 Phase 10](README.md) · [10.4 Input validation: never trust input →](04-input-validation.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
