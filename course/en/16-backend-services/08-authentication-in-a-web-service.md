<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 16.7 Migrations and transactions](07-migrations-and-transactions.md) · [📋 Phase 16](README.md) · [16.9 gRPC with tonic →](09-grpc-with-tonic.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/16-backend-services/08-authentication-in-a-web-service.md) <!-- nav -->

# Lesson 16.8 — Authentication in a web service

> **Phase 16 — Backend services** · Lesson 8 of 16 · ⏱ about 3 hours (2 sittings)

## What you will learn

- How a web service lets users register and log in, using what Phase 10 taught about passwords and tokens.
- How to store passwords with Argon2, and why the hashing runs in `spawn_blocking`.
- How to create random session tokens, store only their hashes, and give them an expiry date.
- How to write an axum extractor, `AuthUser`, that makes any handler require a login just by taking it as a parameter.
- How to test registration, login and protected endpoints.

## Before you start

- You finished [Lesson 16.7 — Migrations and transactions](07-migrations-and-transactions.md).
- You know the ideas of Lesson 10.6 (Authentication and password hashing) and Lesson 10.7 (Authorization, sessions
  and tokens): never store passwords, store slow salted hashes; a session token is a random secret that stands for a
  login. Section 1 repeats the essentials.
- You know hashes and secure randomness from [Lesson 10.9 — Cryptography I](../10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md).

## New words

| Word | Meaning |
|---|---|
| **authentication** | Finding out **who** sends a request ("this is Ana"). Its failure is `401 Unauthorized`. |
| **authorization** | Deciding what that person **may** do ("Ana may change her own tasks"). Its failure is `403 Forbidden` (or `404`). |
| **credentials** | What a user proves their identity with: here an email and a password. |
| **Argon2** | A password hashing function that is deliberately slow and memory-hungry, so guessing passwords is expensive. `Argon2id` is the recommended variant. |
| **salt** | Random bytes mixed into each password hash, so two users with the same password get different hashes. |
| **PHC string** (Password Hashing Competition) | The text format of a password hash that contains everything needed to check it: algorithm, parameters, salt, hash. It starts with `$argon2id$`. |
| **session token** | A long random secret the server gives after login. The client sends it with every request instead of the password. |
| **bearer token** | A token sent in the header `Authorization: Bearer <token>`. Whoever holds ("bears") it is treated as the user. |
| **expiry** | The moment after which a token stops working. |
| **user enumeration** | Finding out which accounts exist by watching how the service answers (different messages, different timing). |

## 1. The plan

Phase 10 taught the rules; this lesson builds them into the tasks service:

```text
 1. POST /users     {"email","password"}  → hash the password (Argon2), store the user
 2. POST /sessions  {"email","password"}  → check the password, create a random token,
                                             store SHA-256(token) with an expiry, return the token once
 3. any request     Authorization: Bearer <token>
                    → AuthUser extractor: SHA-256(token) → find a valid session → user id
```

The important decisions:

- **Passwords are never stored**, only Argon2 hashes. A stolen database does not reveal them.
- **Tokens are random** (32 bytes from the operating system), so nobody can guess one.
- **Only the hash of each token is stored.** Someone who reads the `sessions` table cannot use what they read. A plain
  fast hash (SHA-256, Secure Hash Algorithm, 256-bit) is enough here, because a token is 256 random bits, not a human password that could be guessed.
- **Tokens expire** after seven days, and every check compares with the current time.
- Reading tasks stays public in this lesson; **creating, changing, deleting and archiving require a login.**
  Exercise 3 makes tasks private to their owner.

## 2. Users and sessions in the database

New crates: `argon2` for passwords, `sha2` for SHA-256, `getrandom` for secure random bytes, `hex` to write bytes as
text:

```bash
cd ~/projects/rust/code/p16-tasks
cargo add argon2@0.6 sha2@0.11 getrandom@0.4 hex@0.4
cargo sqlx migrate add create_users_and_sessions
```

The new migration:

```sql
-- Accounts and login sessions.
CREATE TABLE users (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email         TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- We store only a SHA-256 hash of each token, never the token itself.
CREATE TABLE sessions (
    token_hash BYTEA PRIMARY KEY,
    user_id    BIGINT NOT NULL REFERENCES users (id) ON DELETE CASCADE,
    expires_at TIMESTAMPTZ NOT NULL
);

CREATE INDEX sessions_user_id_idx ON sessions (user_id);
```

