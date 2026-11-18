<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.5 Injection attacks: SQL, commands and paths](05-injection-attacks.md) · [📋 Phase 10](README.md) · [10.7 Authorization, sessions and tokens →](07-authorization-sessions-and-tokens.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/06-authentication-and-password-hashing.md) <!-- nav -->

# Lesson 10.6 — Authentication and password hashing

> **Phase 10 — Security fundamentals** · Lesson 6 of 15 · ⏱ about 135 minutes

## What you will learn

- The difference between authentication and authorization, and the three kinds of proof of identity.
- Why a program must never store passwords as plain text, and why even a normal hash is not enough.
- What a salt is and how to use Argon2id with the `argon2` crate.
- How to build a small login system that gives the same answer for "no such user" and "wrong password".
- How to compare secrets in constant time, and what else a safe login needs.

## Before you start

- You finished [Lesson 10.5 — Injection attacks](05-injection-attacks.md).
- You know custom error types ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)), `HashMap` ([Lesson 04.9](../04-structs-enums-collections-errors/09-hash-maps.md)) and crates ([Lesson 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md)).
- Create the package: `cargo new --lib code/p10-passwords`, then inside it `cargo add argon2@0.6 sha2@0.11 subtle@2.6` and `mkdir src/bin`.
- This lesson uses version 0.6 of `argon2`; its API (Application Programming Interface: the functions and types it offers) changed in 2025–2026, so code from old tutorials may not compile.

## New words

| Word | Meaning |
|---|---|
| **authentication** (often "authn") | Proving **who you are**: "this request really comes from Ana". |
| **authorization** ("authz") | Deciding **what you may do** once we know who you are. Lesson 10.7. |
| **credential** | The proof a user gives: a password, a key, a code. |
| **authentication factor** | A kind of proof: something you **know** (password), something you **have** (phone, hardware key), something you **are** (fingerprint). |
| **MFA** (Multi-Factor Authentication) | Asking for two or more different factors, for example a password and a code from a phone. |
| **hash function** | A function that turns any data into a short fixed-size fingerprint. The same input always gives the same fingerprint, and you cannot go back from the fingerprint to the input. Lesson 10.9 explains hashes in detail. |
| **salt** | A random value, different for every password, that is mixed in before hashing. |
| **brute force** | Trying many guesses, one after another, until one works. |
| **rainbow table** | A big pre-computed list of "password → hash" pairs that lets an attacker reverse fast, unsalted hashes by lookup. |
| **Argon2id** | A password-hashing algorithm that is deliberately slow and needs a lot of memory, which makes mass guessing expensive. It won an international competition in 2015. |
| **PHC string** (Password Hashing Competition format) | A single text such as `$argon2id$v=19$m=19456,t=2,p=1$salt$hash` that holds the algorithm, its settings, the salt and the hash. |
| **timing attack** | Learning a secret by measuring how long the program takes to answer. |
| **constant-time comparison** | A comparison that takes the same time whatever the data is. |
| **user enumeration** | Finding out which user names exist, from small differences in the answers. |
| **SHA-256** (Secure Hash Algorithm, 256-bit) | A very common fast hash function. Its result always has 32 bytes. |
| **MAC** (Message Authentication Code) | A short value that proves a message was not changed and comes from someone who knows the secret key. Lesson 10.9. |
| **KiB / MiB** (kibibyte / mebibyte) | Units of memory size: 1024 bytes / 1024 × 1024 bytes. |

## 1. Who are you? Authentication

**Authentication** answers "who is this?". It comes before **authorization** ("what may this person do?").
A user proves their identity with one or more *factors*: something they **know** (a password), **have**
(a phone) or **are** (a fingerprint). Using two different factors is **MFA**, and it stops most attacks that
steal only the password.

Passwords remain the most common factor. So this lesson answers a practical question: **how does a
server check a password without keeping it?**

## 2. Three ways to store a password

Imagine an attacker who steals your database (this happens to real companies every year). What do they
get?

