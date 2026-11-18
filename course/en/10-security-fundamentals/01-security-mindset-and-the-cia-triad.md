<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← Phase 10 — Security fundamentals](README.md) · [📋 Phase 10](README.md) · [10.2 Threat modeling with STRIDE →](02-threat-modeling-with-stride.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/01-security-mindset-and-the-cia-triad.md) <!-- nav -->

# Lesson 10.1 — The security mindset and the CIA triad

> **Phase 10 — Security fundamentals** · Lesson 1 of 15 · ⏱ about 120 minutes

## What you will learn

- The ground rules of security work: attack only what you own, and what responsible disclosure means.
- How to think like an attacker: assets, attackers, attack surface and trust boundaries.
- The CIA triad (confidentiality, integrity, availability), with a small Rust program for each part.
- The core principles that every later lesson of this phase uses: least privilege, defense in depth, secure defaults, never trust input.
- Why safe Rust is a great start for security but not the whole story.

## Before you start

- You finished [Phase 09](../09-testing-debugging-benchmarking/README.md), so you can write and run tests.
- You know integer overflow from [Lesson 02.3 — Operators and expressions](../02-programming-basics/03-operators-and-expressions.md) and error handling from [Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md).
- You remember that a secret committed to Git stays in the history ([Lesson 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md)).
- **Where the code goes in this phase.** Each lesson has its own package in `~/projects/rust/code/p10-<topic>`. Small examples are separate programs inside `src/bin/`, and you run one with `cargo run --bin <name>`. Outputs in the lesson show exactly that file. The phase project lives in `code/p10-vault`.

## New words

| Word | Meaning |
|---|---|
| **security** | Keeping a system working correctly and its data safe **even when someone is trying to make it fail**. |
| **asset** | Anything worth protecting: data (passwords, messages), a service that must stay online, a server, money, a reputation. |
| **attacker** (or *adversary*) | A person or a program that tries to make the system do something its owner did not intend. |
| **vulnerability** | A weakness in a system (a bug, a bad setting, a missing check) that an attacker can use. |
| **exploit** | A way of using a vulnerability, or the program or input that does it. To *exploit* is to use it. |
| **threat** | Something bad that could happen to an asset, for example "someone reads all user passwords". |
| **risk** | How bad a threat is for you: roughly *how likely* it is multiplied by *how much damage* it does. |
| **attack surface** | The sum of all the places where an attacker can reach your program: every input, port, file, command-line argument, dependency. |
| **trust boundary** | A line where data passes from a place you do not control (a user, the network, a file) into code that you do control. |
| **CIA triad** | Three goals of security: **C**onfidentiality, **I**ntegrity, **A**vailability. |
| **confidentiality** | Only the people who are allowed can read the data. |
| **integrity** | Data and programs are changed only in allowed ways, by allowed people. |
| **availability** | The system works and answers when authorised users need it. |
| **DoS** (Denial of Service) | An attack that makes a system unavailable, often by making it crash or by using up its memory, processor time or network. |
| **least privilege** | Give every person and program only the permissions it needs, nothing more. |
| **defense in depth** | Use several independent protections, so that when one fails the next one still stops the attack. |
| **responsible disclosure** | Telling the owner privately about a vulnerability you found, and giving them time to fix it before you speak about it publicly. |
| **CVE** (Common Vulnerabilities and Exposures) | A public identifier such as `CVE-2026-12345` for one known vulnerability. |

## 1. Ground rules: ethics and the law

This phase teaches you how attacks work, because you cannot defend against what you do not understand.
That knowledge comes with strict rules:

1. **Attack only what you own.** You may test your own programs, your own computer, and containers that
   you started yourself. You may not test somebody else's website, server, network or account, even
   "just to see". In many countries this is a crime even when you break nothing. Permission must be
   written and must come from the owner.
2. **Use harmless payloads.** A *payload* is the data an attack sends. In this course the payload only
   prints a word such as `INJECTED`. It never deletes, steals or damages anything.