`UNIQUE` on `email` makes the database refuse a second account with the same address, even if two registrations
arrive at the same moment. `BYTEA` is PostgreSQL's type for raw bytes (`Vec<u8>` in Rust). `ON DELETE CASCADE` ends
all sessions of a deleted user. Run `cargo sqlx migrate run`.

## 3. Errors for authentication

Add three variants to `ApiError` in `src/error.rs`:

```rust
    #[error("{0}")]
    Unauthorized(&'static str),
    #[error("{0}")]
    Conflict(String),
    #[error("internal error")]
    Database(#[from] sqlx::Error),
    #[error("internal error")]
    Internal(String),
```

In `status()`: `Unauthorized` → `StatusCode::UNAUTHORIZED`, `Conflict` → `StatusCode::CONFLICT`, and
`ApiError::Database(_) | ApiError::Internal(_)` → `StatusCode::INTERNAL_SERVER_ERROR`. In `code()`: `"unauthorized"`,
`"conflict"`, and `"internal"` for both internal variants. In `into_response`, log both internal variants:

```rust
        // The details go to the log, never to the client.
        match &self {
            ApiError::Database(error) => tracing::error!(%error, "database error"),
            ApiError::Internal(error) => tracing::error!(%error, "internal error"),
            _ => {}
        }
```

`Internal` is for failures that are not database errors, such as a failed random-number source. Like `Database`, it
shows only `internal error` to the client.

## 4. Passwords and tokens: `src/auth.rs`

Create `src/auth.rs`. It starts with the imports, the JSON (JavaScript Object Notation) types, and the input rules:

```rust
use argon2::Argon2;
use argon2::password_hash::{PasswordHasher, PasswordVerifier, phc::PasswordHash};
use axum::{
    Json,
    extract::{FromRequestParts, State},
    http::{StatusCode, header, request::Parts},
};
use serde::{Deserialize, Serialize};
use sha2::{Digest, Sha256};

use crate::AppState;
use crate::error::{ApiError, AppJson};

#[derive(Debug, Deserialize)]
pub struct Credentials {
    email: String,
    password: String,
}

#[derive(Debug, Serialize)]
pub struct User {
    id: i64,
    email: String,
}

#[derive(Debug, Serialize)]
pub struct Session {
    token: String,
}

/// Lowercase, trimmed, with exactly one `@` that is not at the start or the end.
fn parse_email(text: &str) -> Result<String, ApiError> {
    let email = text.trim().to_lowercase();
    let valid = email.len() <= 254
        && email.matches('@').count() == 1
        && !email.starts_with('@')
        && !email.ends_with('@');
    if valid {
        Ok(email)
    } else {
        Err(ApiError::Validation("email is not valid".to_string()))
    }
}

fn check_password_rules(password: &str) -> Result<(), ApiError> {
    let chars = password.chars().count();
    if !(12..=128).contains(&chars) {
        return Err(ApiError::Validation(
            "password must have 12 to 128 characters".to_string(),
        ));
    }
    Ok(())
}
```

`use crate::AppState;` works because `AppState` is defined in `main.rs`, the root of the crate. The email check is
deliberately simple (real email validation is very hard; the only real proof is an email the user receives). Email
addresses are stored in lowercase, so `Ana@Example.com` and `ana@example.com` are one account. The password has a
minimum length (Lesson 10.6) and also a **maximum**: hashing a megabyte-long "password" would cost the server a lot.

Then the password and token functions:

```rust
/// Argon2 is slow on purpose, so it runs on a blocking thread, not on the async runtime.
async fn hash_password(password: String) -> Result<String, ApiError> {
    tokio::task::spawn_blocking(move || {
        Argon2::default()
            .hash_password(password.as_bytes())
            .map(|hash| hash.to_string())
    })
    .await
    .map_err(|error| ApiError::Internal(error.to_string()))?
    .map_err(|error| ApiError::Internal(error.to_string()))
}

async fn verify_password(password: String, stored_hash: String) -> Result<bool, ApiError> {
    tokio::task::spawn_blocking(move || {
        let parsed = PasswordHash::new(&stored_hash)?;
        Ok::<bool, argon2::password_hash::Error>(
            Argon2::default()
                .verify_password(password.as_bytes(), &parsed)
                .is_ok(),
        )
    })
    .await
    .map_err(|error| ApiError::Internal(error.to_string()))?
    .map_err(|error| ApiError::Internal(error.to_string()))
}

/// 32 random bytes from the operating system, written as 64 hex characters.
fn new_token() -> Result<String, ApiError> {
    let mut bytes = [0u8; 32];
    getrandom::fill(&mut bytes).map_err(|error| ApiError::Internal(error.to_string()))?;
    Ok(hex::encode(bytes))
}

fn hash_token(token: &str) -> Vec<u8> {
    Sha256::digest(token.as_bytes()).to_vec()
}
```