| Storage | What the thief gets | Verdict |
|---|---|---|
| **Plain text** | Every password, at once. Users who reuse the password lose other accounts too (*credential stuffing*, [Lesson 10.3](03-the-owasp-top-10.md)). | Never |
| **A fast hash**, such as SHA-256, with no salt | Every common password within minutes, because identical passwords have identical hashes and a rainbow table reverses them. | Never |
| **A slow, salted password hash** (Argon2id) | Only fingerprints that are expensive to guess one by one. | Yes |

The server never needs the password itself. At registration it stores the *hash*. At login it hashes
the typed password again, with the same settings and salt, and compares the results.

## 3. Why "slow" is a feature

For most uses a hash must be fast. For passwords the opposite is true: a legitimate login hashes **one**
password, but a thief with a stolen database must hash **billions** of guesses. If one guess costs ten
milliseconds instead of ten nanoseconds, the thief's work becomes a million times more expensive, while
a user does not notice.

Measure it yourself. Save `src/bin/speed.rs`:

```rust
use std::time::Instant;

use argon2::Argon2;
use argon2::password_hash::PasswordHasher;
use sha2::{Digest, Sha256};

fn main() {
    // SHA-256 is a fast hash (Lesson 10.9). How many guesses can a thief try per second?
    let start = Instant::now();
    let mut last = [0u8; 32];
    for i in 0..1_000_000u32 {
        last = Sha256::digest(format!("guess-{i}").as_bytes()).into();
    }
    let sha_time = start.elapsed();
    println!(
        "1,000,000 SHA-256 hashes: {sha_time:.2?} (last byte {})",
        last[31]
    );

    // Argon2id with the default settings.
    let argon2 = Argon2::default();
    let start = Instant::now();
    for i in 0..5 {
        argon2
            .hash_password(format!("guess-{i}").as_bytes())
            .expect("hashing failed");
    }
    let argon_time = start.elapsed() / 5;
    println!("one Argon2id hash:        {argon_time:.2?}");

    let ratio = sha_time.as_secs_f64() / 1_000_000.0;
    println!(
        "one Argon2id hash costs about {:.0} times more than one SHA-256 hash",
        argon_time.as_secs_f64() / ratio
    );
}
```

Run it with `cargo run --release --bin speed`. Use `--release`: crypto code is very slow in the debug mode.

```text
1,000,000 SHA-256 hashes: 51.97ms (last byte 83)
one Argon2id hash:        10.77ms
one Argon2id hash costs about 207304 times more than one SHA-256 hash
```

*(This is an example from one machine; your numbers will be different.)*

**What happens:**

1. The first loop computes one million SHA-256 hashes with `Sha256::digest`. (`sha2` is a crate from the
   RustCrypto project; the `Digest` trait gives the method `digest`.) A fast hash finishes in tens of milliseconds.
2. The second part runs Argon2id with the default settings five times and takes the average.
3. `Argon2::default()` is Argon2id version 19 with 19 MiB of memory, 2 passes and 1 lane. Each guess must
   fill and mix 19 MiB of memory, which makes special cracking hardware much less useful.
4. The last line prints the ratio. The thief's cost per guess is hundreds of thousands of times higher.

## 4. Salt and the PHC string

If two users choose the same password and the hash has no salt, the two stored values are identical, and
one successful guess opens both. A **salt** is a random value created for **each** password and stored next to
the hash. It is not secret. It only guarantees that every stored value is different and that tables of
pre-computed hashes are useless. Save `src/bin/hash.rs`:

```rust
use argon2::Argon2;
use argon2::password_hash::phc::PasswordHash;
use argon2::password_hash::{PasswordHasher, PasswordVerifier};

fn main() {
    let password = b"correct horse battery";
    let argon2 = Argon2::default();

    // The salt is chosen at random inside hash_password.
    let first = argon2
        .hash_password(password)
        .expect("hashing failed")
        .to_string();
    let second = argon2
        .hash_password(password)
        .expect("hashing failed")
        .to_string();
    println!("first:  {first}");
    println!("second: {second}");
    println!("same text? {}", first == second);

    // Verification reads the settings and the salt from the stored text.
    let parsed = PasswordHash::new(&first).expect("not a valid hash text");
    println!(
        "right password: {:?}",
        argon2.verify_password(password, &parsed)
    );
    println!(
        "wrong password: {:?}",
        argon2.verify_password(b"correct horse batterY", &parsed)
    );
}
```

```text
first:  $argon2id$v=19$m=19456,t=2,p=1$FYXVw/VkzT+3kvgzFLRSuw$g1Y/9adpmya9FCAsn3eLtODYr3wGeiloOwxxeC4svO8
second: $argon2id$v=19$m=19456,t=2,p=1$ZisNyWGnIyV89bq0Lgnzqg$sJ7glk8AD6bCrxiZNr/rcKeZmAHmqh0uzCsGNRodmjs
same text? false
right password: Ok(())
wrong password: Err(PasswordInvalid)
```

*(The salt and the hash change at every run.)*

**What happens:**

1. `argon2.hash_password(password)` creates a fresh random salt from the operating system's secure
   random generator, hashes the password and returns a `PasswordHash`. `.to_string()` turns it into a PHC
   string. The `PasswordHasher` trait provides the method, so the `use` line is needed.
2. The same password gives two different texts, because the salts differ. `same text? false`.
3. Read the text with `$` as a separator: `argon2id` (the algorithm), `v=19` (version),
   `m=19456,t=2,p=1` (settings: memory in KiB, number of passes, parallel lanes), then the salt, then the hash,
   both in Base64 (a way to write bytes as letters and digits). **Everything needed to check a password
   is in the one text**, so you store one column.
4. `PasswordHash::new(&first)` parses the text. `verify_password` reads the settings and the salt from it, hashes
   the typed password and compares. The comparison is done in constant time. A wrong password returns
   `Err(PasswordInvalid)`.

**The three settings.** *Memory* (`m`) makes guessing expensive for graphics cards. *Passes* (`t`) repeats the work.
*Lanes* (`p`) allows parallel work. OWASP's minimum for Argon2id is 19 MiB, 2 passes and 1 lane, which are the
crate's defaults. Choose the strongest setting your server can afford: each login needs that much memory
**at the same time**, so a thousand logins at once need a thousand times 19 MiB. Because the settings are
stored in the text, you can raise them later without breaking old hashes: each old hash is still checked with its own settings.

> 🔒 **Security:** use a **password-hashing** algorithm (Argon2id; scrypt or bcrypt if you must), never
> plain SHA-256, and never an algorithm you invented. Do not truncate passwords (bcrypt only reads the
> first 72 bytes; Argon2 reads them all).

## 5. A small login system

Now put it together in `src/lib.rs`. The design decisions are marked in the comments.

