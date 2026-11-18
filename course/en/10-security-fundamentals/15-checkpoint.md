<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.14 Project: an encrypted secrets vault](14-project-encrypted-secrets-vault.md) · [📋 Phase 10](README.md) · [Phase 11 — Linux, the shell and containers →](../11-linux-shell-and-containers/README.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/15-checkpoint.md) <!-- nav -->

# Lesson 10.15 — Checkpoint

> **Phase 10 — Security fundamentals** · Lesson 15 of 15 · ⏱ about 105 minutes

## What you will learn

This lesson teaches nothing new. It checks that you can use what Phase 10 taught, and it shows you where to go back if something is still weak. You will:

- Answer 18 questions about the security mindset, threat modeling, the OWASP (Open Worldwide Application Security Project) Top 10, input validation, injection, passwords, authorization, secrets, cryptography, TLS (Transport Layer Security), memory safety and supply-chain security.
- Solve 4 practical tasks that mix several lessons. Three of them have tests.
- Find the lessons you should repeat, with direct links.

## Before you start

- You finished [Lesson 10.14 — Project: an encrypted secrets vault](14-project-encrypted-secrets-vault.md). Your `vault` works, its tests pass, and it is committed.
- Do the quiz **closed book**: no lessons, no compiler, no search. Write your answer down first, then open the hidden answer. A wrong answer found now is cheap; the same mistake found in a real system is not.
- For the practical tasks you may use the compiler and the lessons, but try first without hints. As always in this phase, work only on your own code and machines.
- Create one package for the three coding tasks: `cargo new --lib code/p10-checkpoint`, then `cargo add sha2@0.11 hex@0.4 subtle@2.6 getrandom@0.4 argon2@0.6`. Task 1 goes into `src/session.rs`, Task 2 into `src/record.rs`, Task 3 into `src/password.rs`, and `src/lib.rs` contains `pub mod password; pub mod record; pub mod session;`.
- Your learning repository is clean (`git status` in `~/projects/rust`). Work on a branch: `git switch -c lesson/10-15-checkpoint`.

## New words

| Word | Meaning |
|---|---|
| **checkpoint** | A pause at the end of a phase to test yourself before you go on. |
| **closed book** | Answering from memory only, without looking anything up. |
| **security review** | Reading code or a design with the question "how could this be abused?" instead of "does this work?". |

## Self-test quiz

Mark one point for each answer that is **correct and complete**. Be honest: nobody checks but you.

### Mindset, threats and the OWASP Top 10 (Lessons 10.1 to 10.3)

1. What do the letters of the CIA triad stand for? Give one example of a bug that breaks each.

<details>
<summary>Answer</summary>

Confidentiality (a secret in a log file), Integrity (an integer overflow that changes an amount; a modified file that is accepted) and Availability (a one-byte packet that crashes the service). Any correct example counts. (Lesson 10.1.)

</details>

2. What does STRIDE stand for, and when in a project do you do threat modeling?

<details>
<summary>Answer</summary>

Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege. You do it early, on the design, before the code exists, and update it when the design changes. (Lesson 10.2.)

</details>

3. Which category tops the OWASP Top 10, and what are the two habits that prevent most of it?

<details>
<summary>Answer</summary>

Broken Access Control. Check access on the **server**, for every request and every object, and deny by default. (Lesson 10.3.)

</details>

4. What does "fail closed" mean? Give an example.

<details>
<summary>Answer</summary>

When a security check cannot decide or breaks (an error, a missing setting), the answer is "deny", not "allow". Example: if the permission lookup fails, the request is rejected. (Lessons 10.1 and 10.3.)

</details>

### Input and injection (Lessons 10.4 and 10.5)

5. Why is an allow-list better than a deny-list?

<details>
<summary>Answer</summary>

An allow-list accepts only what you know is valid, so everything you did not think of is rejected. A deny-list rejects only what you remembered to list, and attackers find what you forgot. (Lesson 10.4.)

</details>

6. A function builds `format!("SELECT * FROM users WHERE name = '{name}'")`. What is the bug, and what is the fix?

<details>
<summary>Answer</summary>