**What happens:**

1. `Argon2::default()` is Argon2id with the recommended parameters. `hash_password` creates a random salt itself and
   returns a PHC string like `$argon2id$v=19$m=19456,t=2,p=1$<salt>$<hash>`: algorithm, version, memory (19 MiB; a MiB, mebibyte, is 1 048 576 bytes),
   iterations (2), parallelism (1), salt and hash, all in one text you can store in `TEXT`.
2. `verify_password` reads those parameters back from the stored string (`PasswordHash::new`), hashes the given
   password the same way and compares. It returns `Ok` only for the right password.
3. Hashing takes tens to hundreds of milliseconds **on purpose**: that is what makes guessing expensive. On the async
   runtime that would block a worker thread (Lesson 16.1), so the work runs in `tokio::task::spawn_blocking`, on a
   separate pool of threads made for blocking work. `.await` gives a `Result` from the thread (it fails only if the
   closure panicked), which contains the `Result` of the hashing: hence the two `map_err`.
4. `getrandom::fill` fills the array with bytes from the operating system's secure random generator (Lesson 10.9).
   `hex::encode` turns 32 bytes into 64 characters `0-9a-f`.
5. `hash_token` is SHA-256 of the token, 32 bytes, stored as `BYTEA`.

## 5. Register and log in

Still in `src/auth.rs`, the two handlers:

```rust
/// POST /users: create an account.
pub async fn register(
    State(state): State<AppState>,
    AppJson(credentials): AppJson<Credentials>,
) -> Result<(StatusCode, Json<User>), ApiError> {
    let email = parse_email(&credentials.email)?;
    check_password_rules(&credentials.password)?;
    let password_hash = hash_password(credentials.password).await?;

    let user = sqlx::query_as!(
        User,
        "INSERT INTO users (email, password_hash) VALUES ($1, $2) RETURNING id, email",
        email,
        password_hash
    )
    .fetch_one(&state.pool)
    .await
    .map_err(|error| match &error {
        sqlx::Error::Database(db) if db.is_unique_violation() => {
            ApiError::Conflict("email is already registered".to_string())
        }
        _ => ApiError::Database(error),
    })?;
    Ok((StatusCode::CREATED, Json(user)))
}

/// POST /sessions: log in, get a token.
pub async fn login(
    State(state): State<AppState>,
    AppJson(credentials): AppJson<Credentials>,
) -> Result<(StatusCode, Json<Session>), ApiError> {
    let wrong = || ApiError::Unauthorized("wrong email or password");
    let email = parse_email(&credentials.email).map_err(|_| wrong())?;

    let row = sqlx::query!(
        "SELECT id, password_hash FROM users WHERE email = $1",
        email
    )
    .fetch_optional(&state.pool)
    .await?;
    let Some(row) = row else {
        return Err(wrong());
    };
    if !verify_password(credentials.password, row.password_hash).await? {
        return Err(wrong());
    }

    let token = new_token()?;
    sqlx::query!(
        "INSERT INTO sessions (token_hash, user_id, expires_at)
         VALUES ($1, $2, now() + interval '7 days')",
        hash_token(&token),
        row.id
    )
    .execute(&state.pool)
    .await?;
    Ok((StatusCode::CREATED, Json(Session { token })))
}
```

**What happens:**

1. `register` validates, hashes, and inserts. If the email exists, PostgreSQL refuses the `INSERT` with a *unique
   violation*; `is_unique_violation()` recognises it, and the client gets `409 Conflict` instead of a `500`. Letting
   the database decide is correct even when two requests race.
2. `login` gives **one** answer for every failure, "wrong email or password": unknown email, wrong password and an
   invalid email all look the same, so the answer does not tell an attacker which accounts exist.
3. `let Some(row) = row else { ... };` is a *let-else*: if the pattern does not match, the `else` block must leave the
   function.
4. On success it stores the **hash** of a new token with an expiry seven days ahead (`now() + interval '7 days'`
   is computed by PostgreSQL) and returns the token itself. This is the only moment the token exists outside the
   client.
5. `POST /sessions` creates a session resource, so `201 Created` fits (Lesson 16.2).