```rust
use std::collections::HashMap;

use argon2::password_hash::{PasswordHasher, PasswordVerifier};
use argon2::{Algorithm, Argon2, Params, Version};

pub const MIN_PASSWORD_CHARS: usize = 15;
pub const MAX_PASSWORD_CHARS: usize = 128;

#[derive(Debug, PartialEq, Eq)]
pub enum RegisterError {
    UserExists,
    PasswordTooShort,
    PasswordTooLong,
    HashingFailed,
}

/// One error for every kind of failure: the caller learns nothing extra.
#[derive(Debug, PartialEq, Eq)]
pub struct InvalidCredentials;

pub struct UserStore {
    argon2: Argon2<'static>,
    users: HashMap<String, String>, // user name -> PHC string
    dummy_hash: String,             // used when the user does not exist
}

impl UserStore {
    /// Safe defaults: Argon2id, 19 MiB of memory, 2 passes, 1 lane.
    pub fn new() -> Self {
        Self::with_params(Params::default())
    }

    /// Lets tests use cheap settings. Never do this in production.
    pub fn with_params(params: Params) -> Self {
        let argon2 = Argon2::new(Algorithm::Argon2id, Version::V0x13, params);
        let dummy_hash = argon2
            .hash_password(b"a password that nobody has")
            .expect("hashing with valid parameters works")
            .to_string();
        UserStore {
            argon2,
            users: HashMap::new(),
            dummy_hash,
        }
    }

    pub fn register(&mut self, name: &str, password: &str) -> Result<(), RegisterError> {
        let length = password.chars().count();
        if length < MIN_PASSWORD_CHARS {
            return Err(RegisterError::PasswordTooShort);
        }
        if length > MAX_PASSWORD_CHARS {
            return Err(RegisterError::PasswordTooLong);
        }
        if self.users.contains_key(name) {
            return Err(RegisterError::UserExists);
        }
        let hash = self
            .argon2
            .hash_password(password.as_bytes())
            .map_err(|_| RegisterError::HashingFailed)?
            .to_string();
        self.users.insert(name.to_string(), hash);
        Ok(())
    }

    pub fn login(&self, name: &str, password: &str) -> Result<(), InvalidCredentials> {
        let known = self.users.get(name);
        // Always do the expensive work, even for an unknown user, so that the
        // answer takes the same time and does not reveal which names exist.
        let hash = known.unwrap_or(&self.dummy_hash);
        let password_ok = self
            .argon2
            .verify_password(password.as_bytes(), hash.as_str())
            .is_ok();
        if known.is_some() && password_ok {
            Ok(())
        } else {
            Err(InvalidCredentials)
        }
    }

    /// For the demo only: the stored text of a user.
    pub fn stored_hash(&self, name: &str) -> Option<&str> {
        self.users.get(name).map(String::as_str)
    }
}

impl Default for UserStore {
    fn default() -> Self {
        Self::new()
    }
}
```

**What happens:**

1. `RegisterError` lists why registration can fail. `InvalidCredentials` is **one** error for every failed
   login. A caller (and so an attacker) cannot tell "no such user" from "wrong password".
2. `UserStore` keeps an `Argon2` object, a map from user name to PHC string, and a `dummy_hash`.
   In a real system the map is a database table.
3. `register` checks the password length by **characters** (`chars().count()`). The rules follow the advice of NIST (the
   National Institute of Standards and Technology, the standards agency of the United States) for passwords that are the only factor: at least 15 characters, and long
   passwords up to 128 are allowed. There are **no** rules like "one capital letter and one symbol": they make
   people choose predictable passwords. A real system also checks a list of known leaked passwords.
4. `hash_password(...).to_string()` is the same call as in `hash.rs`. The result is stored, the password is dropped.
5. In `login`, the line `known.unwrap_or(&self.dummy_hash)` is the clever part. For an unknown user we still verify
   the typed password, against a dummy hash. Otherwise "no such user" would answer in a microsecond, and
   "wrong password" in ten milliseconds, and an attacker would learn from the timing which names exist (user
   enumeration).
6. Only if the user exists **and** the password is right does `login` return `Ok`.
7. `with_params` exists for the tests: the cheap settings (8 KiB, one pass) make a test run in milliseconds.
   `Params::new(m_cost, t_cost, p_cost, output_len)` builds the settings, and `None` means the default
   output length. Never use cheap settings for real users.

Now see it work. Save `src/bin/login.rs` and run `cargo run --release --bin login`:

```rust
use std::time::Instant;

use p10_passwords::UserStore;

fn main() {
    let mut store = UserStore::new();

    println!("{:?}", store.register("ana", "short"));
    println!("{:?}", store.register("ana", "correct horse battery"));
    println!("{:?}", store.register("ana", "another long password"));

    let attempts = [
        ("ana", "correct horse battery"),
        ("ana", "wrong password here"),
        ("nobody", "wrong password here"),
    ];
    for (name, password) in attempts {
        let start = Instant::now();
        let result = store.login(name, password);
        println!("{name:<7} {result:?} in {:.1?}", start.elapsed());
    }
}
```