SQL (Structured Query Language) injection: the data becomes part of the code. Use a parameterized query (`?` with bound values) so data and code travel separately. (Lesson 10.5.)

</details>

7. How do you run an external program with a user-supplied file name safely in Rust?

<details>
<summary>Answer</summary>

Do not use a shell. Use `Command::new("program").arg(file_name)` so the name is one argument, and add `--` or reject names that start with `-` to prevent argument injection. (Lesson 10.5.)

</details>

### Passwords, authorization and secrets (Lessons 10.6 to 10.8)

8. How should a server store passwords, and why is `SHA-256(password)` wrong?

<details>
<summary>Answer</summary>

As an Argon2id hash with a random salt, stored as one PHC (Password Hashing Competition format) string. SHA-256 is fast, so a thief with the database can try billions of guesses per second. (Lessons 10.6 and 10.9.)

</details>

9. A login returns "unknown user" for a missing account and "wrong password" for a bad password. What is wrong?

<details>
<summary>Answer</summary>

It tells an attacker which user names exist (user enumeration). Return the same message (and take about the same time) in both cases. (Lesson 10.6.)

</details>

10. Name three properties of a good session token.

<details>
<summary>Answer</summary>

At least 128 random bits from the operating system's CSPRNG; the server stores only its hash and has the state, with idle and absolute timeouts; it is rotated at login and deleted on the server at logout. In a cookie: `HttpOnly`, `Secure`, `SameSite`. (Lesson 10.7.)

</details>

11. A secret was committed to Git by mistake and pushed. What is the first thing you do?

<details>
<summary>Answer</summary>

Revoke it and issue a new one: assume it is already compromised. Cleaning the history comes later and does not make the old value safe again. (Lesson 10.8.)

</details>

### Cryptography and TLS (Lessons 10.9 to 10.11)

12. What is the difference between a hash, a MAC and a digital signature?

<details>
<summary>Answer</summary>

A hash has no key: it detects accidents only. A MAC uses a shared secret key: only key holders can create or check a valid tag. A signature uses a private key to sign and a public key to verify: anyone can check, only the owner can sign. (Lessons 10.9 and 10.10.)

</details>

13. Why must you compare a MAC tag or a token in constant time?

<details>
<summary>Answer</summary>

A normal comparison stops at the first different byte, so its time reveals how many bytes were right. An attacker can use that to guess the secret byte by byte. Use `verify_slice` or `subtle::ConstantTimeEq`. (Lesson 10.9.)

</details>

14. What is a nonce, is it secret, and what happens if you reuse it with the same key?

<details>
<summary>Answer</summary>

A number that must be unique for each message under one key. It is not secret (it is stored next to the ciphertext). Reusing it lets an attacker XOR two ciphertexts and learn the XOR of the plaintexts, and for some ciphers it breaks authentication too. (Lesson 10.10.)

</details>

15. Which four checks does a TLS client make on a server certificate? What does `curl -k` do?

<details>
<summary>Answer</summary>

The chain of signatures ends in a trusted root; every certificate is within its validity dates; the host name is in the SAN; the certificate may be used as a server certificate. `-k` turns the verification off, which removes the protection against a man-in-the-middle. (Lesson 10.11.)

</details>

### Memory, supply chain and the project (Lessons 10.12 to 10.14)

16. Name three security problems that safe Rust does **not** prevent.

<details>
<summary>Answer</summary>

Any three of: panics on bad input (denial of service), huge allocations from announced sizes, stack overflow from deep recursion, wrapped integer arithmetic in release builds, logic bugs (injection, missing authorization), bugs in `unsafe` code or in dependencies. (Lesson 10.12.)

</details>

17. What is the difference between `cargo audit` and `cargo deny`?

<details>
<summary>Answer</summary>

`cargo audit` looks up the crates of `Cargo.lock` in the RustSec advisory database (vulnerabilities, unmaintained, unsound). `cargo deny` checks four policies: advisories, licenses, bans and sources. (Lesson 10.13.)

</details>

18. In the vault, why is the file header passed as associated data, and why is there one error for "wrong password" and "modified file"?

<details>
<summary>Answer</summary>

The header (version, KDF parameters, salt) must be covered by the tag, or an attacker could change it undetected. The tag check fails the same way in both cases and cannot tell them apart; a more specific message would only be a guess. (Lesson 10.14.)

