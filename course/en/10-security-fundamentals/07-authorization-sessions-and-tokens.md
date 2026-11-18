<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.6 Authentication and password hashing](06-authentication-and-password-hashing.md) · [📋 Phase 10](README.md) · [10.8 Secrets management →](08-secrets-management.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/07-authorization-sessions-and-tokens.md) <!-- nav -->

# Lesson 10.7 — Authorization, sessions and tokens

> **Phase 10 — Security fundamentals** · Lesson 7 of 15 · ⏱ about 135 minutes

## What you will learn

- How to decide what a logged-in user may do: roles, ownership, and "deny by default".
- What a session is, and how to create session tokens that cannot be guessed, with timeouts and logout.
- What a cookie is and which attributes make it safer.
- What a JSON Web Token (JWT) is (JSON, JavaScript Object Notation, is a common text format for data), how it is signed, and which checks must never be skipped.
- When to choose a server-side session and when to choose a token.

## Before you start

- You finished [Lesson 10.6 — Authentication and password hashing](06-authentication-and-password-hashing.md).
- You know enums and `match` with tuples ([Lesson 04.4](../04-structs-enums-collections-errors/04-option-and-match.md)), `HashMap`, and `serde` derive ([Lesson 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md)).
- Create the package: `cargo new --lib code/p10-authz`, then inside it run `cargo add getrandom@0.4 hex@0.4 base64@0.23 sha2@0.11`, `cargo add serde@1 --features derive`, `cargo add jsonwebtoken@12 --features rust_crypto`, and `mkdir src/bin`.
- The `jsonwebtoken` crate (version 12) needs you to choose a cryptography backend with a feature (`rust_crypto` is written in Rust; `aws_lc_rs` is the other). Without one, signing fails.

## New words

| Word | Meaning |
|---|---|
| **role** | A named set of permissions given to users, for example *viewer*, *editor*, *admin*. |
| **RBAC** (Role-Based Access Control) | Deciding access from the user's role. |
| **ACL** (Access Control List) | A list on an object that says which users may do what with it. |
| **ABAC** (Attribute-Based Access Control) | Deciding access from many facts (user department, time of day, data label), not only a role. |
| **deny by default** | If no rule allows an action, it is refused. |
| **session** | The server's memory that "this browser belongs to Ana, who logged in 5 minutes ago". |
| **session token** (or session ID) | A random, secret text that the browser sends with each request, so the server can find the session. |
| **cookie** | A small piece of text that the server asks the browser to store and to send back with every later request to the same site. |
| **CSPRNG** (Cryptographically Secure Pseudo-Random Number Generator) | A source of random numbers that an attacker cannot predict. The operating system provides one. Lesson 10.9. |
| **session fixation** | An attack where the attacker gives the victim a session token that the attacker already knows, and then waits for the victim to log in. |
| **CSRF** (Cross-Site Request Forgery) | An attack where a malicious page makes the victim's browser send a request to another site, and the browser adds the victim's cookie automatically. |
| **bearer token** | A token that gives access to whoever *bears* (holds) it. Anyone who copies it can use it. |
| **JWT** (JSON Web Token) | A signed token with three Base64 parts, `header.payload.signature`, that carries claims such as who the user is and when the token expires. |
| **claim** | One statement inside a token: `sub` (subject, the user), `exp` (expiry time), `role`. |
| **signature** | A value computed from the data and a secret key, so that anyone with the key can detect any change to the data. |
| **HMAC** (Hash-based Message Authentication Code) | A way to sign data with a shared secret key. HS256 (HMAC-SHA-256) means HMAC with the hash function SHA-256 (Secure Hash Algorithm, 256-bit). Lesson 10.9. |
| **HTTP** (HyperText Transfer Protocol) | The protocol that browsers and web servers use to talk to each other. |
| **HTTPS** (HTTP Secure) | HTTP inside an encrypted TLS (Transport Layer Security) connection (Lesson 10.11). |
| **Base64** | A way to write any bytes with letters, digits and a few symbols. It is an encoding, not encryption. |

## 1. Authorization: what may you do?