```text
Err(PasswordTooShort)
Ok(())
Err(UserExists)
ana     Ok(()) in 18.2ms
ana     Err(InvalidCredentials) in 10.7ms
nobody  Err(InvalidCredentials) in 13.2ms
```

The known user with a wrong password and the unknown user get the same error and take about the same
time. (Times vary a little from run to run.)

Finally the tests, at the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn cheap_store() -> UserStore {
        // 8 KiB of memory, 1 pass: fast enough for tests (never for real users).
        UserStore::with_params(Params::new(8, 1, 1, None).unwrap())
    }

    #[test]
    fn right_password_logs_in_and_wrong_one_does_not() {
        let mut store = cheap_store();
        store.register("ana", "correct horse battery").unwrap();
        assert_eq!(store.login("ana", "correct horse battery"), Ok(()));
        assert_eq!(
            store.login("ana", "correct horse batterY"),
            Err(InvalidCredentials)
        );
    }

    #[test]
    fn unknown_user_and_wrong_password_give_the_same_error() {
        let mut store = cheap_store();
        store.register("ana", "correct horse battery").unwrap();
        assert_eq!(
            store.login("nobody", "whatever password"),
            store.login("ana", "wrong password!!")
        );
    }

    #[test]
    fn password_length_rules() {
        let mut store = cheap_store();
        assert_eq!(
            store.register("a", "short"),
            Err(RegisterError::PasswordTooShort)
        );
        assert_eq!(
            store.register("a", &"x".repeat(129)),
            Err(RegisterError::PasswordTooLong)
        );
        assert!(store.register("a", &"x".repeat(128)).is_ok());
        assert_eq!(
            store.register("a", "another long password"),
            Err(RegisterError::UserExists)
        );
    }

    #[test]
    fn the_same_password_gives_different_stored_texts() {
        let mut store = cheap_store();
        store.register("ana", "correct horse battery").unwrap();
        store.register("bob", "correct horse battery").unwrap();
        assert_ne!(store.stored_hash("ana"), store.stored_hash("bob"));
    }

    #[test]
    fn the_stored_text_does_not_contain_the_password() {
        let mut store = cheap_store();
        store.register("ana", "correct horse battery").unwrap();
        let stored = store.stored_hash("ana").unwrap();
        assert!(stored.starts_with("$argon2id$v=19$"));
        assert!(!stored.contains("horse"));
    }
}
```

```bash
cargo test
```

```text
running 5 tests
test tests::the_stored_text_does_not_contain_the_password ... ok
test tests::password_length_rules ... ok
test tests::the_same_password_gives_different_stored_texts ... ok
test tests::unknown_user_and_wrong_password_give_the_same_error ... ok
test tests::right_password_logs_in_and_wrong_one_does_not ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**What happens:** the tests check a good login and a bad one, that both failure cases return equal errors,
the length rule (a short password is refused, 128 characters are accepted, 129 are refused, and a second registration of the same name fails), that equal passwords give
different stored texts, and that the stored text starts with `$argon2id$v=19$` and does not contain the password.

> 🧪 **Testing:** you cannot test "timing" reliably in a unit test, so test the *structure* that makes it
> safe: one error type, and the same code path for both failures. Review that code path carefully.

## 6. Comparing secrets in constant time

A normal `==` on two byte strings may stop at the first different byte. The time it takes then tells the
caller how many leading bytes were right: a **timing attack**, which can recover a secret byte by byte.
For hashes of passwords the library already does the safe thing. For anything else that you compare
yourself (an API key, a session token, a MAC from [Lesson 10.9](09-cryptography-hashes-macs-and-randomness.md)),
use the `subtle` crate. Add this to a `#[test]` function in your tests:

```rust
use subtle::ConstantTimeEq;

let stored: &[u8] = b"s3cr3t-token-value";
let attempt: &[u8] = b"s3cr3t-token-valuX";
let equal: bool = stored.ct_eq(attempt).into();
assert!(!equal);
```