</details>

**Score:** 16 to 18 points: you are ready for the next phase. 12 to 15: repeat the lessons of the questions you missed (see the table at the end). Fewer than 12: repeat the phase from Lesson 10.6 on, redo the exercises, and take the quiz again in a few days.

## Practical tasks

For Tasks 1 to 3 put the code in the files named above, run `cargo test`, and commit each task.

### Task 1 — Session tokens (Lessons 10.7, 10.9)

In `src/session.rs` write three functions:

- `pub fn new_token() -> String`: 32 random bytes from the operating system as 64 hex characters;
- `pub fn token_hash(token: &str) -> String`: the SHA-256 of the token in hex. This is what the server stores: **never the token itself**;
- `pub fn check(stored_hash: &str, presented: &str) -> bool`: hashes the presented token and compares with the stored hash in **constant time**.

Write tests for: the shape of a token (length, hex digits) and that two tokens differ; that the stored hash is not the token; that `check` accepts the right token and rejects another token, an empty token and an empty stored hash.

<details>
<summary>Hint</summary>

`getrandom::fill`, `hex::encode`, `Sha256::digest`, and `subtle::ConstantTimeEq::ct_eq(...)` with `.into()` to get a `bool`. Compare the **hashes** (both have the same length), not the raw tokens.

</details>

<details>
<summary>Answer</summary>

```rust
use sha2::{Digest, Sha256};
use subtle::ConstantTimeEq;

/// A new session token: 32 random bytes as 64 hex characters.
pub fn new_token() -> String {
    let mut bytes = [0u8; 32];
    getrandom::fill(&mut bytes).expect("the operating system has no random numbers");
    hex::encode(bytes)
}

/// What the server stores: the SHA-256 of the token, in hex. The token itself is never stored.
pub fn token_hash(token: &str) -> String {
    hex::encode(Sha256::digest(token.as_bytes()))
}

/// Does `presented` match the stored hash? Compares in constant time.
pub fn check(stored_hash: &str, presented: &str) -> bool {
    let presented_hash = token_hash(presented);
    stored_hash.as_bytes().ct_eq(presented_hash.as_bytes()).into()
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn tokens_have_the_right_shape_and_differ() {
        let (a, b) = (new_token(), new_token());
        assert_eq!(a.len(), 64);
        assert!(a.chars().all(|c| c.is_ascii_hexdigit()));
        assert_ne!(a, b);
    }

    #[test]
    fn the_stored_hash_is_not_the_token() {
        let t = new_token();
        assert_ne!(token_hash(&t), t);
        assert_eq!(token_hash(&t).len(), 64);
    }

    #[test]
    fn check_accepts_only_the_right_token() {
        let t = new_token();
        let stored = token_hash(&t);
        assert!(check(&stored, &t));
        assert!(!check(&stored, &new_token()));
        assert!(!check(&stored, ""));
        assert!(!check("", &t));
    }
}
```

Why it works: a session token is high-entropy random data, so a fast hash is enough for storing it (unlike a password): if the table leaks, the attacker still cannot go back from the hash to a token that works. `ct_eq` on slices of different lengths simply returns "not equal", so the empty stored hash is rejected.

</details>

### Task 2 — A parser that never panics (Lessons 10.4, 10.12)

In `src/record.rs` write `pub fn parse_record(input: &[u8]) -> Result<(Record, usize), ParseError>` for a binary record: one byte `name_len`, then `name_len` bytes of name, then two bytes `value_len` (big-endian), then `value_len` bytes of value. Rules: the name has 1 to 32 characters from `a-z`, `0-9` and `_`; the value is at most 1024 bytes. It returns the record and the number of bytes used. `Record` has `name: String` and `value: Vec<u8>`; `ParseError` has `Truncated`, `BadName` and `ValueTooLarge`.

Tests: a good record; **every prefix** of a good record is `Truncated` (never a panic); an empty name, a name over 32 bytes and an upper-case name are `BadName`; an announced value length of 65,535 is `ValueTooLarge`; 200 random inputs of different lengths never panic.

<details>
<summary>Hint</summary>