After authentication ([Lesson 10.6](06-authentication-and-password-hashing.md)) the program knows **who** the
user is. **Authorization** decides **what** that user may do. Broken authorization is number one in the list of OWASP (the Open Worldwide Application Security Project)
([Lesson 10.3](03-the-owasp-top-10.md)). Four rules prevent most problems:

1. **Check on the server, on every request.** Never trust what the client says about its role.
2. **Deny by default.** A new action or a new role has no rights until you add them.
3. **Check the object, not only the role.** "Editors may edit notes" is not enough; it must be "editors may
   edit **their own** notes".
4. **Least privilege.** Give each role the smallest set of rights that does its job.

Common models: **ACL** (a list on each object), **RBAC** (rights belong to roles, users get roles) and **ABAC**
(rules over many attributes). RBAC plus an ownership check is enough for most programs.

Here is a small RBAC with ownership. Save the beginning of `src/lib.rs`:

```rust
use std::collections::HashMap;

use sha2::{Digest, Sha256};

// ---------- Authorization: who may do what ----------

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Role {
    Viewer,
    Editor,
    Admin,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum Action {
    Read,
    Write,
    Delete,
    ManageUsers,
}

pub struct User {
    pub name: String,
    pub role: Role,
}

#[derive(Debug, PartialEq, Eq)]
pub struct Denied;

/// `owner` is the owner of the object (a note, for example), if the action is about one.
pub fn authorize(user: &User, action: Action, owner: Option<&str>) -> Result<(), Denied> {
    let is_owner = owner == Some(user.name.as_str());
    let allowed = match (user.role, action) {
        (Role::Admin, _) => true,
        (Role::Editor, Action::Read) => true,
        (Role::Editor, Action::Write | Action::Delete) => is_owner,
        (Role::Viewer, Action::Read) => true,
        _ => false, // deny by default: anything not listed above is refused
    };
    if allowed { Ok(()) } else { Err(Denied) }
}
```

**What happens:**

1. `Role` and `Action` are enums, so every role and every action is a named, checked value, not a string that can be misspelled.
2. `authorize` gets the user, the action, and the owner of the object (`Option<&str>`, because some actions, like
   managing users, have no owner).
3. `is_owner` is true only when the owner exists **and** equals the user's name. `owner == Some(...)` compares
   `Option` values, so `None` never matches.
4. The `match` on the pair `(role, action)` lists every allowed combination. An administrator may do
   everything. An editor may read, and may write or delete only their own objects. A viewer may only read.
5. The last arm `_ => false` is **deny by default**: whatever is not listed is refused. (The trade-off: if you
   list every case without `_`, the compiler tells you when you add a new role and forget it. Some teams prefer
   that. Either way, the default result must be "no".)
6. The function returns `Result<(), Denied>`, so the caller cannot forget to look at the answer without the
   compiler warning about an unused `Result`.

> 🔒 **Security:** keep authorization in **one** function like this, called by every handler. Rules that are
> copied into ten places will not be changed in all ten when the policy changes.

## 2. Sessions

HTTP, the language of the web, is **stateless**: every request arrives alone, and the server does not remember
the one before. After a login the server must recognise the same user in the next request. It does so with a
**session**:

```text
1. Browser:  POST /login  (name + password)
2. Server:   checks the password, creates a session, makes a random token
             answers: Set-Cookie: session=<token>
3. Browser:  GET /notes   Cookie: session=<token>
4. Server:   finds the session by the token  ->  "this is Ana"
5. Browser:  POST /logout ...  Server: deletes the session
```

The token is the *key* to the session, so it must be secret and impossible to guess. Requirements:

- **Unpredictable.** At least 128 bits from a CSPRNG (the operating system's random generator). Never build it
  from the user name, the time, a counter, or a random generator seeded with the time.
- **Server-side state.** The server decides if a session is valid and can end it at once (logout, password
  change, "log out all devices").
- **Timeouts.** An *idle* timeout ends a session that is not used for a while (15 minutes, for example) and an
  *absolute* timeout ends it after a maximum age even if it is used all the time (for example 8 hours).
- **New token at login and at privilege change.** Otherwise *session fixation* is possible.
- **Not stored in readable form.** If the session table leaks, readable tokens become working logins. Store
  the **hash** of the token (SHA-256 is fine here, because the token is random and long, not a human password).

Continue `src/lib.rs`:

```rust
// ---------- Sessions: remembering who is logged in ----------

pub struct Session {
    pub user: String,
    created: u64,
    last_seen: u64,
}

pub struct SessionStore {
    sessions: HashMap<String, Session>, // key: SHA-256 of the token, never the token itself
    idle_timeout: u64,
    absolute_timeout: u64,
}

fn new_token() -> String {
    let mut bytes = [0u8; 32]; // 32 random bytes = 256 bits
    getrandom::fill(&mut bytes).expect("the operating system has no random numbers");
    hex::encode(bytes)
}

fn key_of(token: &str) -> String {
    hex::encode(Sha256::digest(token.as_bytes()))
}

impl SessionStore {
    /// Times are in seconds. `now` is passed in by the caller, which makes tests easy.
    pub fn new(idle_timeout: u64, absolute_timeout: u64) -> Self {
        SessionStore {
            sessions: HashMap::new(),
            idle_timeout,
            absolute_timeout,
        }
    }

    /// Creates a session after a successful login and returns the token for the cookie.
    pub fn create(&mut self, user: &str, now: u64) -> String {
        let token = new_token();
        let session = Session {
            user: user.to_string(),
            created: now,
            last_seen: now,
        };
        self.sessions.insert(key_of(&token), session);
        token
    }

    /// Returns the user if the token is valid now, and renews the idle timer.
    pub fn check(&mut self, token: &str, now: u64) -> Option<String> {
        let key = key_of(token);
        let session = self.sessions.get_mut(&key)?;
        let idle = now.saturating_sub(session.last_seen);
        let age = now.saturating_sub(session.created);
        if idle > self.idle_timeout || age > self.absolute_timeout {
            self.sessions.remove(&key);
            return None;
        }
        session.last_seen = now;
        Some(session.user.clone())
    }

    /// Logout: the token stops working at once, on the server side.
    pub fn revoke(&mut self, token: &str) {
        self.sessions.remove(&key_of(token));
    }
}
```

**What happens:**

1. `new_token` fills 32 bytes with `getrandom::fill`, which asks the operating system for secure random bytes.
   `hex::encode` writes them as 64 characters `0-9a-f`. That is 256 bits, more than enough.
2. `key_of` hashes the token with `Sha256::digest`. The map is keyed by the hash, so the table never holds a
   usable token.
3. `SessionStore::new` takes the two timeouts. Times are plain numbers of seconds, and `now` is a **parameter**.
   A function that reads the clock itself is hard to test; this one is easy.
4. `create` makes a token and stores the session. The caller sends the token to the browser.
5. `check` looks the session up by the hash of the given token. `?` returns `None` for an unknown token.
   `saturating_sub` avoids a negative result if the clock moves backwards. If the session was idle for too long
   or is too old, it is **removed** and `None` is returned. Otherwise `last_seen` is updated: the idle timer restarts.
6. `revoke` is logout. Because the state is on the server, it works at once.

See it work. Save `src/bin/session.rs` and run `cargo run --bin session`:

```rust
use p10_authz::SessionStore;

fn main() {
    // idle timeout 15 minutes, absolute timeout 8 hours (times in seconds)
    let mut store = SessionStore::new(15 * 60, 8 * 3600);

    let token = store.create("ana", 0);
    println!(
        "token: {} hex characters = {} bits",
        token.len(),
        token.len() * 4
    );

    println!("after 10 minutes:       {:?}", store.check(&token, 10 * 60));
    println!("after 20 minutes:       {:?}", store.check(&token, 20 * 60));
    println!("after 40 minutes:       {:?}", store.check(&token, 40 * 60));

    let token = store.create("ana", 100);
    store.revoke(&token);
    println!("after logout:           {:?}", store.check(&token, 101));
    println!(
        "a guessed token:        {:?}",
        store.check(&"0".repeat(64), 101)
    );
}
```

```text
token: 64 hex characters = 256 bits
after 10 minutes:       Some("ana")
after 20 minutes:       Some("ana")
after 40 minutes:       None
after logout:           None
a guessed token:        None
```

The session survives 10 minutes, and the check at 20 minutes still works because the idle timer was renewed at
minute 10 (only 10 idle minutes). After 20 more idle minutes (minute 40) it has expired. After `revoke` the token is dead, and a guessed token is simply unknown.

Add tests at the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn user(name: &str, role: Role) -> User {
        User {
            name: name.to_string(),
            role,
        }
    }

    #[test]
    fn roles_and_ownership() {
        let ana = user("ana", Role::Editor);
        assert_eq!(authorize(&ana, Action::Write, Some("ana")), Ok(()));
        assert_eq!(authorize(&ana, Action::Write, Some("bob")), Err(Denied));
        assert_eq!(authorize(&ana, Action::Read, Some("bob")), Ok(()));
        assert_eq!(authorize(&ana, Action::ManageUsers, None), Err(Denied));

        let vera = user("vera", Role::Viewer);
        assert_eq!(authorize(&vera, Action::Read, None), Ok(()));
        assert_eq!(authorize(&vera, Action::Delete, Some("vera")), Err(Denied));

        let root = user("root", Role::Admin);
        assert_eq!(authorize(&root, Action::ManageUsers, None), Ok(()));
    }

    #[test]
    fn a_session_works_until_it_expires() {
        let mut store = SessionStore::new(60, 600);
        let token = store.create("ana", 1_000);
        assert_eq!(store.check(&token, 1_030), Some("ana".to_string()));
        // 30 s later it is still fine, because the last check renewed the idle timer:
        assert_eq!(store.check(&token, 1_060), Some("ana".to_string()));
        // 61 s of silence: the idle timeout (60 s) has passed.
        assert_eq!(store.check(&token, 1_121), None);
        assert_eq!(store.check(&token, 1_122), None);
    }

    #[test]
    fn logout_and_guessing() {
        let mut store = SessionStore::new(60, 600);
        let token = store.create("ana", 0);
        assert_eq!(store.check("not-a-real-token", 1), None);
        store.revoke(&token);
        assert_eq!(store.check(&token, 2), None);
    }
}
```

```bash
cargo test
```

```text
running 3 tests
test tests::a_session_works_until_it_expires ... ok
test tests::logout_and_guessing ... ok
test tests::roles_and_ownership ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.01s
```

> 🧪 **Testing:** because the clock is a parameter, the tests can "travel in time" in one microsecond. When your
> code needs the time, pass it in rather than calling `SystemTime::now()` deep inside.

### Cookies

The server gives the token to the browser in a header, and the browser sends it back. The header looks like this:

```text
Set-Cookie: session=<token>; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=900
```

| Attribute | What it does |
|---|---|
| `HttpOnly` | JavaScript on the page cannot read the cookie, so an XSS (cross-site scripting) bug ([Lesson 10.5](05-injection-attacks.md)) cannot steal it. |
| `Secure` | The browser sends the cookie only over HTTPS ([Lesson 10.11](11-tls-and-certificates.md)). |
| `SameSite=Lax` (or `Strict`) | The browser does not add the cookie to most requests started by other sites. This blocks most **CSRF**. |
| `Path`, `Max-Age` | Where the cookie applies, and how long the browser keeps it. |

For actions that change data, add an extra anti-CSRF defense (a secret token inside the form) when the
framework you use does not already do it for you.

## 3. Tokens and JWT

A session needs a server-side table. A **token** such as a **JWT** carries the facts itself and is **signed**,
so the server needs no table: it checks the signature. A JWT looks like `xxxxx.yyyyy.zzzzz`:

```text
header.payload.signature        each part is Base64URL (letters, digits, - and _)
header:    {"typ":"JWT","alg":"HS256"}              which algorithm signed it
payload:   {"sub":"ana","role":"viewer","exp":...}  the claims
signature: HMAC-SHA256(secret, header + "." + payload)
```

Save `src/bin/jwt.rs`. It creates a token, reads it, and then attacks it three ways:

```rust
use base64::Engine;
use base64::engine::general_purpose::URL_SAFE_NO_PAD;
use jsonwebtoken::{Algorithm, DecodingKey, EncodingKey, Header, Validation, decode, encode};
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct Claims {
    sub: String,  // subject: who the token is about
    role: String, // our own claim
    exp: u64,     // expiry: seconds since 1970-01-01 (the "Unix time")
}