> 🔒 **Security:** the same message is not the whole story. An unknown email returns at once, a wrong password only
> after Argon2 has run, so a careful attacker can still tell them apart by **timing**. The usual fix is to verify the
> password against a fixed dummy hash when the email is unknown, so both paths take the same time. Also limit login
> attempts per address and per account (rate limiting, Lesson 16.15), and send tokens only over TLS (Lesson 15.12):
> a bearer token on plain HTTP (HyperText Transfer Protocol) can be read by anyone on the network.

## 6. The `AuthUser` extractor

Every handler that needs a login must read the `Authorization` header, hash the token, and look up a valid session.
Writing that in every handler would be repetitive and easy to forget. An **extractor** does it once. The rest of
`src/auth.rs`:

```rust
/// The user who sent the request. Handlers that take this parameter require a login.
pub struct AuthUser {
    pub id: i64,
}

impl FromRequestParts<AppState> for AuthUser {
    type Rejection = ApiError;

    async fn from_request_parts(parts: &mut Parts, state: &AppState) -> Result<Self, ApiError> {
        let missing = || ApiError::Unauthorized("missing or invalid token");
        let token = parts
            .headers
            .get(header::AUTHORIZATION)
            .and_then(|value| value.to_str().ok())
            .and_then(|value| value.strip_prefix("Bearer "))
            .ok_or_else(missing)?;

        let user_id = sqlx::query_scalar!(
            "SELECT user_id FROM sessions WHERE token_hash = $1 AND expires_at > now()",
            hash_token(token)
        )
        .fetch_optional(&state.pool)
        .await?
        .ok_or_else(missing)?;
        Ok(AuthUser { id: user_id })
    }
}

/// GET /me: who am I?
pub async fn me(user: AuthUser, State(state): State<AppState>) -> Result<Json<User>, ApiError> {
    let user = sqlx::query_as!(User, "SELECT id, email FROM users WHERE id = $1", user.id)
        .fetch_one(&state.pool)
        .await?;
    Ok(Json(user))
}
```

**What happens:**

1. `FromRequestParts` is the trait behind `Path`, `Query` and `State` (Lesson 16.4 derived it for `AppPath`). Here we
   implement it by hand: axum calls `from_request_parts` before the handler, with the request's headers and the state.
2. `type Rejection = ApiError;`: if the extractor fails, its `ApiError` becomes the response (`401` with our JSON) and
   the handler never runs.
3. The header must look like `Bearer <token>`; anything else is "missing or invalid".
4. The query finds a session with this token hash **that has not expired**. No row means an unknown, logged-out or
   expired token: all three get the same `401`.
5. `me` shows the pattern: a handler that takes `user: AuthUser` can only run for a logged-in user, and it knows
   who that is.

Now protect the task endpoints. In `src/main.rs`, add `mod auth;` next to the other modules and `use auth::AuthUser;`
next to the other `use` lines. Then give `create_task`, `update_task`, `delete_task` and `archive_done` a first
parameter `_user: AuthUser`:

```rust
async fn create_task(
    _user: AuthUser,
    State(state): State<AppState>,
    AppJson(new_task): AppJson<NewTask>,
) -> Result<impl IntoResponse, ApiError> {
```

(The `_` says "we need the check, not the value", so the compiler does not warn.) The body of each handler stays the
same. Finally the three new routes in `app()`, after the task routes:

```rust
        .route("/users", post(auth::register))
        .route("/sessions", post(auth::login))
        .route("/me", get(auth::me))
```

Run `cargo sqlx prepare` again (new queries), then `cargo run`.

## 7. Try the whole flow

In the second terminal (`H` is just a shell variable to save typing):

```bash
H='content-type: application/json'
curl -s -H "$H" -d '{"email":"Ana@Example.com","password":"correct horse battery"}' http://127.0.0.1:3000/users; echo
curl -s -H "$H" -d '{"email":"ana@example.com","password":"another long password"}' http://127.0.0.1:3000/users; echo
curl -s -H "$H" -d '{"email":"bob@example.com","password":"short"}' http://127.0.0.1:3000/users; echo
curl -s -H "$H" -d '{"email":"ana@example.com","password":"correct horse battery"}' http://127.0.0.1:3000/sessions; echo
curl -s -H "$H" -d '{"email":"ana@example.com","password":"wrong password"}' http://127.0.0.1:3000/sessions; echo
curl -s -H "$H" -d '{"email":"nobody@example.com","password":"correct horse battery"}' http://127.0.0.1:3000/sessions; echo
```