No `[i]` indexing: use `split_first`, `get(..n)`, and `ok_or(ParseError::Truncated)?`. Check the limits **before** you slice. Derive `Debug` and `PartialEq` on both types so that `assert_eq!` works.

</details>

<details>
<summary>Answer</summary>

```rust
/// A record: name_len (1 byte) | name | value_len (2 bytes, big-endian) | value
#[derive(Debug, PartialEq)]
pub struct Record {
    pub name: String,
    pub value: Vec<u8>,
}

#[derive(Debug, PartialEq)]
pub enum ParseError {
    Truncated,
    BadName,
    ValueTooLarge,
}

pub const MAX_NAME: usize = 32;
pub const MAX_VALUE: usize = 1024;

/// Parses one record from the start of `input`. Returns the record and the number of bytes used.
/// Never panics, whatever the input.
pub fn parse_record(input: &[u8]) -> Result<(Record, usize), ParseError> {
    let (&name_len, rest) = input.split_first().ok_or(ParseError::Truncated)?;
    let name_len = usize::from(name_len);
    if name_len == 0 || name_len > MAX_NAME {
        return Err(ParseError::BadName);
    }
    let name_bytes = rest.get(..name_len).ok_or(ParseError::Truncated)?;
    if !name_bytes.iter().all(|b| b.is_ascii_lowercase() || b.is_ascii_digit() || *b == b'_') {
        return Err(ParseError::BadName);
    }
    let rest = &rest[name_len..];
    let len_bytes = rest.get(..2).ok_or(ParseError::Truncated)?;
    let value_len = usize::from(u16::from_be_bytes([len_bytes[0], len_bytes[1]]));
    if value_len > MAX_VALUE {
        return Err(ParseError::ValueTooLarge);
    }
    let value = rest.get(2..2 + value_len).ok_or(ParseError::Truncated)?;
    let name = String::from_utf8(name_bytes.to_vec()).map_err(|_| ParseError::BadName)?;
    Ok((Record { name, value: value.to_vec() }, 1 + name_len + 2 + value_len))
}

#[cfg(test)]
mod tests {
    use super::*;

    fn good() -> Vec<u8> {
        let mut v = vec![4];
        v.extend_from_slice(b"user");
        v.extend_from_slice(&[0, 5]);
        v.extend_from_slice(b"alice");
        v
    }

    #[test]
    fn a_good_record_is_parsed() {
        let (record, used) = parse_record(&good()).unwrap();
        assert_eq!(record, Record { name: "user".into(), value: b"alice".to_vec() });
        assert_eq!(used, 12);
    }

    #[test]
    fn every_prefix_of_a_good_record_is_truncated_not_a_panic() {
        let bytes = good();
        for n in 0..bytes.len() {
            assert_eq!(parse_record(&bytes[..n]), Err(ParseError::Truncated), "prefix {n}");
        }
    }

    #[test]
    fn bad_names_and_huge_values_are_rejected() {
        assert_eq!(parse_record(&[0]), Err(ParseError::BadName));
        assert_eq!(parse_record(&[255; 300]), Err(ParseError::BadName));
        assert_eq!(parse_record(&[2, b'A', b'b', 0, 0]), Err(ParseError::BadName));
        // announces 65,535 bytes of value but delivers none
        assert_eq!(parse_record(&[1, b'a', 0xff, 0xff]), Err(ParseError::ValueTooLarge));
    }

    #[test]
    fn random_bytes_never_panic() {
        for len in 0..200 {
            let mut buf = vec![0u8; len];
            getrandom::fill(&mut buf).unwrap();
            let _ = parse_record(&buf);
        }
    }
}
```

Why it works: every access is a `get` or a `split_first` that turns "not enough bytes" into `Truncated`. The limits for the name and the value are checked before slicing, so an announced length of 65,535 never causes an allocation. The one index `rest[name_len..]` is safe because `get(..name_len)` just proved that `rest` has at least `name_len` bytes. The tests include the boundary cases and random garbage.

</details>

### Task 3 — Password hashing wrappers (Lessons 10.6, 10.9)