fn validation() -> Validation {
    let mut validation = Validation::new(Algorithm::HS256); // ONLY this algorithm is accepted
    validation.leeway = 0; // the default allows 60 seconds of clock difference
    validation
}

fn main() {
    // A demo secret. A real one is long, random and kept outside the code (Lesson 10.8).
    let secret = b"demo-secret-do-not-use-in-real-life";
    let claims = Claims {
        sub: "ana".into(),
        role: "viewer".into(),
        exp: 4_000_000_000,
    };

    let token = encode(
        &Header::new(Algorithm::HS256),
        &claims,
        &EncodingKey::from_secret(secret).expect("bad key"),
    )
    .expect("encoding failed");
    println!("token: {token}");

    // The first two parts are only Base64: anybody can read them.
    let parts: Vec<&str> = token.split('.').collect();
    for (name, part) in ["header ", "payload"].iter().zip(&parts) {
        let bytes = URL_SAFE_NO_PAD.decode(part).expect("not base64");
        println!("{name}: {}", String::from_utf8_lossy(&bytes));
    }

    let key = DecodingKey::from_secret(secret).expect("bad key");
    let ok = decode::<Claims>(&token, &key, &validation());
    println!("valid token:    {:?}", ok.map(|data| data.claims.role));

    // Attack 1: change "viewer" to "admin" in the payload, keep the old signature.
    let payload = String::from_utf8(URL_SAFE_NO_PAD.decode(parts[1]).unwrap()).unwrap();
    let forged_payload = URL_SAFE_NO_PAD.encode(payload.replace("viewer", "admin"));
    let forged = format!("{}.{}.{}", parts[0], forged_payload, parts[2]);
    let result = decode::<Claims>(&forged, &key, &validation());
    println!(
        "forged payload: {:?}",
        result
            .map(|data| data.claims.role)
            .map_err(|e| e.into_kind())
    );

    // Attack 2: a header that says "no signature needed".
    let none_header = URL_SAFE_NO_PAD.encode(r#"{"alg":"none","typ":"JWT"}"#);
    let unsigned = format!("{none_header}.{}.", parts[1]);
    let result = decode::<Claims>(&unsigned, &key, &validation());
    println!(
        "alg none:       {:?}",
        result
            .map(|data| data.claims.role)
            .map_err(|e| e.into_kind())
    );

    // Attack 3: an old token (expired in the year 2001).
    let old = Claims {
        sub: "ana".into(),
        role: "viewer".into(),
        exp: 1_000_000_000,
    };
    let old_token = encode(
        &Header::new(Algorithm::HS256),
        &old,
        &EncodingKey::from_secret(secret).expect("bad key"),
    )
    .expect("encoding failed");
    let result = decode::<Claims>(&old_token, &key, &validation());
    println!(
        "expired token:  {:?}",
        result
            .map(|data| data.claims.role)
            .map_err(|e| e.into_kind())
    );
}
```

Run `cargo run --bin jwt`:

```text
token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhbmEiLCJyb2xlIjoidmlld2VyIiwiZXhwIjo0MDAwMDAwMDAwfQ.crrEcjh-LPhucxLldtUAxBdYYhcCnwlsdsjBUcLKdzA
header : {"typ":"JWT","alg":"HS256"}
payload: {"sub":"ana","role":"viewer","exp":4000000000}
valid token:    Ok("viewer")
forged payload: Err(InvalidSignature)
alg none:       Err(Json(Error("unknown variant `none`, expected one of `HS256`, `HS384`, `HS512`, `ES256`, `ES384`, `RS256`, `RS384`, `RS512`, `PS256`, `PS384`, `PS512`, `EdDSA`", line: 1, column: 13)))
expired token:  Err(ExpiredSignature)
```

**What happens:**

1. `Claims` is an ordinary struct with `Serialize` and `Deserialize`. `exp` is the **expiry time**, in seconds
   since 1 January 1970 (the "Unix time"). `4_000_000_000` is in the year 2096, so the token is valid.
2. `Validation::new(Algorithm::HS256)` says: **accept only HS256**. The default `leeway` is 60 seconds of
   allowed clock difference; we set 0 so the test is exact. `exp` is checked automatically.
3. `encode` signs the claims with the secret. In version 12 `EncodingKey::from_secret` returns a `Result`, so we call
   `.expect(...)`.
4. The program splits the token at the dots and decodes the first two parts. **The payload is readable by anybody**
   (it is only Base64). A signature protects against *change*, not against *reading*. Never put secrets in a JWT.
5. **Attack 1.** Change `viewer` to `admin` in the payload and keep the signature. The signature was computed
   over the old payload, so it no longer matches: `InvalidSignature`. Without the secret the attacker cannot make a
   new valid signature.
6. **Attack 2** is the infamous `"alg":"none"`: a header claiming that no signature is needed. Some old libraries
   accepted it. Here the algorithm in the header must be one that we allow, and `none` is not even a known
   algorithm, so decoding fails.
7. **Attack 3**: a token that expired in 2001 gives `ExpiredSignature`.

**The checks that must never be skipped:**

- Decide the **algorithm** on the server (`Validation::new(Algorithm::HS256)`); never believe the `alg` of the token.
- Always check **`exp`**, and also `iss` (issuer) and `aud` (audience) when several services share tokens.
- Use a **long random secret** (at least 32 random bytes), kept outside the code ([Lesson 10.8](08-secrets-management.md)).
- Keep tokens **short-lived** (minutes). A JWT cannot be cancelled before it expires, unless you keep a block list,
  which brings back server state.
- Keep **no secrets or sensitive data** in the payload.
- Send tokens only over HTTPS, and treat them as passwords: a bearer token works for whoever holds it.

## 4. Session or token?

For a classic web site with a browser, a **server-side session** with a cookie is the simpler and safer choice:
it can be revoked at once (logout, password change, stolen device), and the cookie is small. A **signed token** is
good when many services must trust one login, because every service can check the signature without a shared
table; the price is that it cannot be cancelled before it expires, and it brings more rules to follow.

## Common mistakes

> ⚠️ **Common mistake:** checking the role in the user interface only. The server must check every request.

> ⚠️ **Common mistake:** a session ID made from a counter, the time, or a generator seeded with the time. These are
> guessable. Use bytes from the operating system.

> ⚠️ **Common mistake:** calling `decode` without a fixed algorithm, without `exp`, or with the secret `"secret"`.

## Try it yourself

1. Type the library, both programs and the tests. Run `cargo test`, `cargo run --bin session` and `cargo run --bin jwt`.
2. In `jwt.rs`, change `validation.leeway` back to the default (delete the line) and give the old token an `exp`
   of the current time minus 30 seconds. Does the token still pass? Why?
3. In `session.rs`, make the idle timeout 5 minutes. At which of the three checks does the session end?

## Exercises

### Exercise 1 — Find the missing check (easy)

What is missing? (a) `if user.role == Role::Editor { delete(note) }`. (b) A handler reads the role from a
JSON field in the request body. (c) `Validation::default()` is used, but the tokens are signed with RS256
(an RSA signature with SHA-256: a private key signs, a public key checks; RSA is named after its inventors Rivest,
Shamir and Adleman, and Lesson 10.10 explains signatures). (d) The session token is the user's id.

<details>
<summary>Hint</summary>

Think about which rule of Section 1 or Section 3 each line breaks.

</details>

<details>
<summary>Answer</summary>

(a) The ownership check: any editor can delete any note. (b) The client chooses its own role: the role must come from
the server (the session or a verified token). (c) The server never named the algorithm it accepts. The default is HS256, so RS256 tokens are refused;
always set it explicitly, for example `Validation::new(Algorithm::RS256)`. (d) The token is guessable: ids are small numbers; use 256 random bits.

</details>

### Exercise 2 — The absolute timeout (medium)

Write a **test** that proves a session that is used every 30 seconds still ends when it is older than the
absolute timeout (600 seconds).

<details>
<summary>Hint</summary>

Create the store with `SessionStore::new(60, 600)`, call `check` for `t` in `(30..=600).step_by(30)` and
expect `Some`, then check at 630.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn the_absolute_timeout_ends_even_a_busy_session() {
        let mut store = SessionStore::new(60, 600);
        let token = store.create("ana", 0);
        for t in (30..=600).step_by(30) {
            assert!(store.check(&token, t).is_some());
        }
        assert_eq!(store.check(&token, 630), None);
    }
```

Put it inside `mod tests`. Why it works: every check renews the idle timer, so the idle limit never triggers; at
630 seconds the age is above 600, and only the absolute timeout can end the session.

</details>

### Exercise 3 — Rotate the token (hard)

Add `rotate(&mut self, old_token: &str, now: u64) -> Option<String>` to `SessionStore`. It must give the same user a **new**
token and make the old one useless. Write a **test**.

<details>
<summary>Hint</summary>

`check` returns the user name (or `None` for an invalid token), then `revoke` the old token and `create` a new one.

</details>

<details>
<summary>Answer</summary>

```rust
    /// Gives the user a new token and kills the old one (use it after login or a role change).
    pub fn rotate(&mut self, old_token: &str, now: u64) -> Option<String> {
        let user = self.check(old_token, now)?;
        self.revoke(old_token);
        Some(self.create(&user, now))
    }
```

```rust
    #[test]
    fn rotate_replaces_the_token() {
        let mut store = SessionStore::new(60, 600);
        let old = store.create("ana", 0);
        let new = store.rotate(&old, 10).unwrap();
        assert_ne!(old, new);
        assert_eq!(store.check(&old, 11), None);
        assert_eq!(store.check(&new, 11), Some("ana".to_string()));
        assert_eq!(store.rotate("bogus", 12), None);
    }
```

Why it works: after `rotate`, an attacker who knew the old token (session fixation) holds a token that no longer
exists, and an invalid token cannot be rotated into a valid one.

</details>

## Quiz

1. Name the four rules of authorization from Section 1.

<details>
<summary>Answer</summary>

Check on the server on every request; deny by default; check the object (ownership) as well as the role; least
privilege.

</details>

2. Why does the session store keep the hash of the token and not the token?

<details>
<summary>Answer</summary>

If the table leaks, hashes cannot be used as tokens, so the leak does not give attackers working sessions.

</details>

3. Is the payload of a JWT secret? What protects it?

<details>
<summary>Answer</summary>

No, it is only Base64 and can be read by anybody. The signature protects it from being changed, not from being
read.

</details>

4. Why should the server choose the algorithm and not the token?

<details>
<summary>Answer</summary>

An attacker controls the token. If the server believed the `alg` in the header, the attacker could ask for `none` or
for a weaker algorithm.

</details>

## Summary

- Authorization: check on the server on every request, deny by default, check ownership as well as role, least privilege.
- Keep the policy in one function, and return a `Result` so it cannot be ignored.
- A session token is 128 or more random bits from the operating system; the server keeps the state, the hash of the token, and idle and absolute timeouts.
- Rotate the token at login; logout deletes the session on the server.
- Cookies: `HttpOnly`, `Secure`, `SameSite`.
- A JWT is signed, not encrypted: fix the algorithm, check `exp`, keep it short-lived, no secrets inside.
- Pass the time into functions so that time-dependent rules are testable.

## Sources and further reading

- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html) — deny by default, least privilege, and checks on every request.
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) — token length, cookies, timeouts and session fixation.
- [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html) — how cross-site request forgery works and how to stop it.
- [MDN — Set-Cookie header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Set-Cookie) — every cookie attribute explained, on MDN (the Mozilla Developer Network, a reference site for web technologies).
- [RFC 7519 — JSON Web Token (JWT)](https://www.rfc-editor.org/rfc/rfc7519) — the standard that defines the token and its claims (an RFC, Request for Comments, is a document of the official internet standards series).
- [RFC 8725 — JWT Best Current Practices](https://www.rfc-editor.org/rfc/rfc8725) — the list of known attacks and the checks that stop them.
- [`jsonwebtoken` crate documentation](https://docs.rs/jsonwebtoken/12.0.0/jsonwebtoken/) — `encode`, `decode`, `Validation` and the backend features.
- [`getrandom` crate documentation](https://docs.rs/getrandom/0.4.3/getrandom/) — secure random bytes from the operating system.

---

[🏠 Start](../START-HERE.md) · [← 10.6 Authentication and password hashing](06-authentication-and-password-hashing.md) · [📋 Phase 10](README.md) · [10.8 Secrets management →](08-secrets-management.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