3. **Never use real secrets in examples.** All passwords and keys in the lessons are made-up values.
4. **If you find a vulnerability in someone else's software, report it privately.** This is
   *responsible disclosure*: write to the security contact of the project (look for a file named
   `SECURITY.md`, a "security" page, or an address like `security@…`), describe the problem and how to
   reproduce it, and give the owners reasonable time (often 90 days) to fix it before you publish
   anything. Publishing at once puts every user at risk. The Rust project has such a page, see the
   sources at the end. When the fix is released, the problem usually gets a **CVE** number, and for
   Rust crates an entry in the RustSec database, which you will use in [Lesson 10.13](13-supply-chain-security.md).

> 🔒 **Security:** the same skills that protect systems can harm them. Treat every technique in this
> phase as something you may use only on your own machine.

## 2. Think like an attacker

When you write a program you think about the **happy path**: the user types a number, the program
prints a result. An attacker thinks about the **unhappy paths**: what if the number is negative, huge,
missing, or a million characters long? What if the file is a link to another file? What if the same request
arrives a thousand times per second?

Security work starts with four questions:

1. **What are we protecting?** The assets. A notes app protects the text of the notes. A bank also
   protects the balances and the fact that it can answer at all.
2. **Who might attack, and why?** A curious user who changes a number in a web address. A criminal who
   wants money. A bot that tries every server on the internet. An insider (an employee or a
   contractor) who already has some access. Each has different skills and different goals.
3. **Where can they reach us?** This is the **attack surface**. Every open port, every command-line
   argument, every file you read, every library you use is a place where something unexpected can
   come in. A smaller surface is easier to defend.
4. **Where does untrusted data enter?** Draw a line around the code you control. Everything that
   crosses the line from the outside is **untrusted** until you have checked it. This line is the
   **trust boundary**.

```text
   untrusted world                  trust boundary                 your code
 ┌────────────────────┐                  │                  ┌─────────────────────┐
 │ users, network,    │ ── bytes, text ──┼──►  check! ──►   │ logic, database,    │
 │ files, env vars,   │                  │                  │ files you protect   │
 │ other programs     │                  │                  │                     │
 └────────────────────┘                  │                  └─────────────────────┘
```

> 💡 **Tip:** a good habit while reading any code is to ask "where does this value come from, and who
> controls it?" If the answer is "someone outside", the next question is "what happens if it is the worst
> possible value?"

## 3. The CIA triad

Almost every security problem breaks one of three goals. The goals are easy to remember as **CIA**.
This has nothing to do with the intelligence agency; it is just the three first letters.

| Goal | The question | An attack on it | A real-life example |
|---|---|---|---|
| **C**onfidentiality | Can only the right people read it? | Stealing or leaking data | A password appears in a log file |
| **I**ntegrity | Is it changed only in allowed ways? | Changing data or code | A balance silently becomes a wrong number |
| **A**vailability | Does it work when needed? | Making it unusable (DoS) | One request makes the server crash |

Let us see one small, real bug for each goal. Create the package for this lesson:

```bash
cd ~/projects/rust
cargo new code/p10-cia
cd code/p10-cia
mkdir src/bin
rm src/main.rs
```

Each program below goes into its own file in `src/bin/`. (If you keep `src/main.rs` the commands
still work; we delete it only to keep the folder tidy.)

### 3.1 Confidentiality: the secret in the log

A very common leak is not a clever attack at all. A developer adds a debug line, and the line prints
a secret into a log file that many people can read. Save this as `src/bin/leak.rs`:

```rust
#[derive(Debug)]
struct Account {
    name: String,
    password: String,
}

fn main() {
    let account = Account {
        name: String::from("ana"),
        password: String::from("correct-horse-battery"),
    };
    println!("checking user {}", account.name);
    println!("password length is {}", account.password.len());
    // A "harmless" debug line that someone left in the code:
    println!("login attempt: {:?}", account);
}
```

Run it with `cargo run --bin leak`.

**What happens:**

1. `#[derive(Debug)]` asks the compiler to write the `{:?}` formatting for `Account` automatically. The
   generated code prints **every field**.
2. The first two `println!` lines use single fields in a normal way.
3. The last line prints the whole struct, and the password is part of it.

```text
checking user ana
password length is 21
login attempt: Account { name: "ana", password: "correct-horse-battery" }
```

Anyone who can read the log now knows the password. The fix is to write `Debug` by hand and hide the
field. Save this as `src/bin/leak_fixed.rs`:

```rust
use std::fmt;

struct Account {
    name: String,
    password: String,
}

impl fmt::Debug for Account {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        f.debug_struct("Account")
            .field("name", &self.name)
            .field("password", &"[hidden]")
            .finish()
    }
}

fn main() {
    let account = Account {
        name: String::from("ana"),
        password: String::from("correct-horse-battery"),
    };
    println!("login attempt: {:?}", account);
    println!("the password has {} characters", account.password.len());
}
```

**What happens:**

1. `impl fmt::Debug for Account` replaces the generated code with ours.
2. `f.debug_struct("Account")` starts the usual `Account { … }` style of output, and `.field(name, value)`
   adds one field. We print the text `[hidden]` instead of the real password.
3. `.finish()` closes the output.

```text
login attempt: Account { name: "ana", password: "[hidden]" }
the password has 21 characters
```

Lesson [10.8](08-secrets-management.md) shows ready-made types (`secrecy`) that do this for you.

### 3.2 Integrity: the balance that wraps around

Integrity means data is correct. A classic way to damage it is **integer overflow**: a number goes
past the largest value its type can hold. You met this in Lesson 02.3. Here is a bank-like function in
`src/bin/overflow.rs`:

```rust
fn deposit(balance: u32, amount: u32) -> u32 {
    balance + amount
}

fn main() {
    let balance: u32 = 4_000_000_000;
    let amount: u32 = 500_000_000;
    let new_balance = deposit(balance, amount);
    println!("balance after deposit: {new_balance}");
}
```

A `u32` holds numbers up to 4 294 967 295, and the sum is 4 500 000 000. Run it twice, first in the
default **debug** mode and then in **release** mode (the optimised build you use for the finished program):

```bash
cargo run --bin overflow
cargo run --release --bin overflow
```

```text
thread 'main' (4607053) panicked at src/bin/overflow.rs:2:5:
attempt to add with overflow
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
```

```text
balance after deposit: 205032704
```

**What happens:**

1. In debug mode Rust checks every `+` and panics when the result does not fit. That is an
   **availability** problem if the number came from an attacker: one request stops the program.
2. In release mode these checks are off by default. The result silently **wraps around** (counts
   again from zero): 4 500 000 000 − 4 294 967 296 = 205 032 704. The customer who deposited 500 million
   now has 205 million. That is an **integrity** failure, and nobody notices.

The fix is to make the decision explicit with `checked_add`, which returns `None` when the sum does not
fit. `src/bin/overflow_fixed.rs`:

```rust
fn deposit(balance: u32, amount: u32) -> Result<u32, String> {
    balance
        .checked_add(amount)
        .ok_or_else(|| String::from("deposit rejected: the balance would overflow"))
}

fn main() {
    let balance: u32 = 4_000_000_000;
    let amount: u32 = 500_000_000;
    match deposit(balance, amount) {
        Ok(new_balance) => println!("balance after deposit: {new_balance}"),
        Err(message) => println!("{message}"),
    }
}
```

```text
deposit rejected: the balance would overflow
```

> 🔒 **Security:** whenever a number comes from outside and you add, multiply or subtract it, use
> `checked_*` (returns `Option`), or `saturating_*` / `wrapping_*` when you really want that behaviour.
> The same check works in debug and release mode.

### 3.3 Availability: the line that never ends

Availability fails when one request can use up a resource: memory, processor time, disk or
connections. A program that reads "one line of input" with `read_line` waits until it sees a newline
character, however many bytes come first. An attacker who sends bytes without a newline makes your
program keep all of them in memory.

Save `src/bin/limit.rs`:

```rust
use std::io::{self, BufRead, BufReader, Read};

/// Reads one line, but never more than `limit` bytes.
fn read_line_limited<R: Read>(source: R, limit: u64) -> io::Result<Option<String>> {
    let mut reader = BufReader::new(source.take(limit + 1));
    let mut line = String::new();
    reader.read_line(&mut line)?;
    if line.len() as u64 > limit {
        return Ok(None); // too long: refuse it
    }
    Ok(Some(line))
}

fn main() -> io::Result<()> {
    // An endless stream of the letter 'a' with no end of line, cut at 200 MB for this demo.
    let attacker = io::repeat(b'a').take(200_000_000);
    let mut reader = BufReader::new(attacker);
    let mut line = String::new();
    reader.read_line(&mut line)?;
    println!("unbounded read: the program now holds {} bytes", line.len());
    drop(line);

    let attacker = io::repeat(b'a');
    match read_line_limited(attacker, 1024)? {
        Some(line) => println!("accepted {} bytes", line.len()),
        None => println!("bounded read: refused, the line is longer than 1024 bytes"),
    }
    Ok(())
}
```