```text
{"id":1,"email":"ana@example.com"}
{"code":"conflict","message":"email is already registered"}
{"code":"validation_failed","message":"password must have 12 to 128 characters"}
{"token":"155e63c7c5629273e9c94777105fecbb98e295537822ee39e89982194c84316e"}
{"code":"unauthorized","message":"wrong email or password"}
{"code":"unauthorized","message":"wrong email or password"}
```

The email was stored in lowercase, so the second registration with `ana@example.com` is a conflict. Your token is
different, of course. Now a write without and with a token. The shell command `$(...)` runs a command and puts its
output into the variable; `sed` cuts the token out of the JSON:

```bash
curl -s -H "$H" -d '{"title":"Buy milk"}' http://127.0.0.1:3000/tasks; echo
TOKEN=$(curl -s -H "$H" -d '{"email":"ana@example.com","password":"correct horse battery"}' \
  http://127.0.0.1:3000/sessions | sed 's/.*"token":"\([0-9a-f]*\)".*/\1/')
curl -s -H "$H" -H "authorization: Bearer $TOKEN" -d '{"title":"Buy milk"}' http://127.0.0.1:3000/tasks; echo
curl -s -H "authorization: Bearer $TOKEN" http://127.0.0.1:3000/me; echo
curl -s -H "authorization: Bearer 0000" http://127.0.0.1:3000/me; echo
```

```text
{"code":"unauthorized","message":"missing or invalid token"}
{"id":1,"title":"Buy milk","done":false}
{"id":1,"email":"ana@example.com"}
{"code":"unauthorized","message":"missing or invalid token"}
```

Look at what the database knows (in `psql`):

```sql
SELECT id, email, password_hash FROM users;
SELECT encode(token_hash, 'hex') AS token_hash, user_id, expires_at > now() AS valid FROM sessions;
```

```text
 id |      email      |                                           password_hash
----+-----------------+---------------------------------------------------------------------------------------------------
  1 | ana@example.com | $argon2id$v=19$m=19456,t=2,p=1$TmnAISn+Stbdb/Njn9u+iQ$2CqEm5IPbGeZ3QlkNtF/vtHhp7Zl57gRGTUdXDelMJU
(1 row)

                            token_hash                            | user_id | valid
------------------------------------------------------------------+---------+-------
 408ff18aeaea528da6a07d1883ee82a3ba07c54ae4183cd135896cf969d7da0f |       1 | t
 2c0bd48eb942127d9725a743ec1c62ae73415ce8c4e41f38b64e5051f5c81b16 |       1 | t
(2 rows)
```

No password and no token appears anywhere: only the PHC string (with its salt `TmnAISn+...`) and two token hashes,
one per login. Compare the token you received with the hashes: they are unrelated, as they must be.

> ⚡ **Performance:** the service log shows `latency=239 ms` for `POST /users`, almost all of it Argon2 in a debug
> build. That cost is the protection, but it also means that login is an expensive endpoint that attackers can use to
> load your server: one more reason for rate limiting. Release builds (`cargo run --release`) are much faster.

## 8. Tests with a logged-in user

The old tests that create tasks now get `401`. Give the test helpers a token: `post_json` adds an `Authorization`
header, and `login` registers and logs in a test user:

```rust
    fn post_json(uri: &str, token: &str, json: &'static str) -> Request<Body> {
        Request::post(uri)
            .header(header::CONTENT_TYPE, "application/json")
            .header(header::AUTHORIZATION, format!("Bearer {token}"))
            .body(Body::from(json))
            .unwrap()
    }

    /// Registers a user, logs in and returns the token.
    async fn login(app: &Router) -> String {
        let credentials = r#"{"email":"ana@example.com","password":"correct horse battery"}"#;
        let (status, _) = send(app.clone(), post_json("/users", "", credentials)).await;
        assert_eq!(status, StatusCode::CREATED);
        let (status, body) = send(app.clone(), post_json("/sessions", "", credentials)).await;
        assert_eq!(status, StatusCode::CREATED);
        // The body is {"token":"<64 hex characters>"}.
        body[10..74].to_string()
    }
```

`body[10..74]` cuts the token out of `{"token":"` (10 characters) plus 64 hex characters; for the anonymous calls the
token is `""`, which the server simply ignores for `/users` and `/sessions`. Update the tests that write tasks to call
`let token = login(&app).await;` first and pass `&token`, for example:

```rust
    #[sqlx::test]
    async fn created_task_can_be_read_back(pool: PgPool) {
        let app = test_app(pool).await;
        let token = login(&app).await;

        let (status, body) = send(
            app.clone(),
            post_json("/tasks", &token, r#"{"title":"Buy milk"}"#),
        )
        .await;
        assert_eq!(status, StatusCode::CREATED);
        assert_eq!(body, r#"{"id":1,"title":"Buy milk","done":false}"#);

        let request = Request::get("/tasks/1").body(Body::empty()).unwrap();
        let (status, body) = send(app, request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, r#"{"id":1,"title":"Buy milk","done":false}"#);
    }
```

Do the same in `empty_title_is_rejected` and `archive_is_all_or_nothing` (there `post_empty` also takes the token and
adds the same header). Then a test for the authentication itself:

```rust
    #[sqlx::test]
    async fn the_token_identifies_the_user(pool: PgPool) {
        let app = test_app(pool).await;
        let token = login(&app).await;

        let request = Request::get("/me")
            .header(header::AUTHORIZATION, format!("Bearer {token}"))
            .body(Body::empty())
            .unwrap();
        let (status, body) = send(app.clone(), request).await;
        assert_eq!(status, StatusCode::OK);
        assert_eq!(body, r#"{"id":1,"email":"ana@example.com"}"#);

        let wrong = r#"{"email":"ana@example.com","password":"wrong password!!"}"#;
        let (status, body) = send(app, post_json("/sessions", "", wrong)).await;
        assert_eq!(status, StatusCode::UNAUTHORIZED);
        assert_eq!(
            body,
            r#"{"code":"unauthorized","message":"wrong email or password"}"#
        );
    }
```

```text
test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.76s
```

The tests take longer now (Argon2 in a debug build); that is expected.

> 🧪 **Testing:** test the **refusals** of an authentication system at least as carefully as the success: no token,
> a wrong token, an expired one, a wrong password. A login that accepts everything passes every happy-path test.

## Common mistakes

**1. Storing tokens or passwords in plain text.** A leaked backup, a log file or one SQL (Structured Query Language) injection then gives away
every account. Store Argon2 hashes for passwords and SHA-256 hashes for tokens.

**2. Logging secrets.** `tracing::debug!(?credentials)` writes the password into the log; a logged request header
writes the token. Never log `Credentials`, `Authorization` headers or tokens (Lesson 09.10, Section 8).

**3. Hashing on the async runtime.** Calling `Argon2::default().hash_password(...)` directly in the handler blocks a
runtime thread for every login. Use `spawn_blocking`.

**4. Different answers for "no such user" and "wrong password".** That turns the login form into a tool for user
enumeration. Use one message (and, ideally, the same timing).

**5. Forgetting the expiry check.** `WHERE token_hash = $1` without `AND expires_at > now()` makes every token valid
forever.

**6. Checking only in the frontend.** Hiding the "delete" button for logged-out users protects nothing (Lesson 16.1).
The `AuthUser` parameter on the handler is the real check.

## Try it yourself

1. Log in twice and see two rows in `sessions`. Every device of a user gets its own session.
2. In `psql`, expire your session: `UPDATE sessions SET expires_at = now() - interval '1 second';`, then call `/me`
   with your token: `401`.
3. Register with `"   Bob@Example.COM  "` and check what is stored.
4. Commit: `git add code/p16-tasks && git commit -m "p16-tasks: users, sessions and protected writes"`.

## Exercises

### Exercise 1 — Writes need a token (easy)

Write a `#[sqlx::test]` `writes_need_a_token`: `POST /tasks` without a token must return `401` with
`{"code":"unauthorized","message":"missing or invalid token"}`, and `DELETE /tasks/1` without a token must return `401`.

<details>
<summary>Hint</summary>

`post_json("/tasks", "", ...)` sends `Authorization: Bearer ` with an empty token, which no session matches.

</details>

<details>
<summary>Answer</summary>

```rust
    #[sqlx::test]
    async fn writes_need_a_token(pool: PgPool) {
        let app = test_app(pool).await;
        let (status, body) = send(app.clone(), post_json("/tasks", "", r#"{"title":"x"}"#)).await;
        assert_eq!(status, StatusCode::UNAUTHORIZED);
        assert_eq!(
            body,
            r#"{"code":"unauthorized","message":"missing or invalid token"}"#
        );

        let request = Request::delete("/tasks/1").body(Body::empty()).unwrap();
        let (status, _) = send(app, request).await;
        assert_eq!(status, StatusCode::UNAUTHORIZED);
    }
```