In `src/password.rs` write `pub fn hash_password(password: &str) -> String` (Argon2id with default settings and a fresh random salt; the result is the PHC string) and `pub fn verify_password(stored: &str, password: &str) -> bool`. A stored text that is not a valid hash must give `false`, not a panic. Tests: the right password verifies and a wrong one does not; the hash starts with `$argon2id$`; the same password gives two different hashes; the malformed texts `"garbage"`, `""` and `"$argon2id$v=19$m=1,t=1,p=1$"` give `false`.

<details>
<summary>Hint</summary>

The same API (Application Programming Interface) as in Lesson 10.6: `Argon2::default().hash_password(bytes)`, `PasswordHash::new(text)` from `argon2::password_hash::phc`, and `verify_password`. Both traits `PasswordHasher` and `PasswordVerifier` must be imported.

</details>

<details>
<summary>Answer</summary>

```rust
use argon2::Argon2;
use argon2::password_hash::phc::PasswordHash;
use argon2::password_hash::{PasswordHasher, PasswordVerifier};

/// Hashes a password with Argon2id and a fresh random salt. Returns the PHC string.
pub fn hash_password(password: &str) -> String {
    Argon2::default().hash_password(password.as_bytes()).expect("hashing failed").to_string()
}

/// True only if `password` matches `stored`. A malformed `stored` text is `false`, not a panic.
pub fn verify_password(stored: &str, password: &str) -> bool {
    match PasswordHash::new(stored) {
        Ok(parsed) => Argon2::default().verify_password(password.as_bytes(), &parsed).is_ok(),
        Err(_) => false,
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn right_password_verifies_and_wrong_does_not() {
        let stored = hash_password("correct horse battery");
        assert!(stored.starts_with("$argon2id$"));
        assert!(verify_password(&stored, "correct horse battery"));
        assert!(!verify_password(&stored, "correct horse batterY"));
    }

    #[test]
    fn the_same_password_gives_different_hashes() {
        assert_ne!(hash_password("pw"), hash_password("pw"));
    }

    #[test]
    fn a_malformed_stored_hash_is_false_not_a_panic() {
        assert!(!verify_password("garbage", "pw"));
        assert!(!verify_password("", "pw"));
        assert!(!verify_password("$argon2id$v=19$m=1,t=1,p=1$", "pw"));
    }
}
```

Why it works: the stored text is data from a database, so it is untrusted too: a parse error becomes `false`. `verify_password` compares in constant time inside the library. Together, the three tasks give `test result: ok. 10 passed` (3 + 4 + 3 tests).

</details>

### Task 4 — Review your own vault (no new code)

Review `code/p10-vault` as an attacker would. Write the answers in `code/p10-vault/SECURITY.md` and commit it.

1. Run `cargo audit` and `cargo deny check`. Paste the last line of each. Fix every vulnerability, or write down why you ignore it and until when.
2. List every place where a secret (password, key, plaintext, secret value) is held in memory. For each: is it in a `Zeroizing` (or wiped by hand)? Does any `Debug`, `println!` or error message print it?
3. Try five hostile files: empty, one byte, 200 random bytes, a valid vault with one flipped byte, and a vault with `m_cost` set to `0xFFFFFFFF` in the header. What does each print, and what is the exit code?
4. Write the threat model in six lines: three things the vault protects against and three it does not.
5. Name the single change that would improve the security of the vault most, and why you did not make it yet.

<details>
<summary>Hint</summary>

For item 3 use `printf` and `dd`, or write the files from a small test. The commands from M4 and M5 of Lesson 10.14 give you the file layout (the `m_cost` field is bytes 5 to 8).

</details>

<details>
<summary>Answer</summary>

There is no single right text, but a good review contains:

- a clean `cargo audit` and `cargo deny check` (`advisories ok, bans ok, licenses ok, sources ok`), or each exception with a reason and a date;
- a table of secrets in memory: the master password, the derived key, the decrypted JSON (JavaScript Object Notation) bytes and the value read from standard input are all in `Zeroizing`; `Vault` wipes its values in `Drop`; no `derive(Debug)` on a type with secrets;
- for the five hostile files, an `error:` line and exit code `1` each, and no `panicked`; the absurd `m_cost` is rejected as invalid parameters, **before** Argon2 runs;
- a threat model like the table in Lesson 10.14;
- an honest "next improvement", for example: a key file as a second factor, `mlock`, or rate limiting of password attempts through a slower `t_cost`.