`ct_eq` (from the `ConstantTimeEq` trait) compares **every** byte and returns a `Choice`, a value that is not a
plain `bool`, so the compiler does not branch early on it. `.into()` converts it to `bool` at the end.

## 7. What else a safe login needs

- **Rate limiting.** Allow only a few attempts per minute per account and per address, with growing
  delays. Guessing online must be slow. Careful: a hard lock after three tries lets an attacker lock others
  out (Lesson 10.1, Section 3.4).
- **MFA.** A stolen or guessed password alone is then not enough.
- **Breached-password check.** Refuse passwords that appear in public leaks (services such as
  "Have I Been Pwned" offer a range query that does not reveal the password).
- **Encrypted transport.** The password travels to the server, so use TLS (Transport Layer Security, the encryption of network connections, [Lesson 10.11](11-tls-and-certificates.md)).
- **Safe resets.** A reset link must be random, single-use and short-lived, and a reset must not reveal
  whether the e-mail address exists. Never e-mail a password.
- **No passwords in logs, errors or URLs** (Uniform Resource Locators, web addresses; [Lesson 10.8](08-secrets-management.md)).

## Common mistakes

> ⚠️ **Common mistake:** hashing with a general-purpose hash: `sha256(password)`. It is fast and gives a thief
> billions of guesses per second.

> ⚠️ **Common mistake:** returning "user not found" and "wrong password" as two different messages, or one
> of them much faster than the other.

> ⚠️ **Common mistake:** a maximum password length of 12 or 16 characters, or silently cutting passwords. Allow at least 64.

> ⚠️ **Common mistake:** testing with the production settings. 19 MiB per hash makes a big test suite slow; use
> cheap settings **in tests only**.

## Try it yourself

1. Type the library, the three programs and the tests. Run `cargo test`, then the programs with `--release`.
2. In `hash.rs`, create `Argon2` with `Params::new(64 * 1024, 3, 1, None)` (64 MiB, 3 passes) using
   `Argon2::new(Algorithm::Argon2id, Version::V0x13, params)` (add `use argon2::{Algorithm, Params, Version};`). How does the PHC string change?

## Exercises

### Exercise 1 — Choose the storage (easy)

Rate each way to store `"hunter2hunter2"` as good or bad, and say why: (a) the text itself; (b) base64 of the text;
(c) `sha256` with no salt; (d) `sha256` of the user name plus the password; (e) the Argon2id PHC string.

<details>
<summary>Hint</summary>

Ask two questions: can the thief reverse it at once (no work)? Can the thief try billions of guesses per second?

</details>

<details>
<summary>Answer</summary>

(a) Bad: plain text. (b) Bad: Base64 is an encoding, not a protection; anyone can decode it. (c) Bad: fast and
unsalted, so common passwords fall to a rainbow table. (d) Bad: the user name makes it unique per user (a weak
salt), but it is still a fast hash, so guessing is cheap, and the name is not random. (e) Good.

</details>

### Exercise 2 — Test the border of the rule (medium)

Write a **test** showing that `register` refuses a 14-character password and accepts a 15-character one.
Use cheap parameters.

<details>
<summary>Hint</summary>

`UserStore::with_params(Params::new(8, 1, 1, None).unwrap())`. Strings such as `"fourteen chars"` (14 characters)
and `"fifteen chars!!"` (15 characters) are easy to count.

</details>

<details>
<summary>Answer</summary>

```rust
#[test]
fn fourteen_characters_are_refused_and_fifteen_accepted() {
    let mut store = UserStore::with_params(Params::new(8, 1, 1, None).unwrap());
    assert_eq!(
        store.register("a", "fourteen chars"),
        Err(RegisterError::PasswordTooShort)
    );
    assert!(store.register("a", "fifteen chars!!").is_ok());
}
```