Why it works: the `AuthUser` extractor runs before the handler, so the `DELETE` is refused even though task 1 does not
exist: an anonymous client learns nothing, not even whether the task exists.

</details>

### Exercise 2 — Log out (medium)

Add `DELETE /sessions/current`: it deletes the session of the token that was sent and answers `204`. Afterwards the
token must not work any more. Write a test that logs in, logs out, and checks that `/me` then answers `401`.

<details>
<summary>Hint</summary>

The handler needs the token hash, not only the user id. Add `pub token_hash: Vec<u8>` to `AuthUser` and fill it in
the extractor. Register the route with `axum::routing::delete(auth::logout)`.

</details>

<details>
<summary>Answer</summary>

```rust
// in AuthUser:              pub token_hash: Vec<u8>,
// in from_request_parts:    let token_hash = hash_token(token);
//                           ... use `token_hash` in the query ...
//                           Ok(AuthUser { id: user_id, token_hash })

/// DELETE /sessions/current: log out. The token stops working at once.
pub async fn logout(user: AuthUser, State(state): State<AppState>) -> Result<StatusCode, ApiError> {
    sqlx::query!(
        "DELETE FROM sessions WHERE token_hash = $1",
        user.token_hash
    )
    .execute(&state.pool)
    .await?;
    Ok(StatusCode::NO_CONTENT)
}

// in app():   .route("/sessions/current", axum::routing::delete(auth::logout))

    #[sqlx::test]
    async fn logout_ends_the_session(pool: PgPool) {
        let app = test_app(pool).await;
        let token = login(&app).await;
        let with_token = |request: axum::http::request::Builder| {
            request
                .header(header::AUTHORIZATION, format!("Bearer {token}"))
                .body(Body::empty())
                .unwrap()
        };

        let (status, _) = send(
            app.clone(),
            with_token(Request::delete("/sessions/current")),
        )
        .await;
        assert_eq!(status, StatusCode::NO_CONTENT);

        let (status, _) = send(app, with_token(Request::get("/me"))).await;
        assert_eq!(status, StatusCode::UNAUTHORIZED);
    }
```

Why it works: because sessions live in the database, logging out is one `DELETE`, and it takes effect immediately on
**every** instance of the service. This is the big advantage of server-side sessions over self-contained tokens
(Lesson 10.7), which cannot be cancelled before they expire.

</details>

### Exercise 3 — Private tasks (hard)

Make tasks belong to their creator. Add a migration `add_owner_to_tasks` with
`ALTER TABLE tasks ADD COLUMN owner_id BIGINT REFERENCES users (id) ON DELETE CASCADE;`. Store `user.id` when a task is
created, and make `GET /tasks/{id}` require a login and return only the caller's own tasks (`404` for other people's
tasks, Lesson 16.2). Write a test where Bob cannot read Ana's task. (Doing the same for the list, `PATCH` and `DELETE`
follows the same pattern.)

<details>
<summary>Hint</summary>

The column must allow `NULL` because old tasks have no owner. In SQL, add `AND owner_id = $2` to the `WHERE`. For two
users, split `login` into `login_as(app, credentials)`. Old tests that read `/tasks/1` without a token now get `401`:
give them a token too.

</details>

<details>
<summary>Answer</summary>

```sql
-- Every new task belongs to the user who created it.
-- Old tasks have no owner (NULL), so nobody can read them through the API.
ALTER TABLE tasks ADD COLUMN owner_id BIGINT REFERENCES users (id) ON DELETE CASCADE;
```

```rust
// create_task takes `user: AuthUser` and inserts the owner:
//     "INSERT INTO tasks (title, owner_id) VALUES ($1, $2) RETURNING id, title, done",
//     title,
//     user.id

async fn get_task(
    user: AuthUser,
    State(state): State<AppState>,
    Path(id): Path<i64>,
) -> Result<Json<Task>, ApiError> {
    let task = sqlx::query_as!(
        Task,
        "SELECT id, title, done FROM tasks WHERE id = $1 AND owner_id = $2",
        id,
        user.id
    )
    .fetch_optional(&state.pool)
    .await?;
    task.map(Json).ok_or(ApiError::NotFound(id))
}

    #[sqlx::test]
    async fn users_cannot_read_each_others_tasks(pool: PgPool) {
        let app = test_app(pool).await;
        let ana = login(&app).await;
        let bob = login_as(
            &app,
            r#"{"email":"bob@example.com","password":"another long password"}"#,
        )
        .await;
        send(
            app.clone(),
            post_json("/tasks", &ana, r#"{"title":"Ana's task"}"#),
        )
        .await;

        let get_as = |token: &str| {
            Request::get("/tasks/1")
                .header(header::AUTHORIZATION, format!("Bearer {token}"))
                .body(Body::empty())
                .unwrap()
        };
        let (status, _) = send(app.clone(), get_as(&ana)).await;
        assert_eq!(status, StatusCode::OK);
        let (status, _) = send(app, get_as(&bob)).await;
        assert_eq!(status, StatusCode::NOT_FOUND);
    }
```