Why it works: this is how a real security review works: you attack your own work, write down the evidence, and record what you accept.

</details>

## If you got something wrong

| Topic | Go back to |
|---|---|
| CIA triad, attack surface, security principles | [10.1 The security mindset and the CIA triad](01-security-mindset-and-the-cia-triad.md) |
| STRIDE, risk rating, attack trees | [10.2 Threat modeling with STRIDE](02-threat-modeling-with-stride.md) |
| Access control, misconfiguration, fail closed | [10.3 The OWASP Top 10](03-the-owasp-top-10.md) |
| Allow-lists, parsing into types, limits | [10.4 Input validation](04-input-validation.md) |
| SQL, command and path injection | [10.5 Injection attacks](05-injection-attacks.md) |
| Argon2id, salts, login errors | [10.6 Authentication and password hashing](06-authentication-and-password-hashing.md) |
| Authorization, sessions, cookies, JWT | [10.7 Authorization, sessions and tokens](07-authorization-sessions-and-tokens.md) |
| Secrets, `secrecy`, `zeroize`, leaks | [10.8 Secrets management](08-secrets-management.md) |
| Hashes, HMAC, constant-time, randomness | [10.9 Cryptography I: hashes, MACs and randomness](09-cryptography-hashes-macs-and-randomness.md) |
| AEAD, nonces, signatures | [10.10 Cryptography II: encryption and signatures](10-cryptography-encryption-and-signatures.md) |
| TLS, certificates, `curl -v`, `openssl s_client` | [10.11 TLS and certificates](11-tls-and-certificates.md) |
| Panics as DoS, limits, `unsafe` review, Miri, ASan | [10.12 Memory safety and reviewing unsafe code](12-memory-safety-and-reviewing-unsafe.md) |
| `cargo audit`, `cargo deny`, dependency habits | [10.13 Supply-chain security](13-supply-chain-security.md) |
| File format, Argon2 + AEAD together, hardening | [10.14 Project: an encrypted secrets vault](14-project-encrypted-secrets-vault.md) |

## What's next

You finished Phase 10. You can now think like an attacker about your own designs, validate input, avoid injection, store passwords and secrets, use the standard cryptographic building blocks, read a TLS connection, review `unsafe` code and check your dependencies. Before you go on:

1. Merge your checkpoint branch (`git switch main && git merge lesson/10-15-checkpoint`) and delete finished branches you no longer need.
2. Keep `SECURITY.md` and the threat model of the vault: you will want to update them when you deploy it.
3. Add a few lines to your learning notes: which attacks surprised you, and which habit you will start using now (for example, `cargo audit` in CI, Continuous Integration).

In **[Phase 11 — Linux, the shell and containers](../11-linux-shell-and-containers/README.md)** you will move to the operating system itself: a Linux virtual machine on your Mac, the file system, text tools, users and permissions (the `0600` of the vault file will make full sense), processes, services, `bash`, SSH, networking commands and Docker. Many of the security ideas of this phase return there: least privilege, secrets in containers, and the safe handling of untrusted input in scripts.

## Sources and further reading

- [OWASP Top 10](https://owasp.org/www-project-top-ten/) — the list of risk categories behind Lesson 10.3.
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) — short, practical guides for each topic of this phase.
- [The Rust Secure Code Working Group](https://github.com/rust-secure-code/wg) — guides, tools and advisories for secure Rust.
- [RustSec advisory database](https://rustsec.org/) — vulnerabilities and unmaintained crates in the Rust ecosystem.
- [Cryptographic Right Answers](https://www.latacora.com/blog/2018/04/03/cryptographic-right-answers/) — a short article that says which primitive to choose for which job.
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) — the official guide to `unsafe` Rust.

---

[🏠 Start](../START-HERE.md) · [← 10.14 Project: an encrypted secrets vault](14-project-encrypted-secrets-vault.md) · [📋 Phase 10](README.md) · [Phase 11 — Linux, the shell and containers →](../11-linux-shell-and-containers/README.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