Put it inside `mod tests`. Why it works: the rule is `length < 15`, so 14 fails and 15 passes. Tests on the border
catch `<` against `<=` mistakes.

</details>

### Exercise 3 — Review a design (hard)

A colleague proposes: "We store `sha256(password + SECRET)`, where `SECRET` is one long random string in our
source code. That is like a salt, so we are safe." Name at least three problems and give the fix.

<details>
<summary>Hint</summary>

Is `SECRET` different for each user? Is SHA-256 slow? Where else does the source code go?

</details>

<details>
<summary>Answer</summary>

(1) `SECRET` is the same for every user, so equal passwords still give equal hashes, and one cracked
password shows who else has it. A salt must be random and unique per password. (2) SHA-256 is fast, so a thief
who has the database and the secret can try billions of guesses per second. (3) A secret inside the source code
ends up in Git history, builds and backups ([Lesson 10.8](08-secrets-management.md)). A single global secret
can be a *pepper* as an **extra** layer, kept outside the code and the database, but it does not replace a
salt and a slow hash. Fix: store an Argon2id PHC string created by `hash_password`, which makes a random salt for
each user.

</details>

## Quiz

1. What is the difference between authentication and authorization?

<details>
<summary>Answer</summary>

Authentication proves who someone is; authorization decides what that person may do.

</details>

2. Why is SHA-256 a bad choice for storing passwords?

<details>
<summary>Answer</summary>

It is designed to be fast, so a thief can try billions of guesses per second. A password-hashing algorithm is
deliberately slow and memory-hungry.

</details>

3. What does a salt do, and does it have to be secret?

<details>
<summary>Answer</summary>

It makes every stored hash different, even for equal passwords, and makes pre-computed tables useless. It does not have
to be secret and is stored next to the hash.

</details>

4. Why does `login` verify the password against a dummy hash for an unknown user?

<details>
<summary>Answer</summary>

So that the answer takes the same time as for a known user, and an attacker cannot find out which user names
exist by measuring the time.

</details>

5. When must you use constant-time comparison?

<details>
<summary>Answer</summary>

When you compare secret values yourself (tokens, keys, MACs). An ordinary `==` can stop early and leak how many
leading bytes were correct.

</details>

## Summary

- Authentication proves identity; authorization decides permissions. Use MFA when you can.
- Never store passwords as plain text or with a fast hash. Store a salted Argon2id hash as one PHC string.
- Slow and memory-hungry hashing makes mass guessing expensive; the settings are stored in the text and can be raised later.
- Give the same error, and about the same time, for an unknown user and a wrong password.
- Allow long passwords, require a sensible minimum length, no composition rules, and check a blocklist.
- Compare secrets with constant-time functions (`subtle`).
- Add rate limiting, TLS, safe resets, and never log passwords.

## Sources and further reading

- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — the recommended Argon2id settings and the alternatives.
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) — generic error messages, rate limiting, MFA and resets.
- [NIST SP 800-63B-4 — Authentication and Authenticator Management](https://pages.nist.gov/800-63-4/sp800-63b.html) — the official guidance of the United States government on password length, blocklists and composition rules (SP means Special Publication, the NIST series of security guides).
- [RFC 9106 — Argon2](https://www.rfc-editor.org/rfc/rfc9106) — the specification of the algorithm and advice on parameters (an RFC, Request for Comments, is a document in the official series of internet standards).
- [PHC string format](https://github.com/P-H-C/phc-string-format/blob/master/phc-sf-spec.md) — how the `$argon2id$…` text is structured.
- [`argon2` crate documentation](https://docs.rs/argon2/0.6.0/argon2/) — the API used here.
- [`subtle` crate documentation](https://docs.rs/subtle/2.6.1/subtle/) — constant-time comparison.

---

[🏠 Start](../START-HERE.md) · [← 10.5 Injection attacks: SQL, commands and paths](05-injection-attacks.md) · [📋 Phase 10](README.md) · [10.7 Authorization, sessions and tokens →](07-authorization-sessions-and-tokens.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