`login` becomes `login_as(app, ana_credentials)`, and `login_as` contains the old body. After giving the old tests
that read `/tasks/1` and `/tasks/42` a token, all twelve tests (with Exercises 1 and 2) pass:

```text
test result: ok. 12 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 1.26s
```

Why it works: authorization is now part of the query itself, so there is no way to forget it between "load" and
"check". The answer for Bob is `404`, not `403`, so he does not even learn that task 1 exists. This closes the "number
in the address" hole from Lesson 10.3.

</details>

## Quiz

1. What is the difference between authentication and authorization, and which status codes belong to each?

<details>
<summary>Answer</summary>

Authentication finds out who is calling (failure: `401`). Authorization decides what that caller may do (failure:
`403`, or `404` to hide that the resource exists).

</details>

2. Why does the service store `SHA-256(token)` and not the token?

<details>
<summary>Answer</summary>

Someone who reads the database (a leak, a backup, an SQL injection) gets only hashes and cannot use them as tokens.
A fast hash is enough because a 256-bit random token cannot be guessed.

</details>

3. Why is Argon2 run inside `spawn_blocking`?

<details>
<summary>Answer</summary>

Argon2 is deliberately slow: it keeps the CPU (Central Processing Unit, the processor) busy for tens to hundreds of
milliseconds. On the async runtime it would block a
worker thread and delay other requests; `spawn_blocking` moves it to a thread pool made for blocking work.

</details>

4. Why does `login` answer "wrong email or password" for both an unknown email and a wrong password?

<details>
<summary>Answer</summary>

To avoid user enumeration: different messages would tell an attacker which email addresses have accounts.

</details>

5. How does a handler require a login, and what happens for a request without a valid token?

<details>
<summary>Answer</summary>

It takes an `AuthUser` parameter. The extractor runs before the handler; without a valid, unexpired session it returns
`ApiError::Unauthorized`, the client gets `401`, and the handler does not run.

</details>

6. Why is `email` declared `UNIQUE` in the database although `register` could check first with a `SELECT`?

<details>
<summary>Answer</summary>

Two registrations can arrive at the same time; both `SELECT`s would find nothing and both `INSERT`s would succeed.
The `UNIQUE` constraint makes the database refuse the second one, and the handler turns that into `409`.

</details>

## Summary

- Registration stores an Argon2id hash (a PHC string with salt and parameters); login verifies it. Hashing runs in
  `spawn_blocking`.
- A login creates a 32-byte random token; the database stores only its SHA-256 hash with an expiry; the client sends
  it as `Authorization: Bearer <token>`.
- One error message for every login failure; unique constraints decide races; limits on password length.
- A custom `FromRequestParts` extractor (`AuthUser`) turns "requires a login" into a handler parameter; its rejection
  is a JSON `401`.
- Server-side sessions can be ended at once (logout = `DELETE`); authorization belongs in the query (`owner_id = $2`).
- Never log or store passwords or tokens in plain text; use TLS and rate limiting in production.

## Sources and further reading

- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — Argon2id parameters, salts and peppers.
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html) — token length, expiry, logout and storage.
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html) — error messages, user enumeration and login throttling.
- [argon2 crate](https://docs.rs/argon2/0.6) — the API (Application Programming Interface) used here, with hashing and verification examples.
- [axum `FromRequestParts`](https://docs.rs/axum/0.8/axum/extract/trait.FromRequestParts.html) — writing your own extractors.
- [RFC 6750 — Bearer Token Usage](https://www.rfc-editor.org/rfc/rfc6750) — the standard for `Authorization: Bearer`.

---

[🏠 Start](../START-HERE.md) · [← 16.7 Migrations and transactions](07-migrations-and-transactions.md) · [📋 Phase 16](README.md) · [16.9 gRPC with tonic →](09-grpc-with-tonic.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