**What happens:**

1. `io::repeat(b'a')` is a reader that produces the byte `a` forever. It plays the attacker. `.take(n)`
   (from the `Read` trait) wraps any reader so that it ends after `n` bytes.
2. The first part reads "a line" from 200 million `a` characters. There is no newline, so `read_line` keeps
   going and stores 200 MB (megabytes: one MB is a million bytes) in the `String`. With a real network connection and many clients this
   quickly exhausts the memory.
3. `read_line_limited` wraps the source in `.take(limit + 1)`. Reading stops after at most
   `limit + 1` bytes, even from an endless source. If we got more than `limit` bytes, we refuse the
   line. The `+ 1` is what lets us tell "exactly `limit` bytes" from "too long".
4. The second part uses a truly endless source and still finishes at once.

```text
unbounded read: the program now holds 200000000 bytes
bounded read: refused, the line is longer than 1024 bytes
```

> ⚡ **Performance:** every limit has a cost and a benefit. A limit on request size, on the number of
> items, on the time a request may take, protects availability. Choose limits that real users never
> reach but attackers cannot go far beyond.

### 3.4 Three goals, one trade-off

Most security measures help one goal and can hurt another. Encrypting a backup protects
confidentiality, but if you lose the key you lose availability for ever. Locking an account after three
wrong passwords protects confidentiality, but lets an attacker lock **other people** out on purpose
(an availability attack). Security design means choosing the balance for your asset.

## 4. Risk: how much should we worry?

You cannot make a system perfectly safe, and you do not have the time to defend against everything. So
security people estimate **risk** for each threat:

```text
risk = likelihood × impact
```

Use simple scales, for example 1 (low) to 3 (high). A bot that can guess a weak admin password:
likelihood 3, impact 3, risk 9, fix this today. A skilled attacker who needs physical access to your
laptop to read a recipe file: likelihood 1, impact 1, risk 1, you may accept it. Writing down what you
decided to accept is also part of the work. [Lesson 10.2](02-threat-modeling-with-stride.md) turns this
idea into a method.

## 5. Principles you will meet again and again

| Principle | In one sentence | Example |
|---|---|---|
| **Never trust input** | All data from outside is hostile until checked. | Limit the length of a line (Section 3.3); [Lesson 10.4](04-input-validation.md). |
| **Least privilege** | Give only the permissions that are needed. | A web server that can read its own folder but not the whole disk. |
| **Defense in depth** | Several independent layers. | Validate input, **and** use safe database calls, **and** run with a limited account. |
| **Secure by default** | The safest option is the default. | A new user has no admin rights; a missing setting means "deny". |
| **Fail closed** | When something goes wrong, refuse. | If the permission check cannot run, answer "no", not "yes". |
| **Keep it small and simple** | Less code and fewer features mean less attack surface. | Remove the options and dependencies that you do not need. |
| **Do not invent cryptography** | Use audited libraries, not your own scheme. | Phase 10 uses well-known crates; [Lesson 10.9](09-cryptography-hashes-macs-and-randomness.md). |
| **Open design** | Security must not depend on the attacker not knowing how it works. | A lock is good because the key is secret, not because the lock type is secret. |

> 📏 **Convention:** "security through obscurity" means relying on secrecy of the design (a hidden URL,
> Uniform Resource Locator, that is a web address; a secret algorithm) as the only protection. It is not enough. Secrecy of the **key** is fine; secrecy of
> the **method** is not a defense.

## 6. What Rust gives you, and what it does not

Safe Rust removes a whole family of vulnerabilities: reading or writing outside an array, using memory
after it was freed, and data races between threads. These bugs cause a large share of the serious
vulnerabilities found in programs written in languages without these checks. You learned why in
Phase 03 and in [Lesson 06.6](../06-rust-advanced/06-unsafe-rust-and-miri.md).

But Rust cannot know your *intentions*. It will happily compile a program that:

- prints a password into a log (Section 3.1),
- accepts any amount of data from the network (Section 3.3),
- gives every user the rights of an administrator,
- builds a command or a database query from raw user text ([Lesson 10.5](05-injection-attacks.md)),
- uses a weak or home-made encryption scheme.

These are **logic** and **design** bugs. The rest of the phase teaches you to avoid them.

## Common mistakes

> ⚠️ **Common mistake:** "My program is small and nobody knows about it, so nobody will attack it."
> Attacks are mostly automated. Bots scan the whole internet for open ports, weak passwords and known
> vulnerable versions, and they do not care how small you are.

> ⚠️ **Common mistake:** checking input only in the user interface (for example only in a web form).
> An attacker does not use your form; they send requests directly. Always check again on the side that
> you control.

> ⚠️ **Common mistake:** assuming debug and release builds behave the same. Overflow checks, `debug_assert!`
> and other safety nets exist only in debug mode by default. Test with `cargo test --release` as well
> when your code does arithmetic on outside numbers.

> ⚠️ **Common mistake:** treating "it did not crash" as "it is secure". A silent wrong answer (Section 3.2)
> is often worse than a crash.

## Try it yourself

1. Run all five programs of this lesson in order (`leak`, `leak_fixed`, `overflow` in debug and in
   release, `overflow_fixed`, `limit`) and compare your output with the lesson.
2. In `limit.rs`, change the limit from 1024 to 50 000 000. What changes? Which resource do you spend
   now? Write one sentence about how you would choose a good limit for a real program.
3. Take a small program from an earlier phase. List its inputs (arguments, files, standard input) and
   write down the worst value an attacker could send to each one.

## Exercises

### Exercise 1 — Name the broken goal (easy)

For each event say which goal of the CIA triad is mainly broken: (a) an employee copies the customer
list and sells it; (b) a script changes the prices in a database to zero; (c) a flood of requests makes
the shop website unreachable on a sale day; (d) an old backup disk with unencrypted data is lost in a
taxi.

<details>
<summary>Hint</summary>

Ask: was something *read*, *changed*, or *made unusable*?

</details>

<details>
<summary>Answer</summary>

(a) confidentiality, the data was read by someone not allowed. (b) integrity, data was changed in a way
that is not allowed. (c) availability, the service cannot be used. (d) confidentiality, the data can be
read by whoever finds the disk (encrypting the backup would protect it).

</details>

### Exercise 2 — Test the deposit function (medium)

Put the checked version of `deposit` into a library package and **write a test** that proves: a normal
deposit works, a deposit that makes the balance exactly `u32::MAX` works, and a deposit that goes one
past `u32::MAX` is rejected.

<details>
<summary>Hint</summary>

Create `cargo new --lib code/p10-cia-lib`. Put the function in `src/lib.rs` as `pub fn`, and the test in
`#[cfg(test)] mod tests`. `assert_eq!(deposit(10, 5), Ok(15))` works because `Result<u32, String>`
implements `PartialEq`.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn deposit(balance: u32, amount: u32) -> Result<u32, String> {
    balance
        .checked_add(amount)
        .ok_or_else(|| String::from("deposit rejected: the balance would overflow"))
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn deposit_works_and_rejects_overflow() {
        assert_eq!(deposit(10, 5), Ok(15));
        assert!(deposit(u32::MAX, 1).is_err());
        assert_eq!(deposit(u32::MAX - 1, 1), Ok(u32::MAX));
    }
}
```

Why it works: `checked_add` returns `None` exactly when the mathematical sum is larger than `u32::MAX`, in
debug **and** in release mode, so the test result does not depend on the build mode.

</details>

### Exercise 3 — Prove the limit is exact (hard)

Write tests for `read_line_limited` that prove: a line of exactly `limit` bytes is accepted, and a line
of `limit + 1` bytes is refused. Why does the function use `take(limit + 1)` and not `take(limit)`?

<details>
<summary>Hint</summary>

A `&[u8]` implements `Read`, so `"aaaa".as_bytes()` can be the source in a test. Think about what the
function would see if the input were `limit + 5` bytes and you had used `take(limit)`.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn limit_is_exact() {
        let exactly = "a".repeat(10);
        assert_eq!(
            read_line_limited(exactly.as_bytes(), 10).unwrap(),
            Some(exactly.clone())
        );
        let too_long = "a".repeat(11);
        assert_eq!(read_line_limited(too_long.as_bytes(), 10).unwrap(), None);
    }
}
```

Why it works: with `take(limit)` the reader would stop after exactly `limit` bytes for both inputs, and we
could not tell a line of exactly `limit` bytes from a longer one that was cut. Reading one extra byte
makes "too long" visible: if the line has more than `limit` bytes, the extra byte appears.

</details>

## Quiz

1. What does each letter of CIA stand for?

<details>
<summary>Answer</summary>

Confidentiality (only allowed people can read), Integrity (changes only in allowed ways), Availability
(the system works when needed).

</details>

2. What is the difference between a vulnerability and an exploit?

<details>
<summary>Answer</summary>

A vulnerability is the weakness itself (a bug or a missing check). An exploit is the way, or the input or
program, that uses the weakness.

</details>

3. Why can the same arithmetic bug break integrity in release mode and availability in debug mode?

<details>
<summary>Answer</summary>

In debug mode an overflow panics, so the program stops (availability). In release mode it wraps around
silently, so the data becomes wrong (integrity).

</details>

4. What is a trust boundary, and why is it useful?

<details>
<summary>Answer</summary>

It is the line between data you do not control and code you control. Checking data when it crosses the
line means the rest of the code can rely on checked values.

</details>

5. You find a serious vulnerability in an open-source library. What do you do?

<details>
<summary>Answer</summary>

Report it privately to the maintainers through their security contact, describe how to reproduce it,
and allow reasonable time to fix it before any public talk. Do not publish it at once and do not
exploit it on systems that you do not own.

</details>

6. Does safe Rust make a program secure? Why or why not?

<details>
<summary>Answer</summary>

It prevents memory-safety bugs (out-of-bounds access, use after free, data races) but not logic and design
bugs such as leaking secrets, missing permission checks, unlimited input or weak cryptography.

</details>

## Summary

- Attack only what you own, use harmless payloads, and report found vulnerabilities privately.
- Security is about the unhappy paths: assets, attackers, attack surface, trust boundaries.
- The CIA triad: confidentiality (no leaks), integrity (no wrong changes), availability (keeps working).
- Overflow, unbounded input and secrets in logs are small, real bugs that break these goals.
- Risk is likelihood times impact; decide what to fix first and write down what you accept.
- Principles: never trust input, least privilege, defense in depth, secure defaults, fail closed, keep it simple, no home-made cryptography.
- Safe Rust removes memory bugs, not logic bugs.

## Sources and further reading

- [Rust Security Policy](https://www.rust-lang.org/policies/security) — how the Rust project asks you to report a vulnerability, a real example of responsible disclosure.
- [OWASP Vulnerability Disclosure Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Vulnerability_Disclosure_Cheat_Sheet.html) — practical advice for researchers and for owners, from OWASP (the Open Worldwide Application Security Project, a non-profit security community).
- [NIST Computer Security Resource Center — glossary: confidentiality](https://csrc.nist.gov/glossary/term/confidentiality) — the formal definition of one of the three goals, and links to the other two, from NIST (the National Institute of Standards and Technology, the United States government agency that publishes security standards).
- [OWASP Top Ten](https://owasp.org/www-project-top-ten/) — the list of the most common web risks that [Lesson 10.3](03-the-owasp-top-10.md) explains.
- [The Rust standard library — trait `Read`](https://doc.rust-lang.org/std/io/trait.Read.html) — `take`, `repeat` and the other tools used in Section 3.3.
- [RustSec Advisory Database](https://rustsec.org/) — the public list of known vulnerabilities in Rust crates.

---

[🏠 Start](../START-HERE.md) · [← Phase 10 — Security fundamentals](README.md) · [📋 Phase 10](README.md) · [10.2 Threat modeling with STRIDE →](02-threat-modeling-with-stride.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
