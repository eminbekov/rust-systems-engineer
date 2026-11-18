<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.7 Authorization, sessions and tokens](07-authorization-sessions-and-tokens.md) · [📋 Phase 10](README.md) · [10.9 Cryptography I: hashes, MACs and randomness →](09-cryptography-hashes-macs-and-randomness.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/08-secrets-management.md) <!-- nav -->

# Lesson 10.8 — Secrets management

> **Phase 10 — Security fundamentals** · Lesson 8 of 15 · ⏱ about 135 minutes

## What you will learn

- What counts as a secret and the places where secrets leak most often.
- How a program should receive secrets: files, environment variables, never code or command-line arguments.
- How to keep a secret out of logs with the `secrecy` crate, and how to wipe memory with `zeroize`.
- How to load a secret safely, including a check of file permissions, with tests.
- What to do when a secret leaks: rotate first, clean up second.

## Before you start

- You finished [Lesson 10.7 — Authorization, sessions and tokens](07-authorization-sessions-and-tokens.md).
- You know the rules about secrets and Git from [Lesson 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md) and [Lesson 01.5](../01-git-and-version-control/05-undoing-changes-safely.md), and configuration through the environment from [Lesson 08.15](../08-software-design/15-the-twelve-factor-app.md).
- Docker Desktop is running (one small experiment uses a Linux container). The container is named with the prefix `course-` and removed automatically when it ends (`--rm`).
- Create the package: `cargo new --lib code/p10-secrets`, then inside it `cargo add secrecy@0.10 zeroize@1.9`, `cargo add --dev tempfile@3.27` and `mkdir src/bin`.

## New words

| Word | Meaning |
|---|---|
| **secret** | A value that gives access or power to whoever knows it: a password, an API key, a token, a private key, a database connection string. |
| **API key** (Application Programming Interface key) | A secret text that identifies and authorises a program when it calls another service. |
| **environment variable** | A named text value that the operating system gives to every program it starts (`HOME`, `PATH`). Programs read them with `std::env::var`. |
| **process** | A running program. |
| **rotation** | Replacing a secret with a new one, on a schedule or after a leak, and cancelling the old one. |
| **revoke** | Cancel a secret at the service that issued it, so it stops working. |
| **secret manager** | A service that stores secrets encrypted, controls who may read them, and records every access (examples: HashiCorp Vault, cloud key services, Kubernetes Secrets with encryption). |
| **file mode** (permissions) | The numbers that say who may read, write or run a file. `0o600` means: the owner can read and write, nobody else can do anything. |
| **zeroize** | To overwrite memory with zeros so that a secret does not stay there after use. |
| **blast radius** | How much damage one leaked secret can do. Smaller is better. |
| **CI** (Continuous Integration) | A server that runs your checks and builds automatically on every push (Phase 09). |
| **PID** (Process IDentifier) | The number that the operating system gives to a running program. |
| **root** | The all-powerful administrator user of a Linux system. |

## 1. What is a secret, and where does it leak?

A **secret** is any value that gives access: passwords, API keys, tokens (Lesson 10.7), the signing key of a JWT (JSON Web Token),
private keys, database passwords. Notice what is **not** a secret: a user name, the address of a server, the algorithm
you use (open design, Lesson 10.1).

Secrets rarely leak through clever attacks. They leak through ordinary places:

| Place | Why it leaks |
|---|---|
| **Source code** | Everyone who can read the repository can read the key, forever ([Lesson 01.3](../01-git-and-version-control/03-your-first-repository-and-commits.md)). |
| **Git history** | A deleted file is still in the history ([Lesson 01.5](../01-git-and-version-control/05-undoing-changes-safely.md)). |
| **Logs and error messages** | `{:?}` prints every field (Lesson 10.1). Logs are copied, shipped and read by many people. |
| **Command-line arguments** | The list of processes shows them to other users of the machine. |
| **Environment variables** | Child processes inherit them; they appear in crash reports, debug pages and, on Linux, in `/proc/<pid>/environ`. |
| **Container images** | `ENV` and `COPY` write the secret into a layer that anyone with the image can read. |
| **Backups, chat, tickets, screenshots** | People paste secrets where they should not. |

A small experiment makes this concrete. It runs in a Linux container that is removed at the end, and the
"secrets" are made-up values.

```bash
docker run --rm --name course-10-08-leaks ubuntu:26.04 bash -c '
bash -c "sleep 60; true" myapp --password=hunter2-demo &
sleep 1
export APP_TOKEN=demo-token-12345
bash -c "sleep 60; true" &
CH=$!
sleep 1
ps -eo pid,args | grep -v grep | grep myapp
tr "\0" "\n" < /proc/$CH/environ | grep APP_TOKEN
umask 077; echo -n s3cret > /tmp/secret.txt; ls -l /tmp/secret.txt | cut -c1-10
'
```

```text
      7 bash -c sleep 60; true myapp --password=hunter2-demo
APP_TOKEN=demo-token-12345
-rw-------
```

**What happens:**

1. The first `bash -c "sleep 60; true" myapp --password=hunter2-demo` imitates a program that was started as
   `myapp --password=...`. The command `ps -eo pid,args` lists every process **with its arguments**, so the password
   is visible to every user of the machine. (On a shared server this is a real leak.)
2. The second process was started with `APP_TOKEN` in its environment. The file `/proc/<pid>/environ` shows the
   environment of a process on Linux. The owner of the process and `root` can read it; so can anything that
   inherits the environment.
3. After `umask 077` (a setting that removes all rights for the group and for others from new files) the new
   file has mode `-rw-------`: only the owner can read it. This is the safest of the three.

A secret written into a container image is even worse: `ENV API_KEY=...` in a Dockerfile stays in the image
metadata, and `docker inspect` and `docker history` show it to anyone who can pull the image. You will build
images properly in [Lesson 11.13](../11-linux-shell-and-containers/13-dockerfiles-for-rust.md); keep secrets out of them.

## 2. How a program should get a secret

From best to worst:

1. **A secret manager or the platform** gives the program a short-lived secret, or mounts it as a **file** inside
   the container (Docker secrets and Kubernetes Secrets do this). The program receives only the path.
2. **A file** with mode `0600`, outside the repository. The program checks the mode before it reads.
3. **An environment variable**, set by the platform at start. It is simple and common, but see the table above.
4. **Never:** source code, a committed `.env` file, a command-line argument, an image layer.

A good habit from Docker: the program accepts **both** `NAME` and `NAME_FILE`. If `NAME_FILE` is set, the secret is
read from that file; otherwise from `NAME`. Then the same program works on your laptop (a variable) and in
production (a mounted file).

## 3. Loading a secret in Rust

Start with the main risk, the accidental print. Save `src/bin/leaky.rs`:

```rust
use secrecy::{ExposeSecret, SecretString};

#[derive(Debug)]
struct PlainConfig {
    user: String,
    password: String,
}

#[derive(Debug)]
struct SafeConfig {
    user: String,
    password: SecretString,
}

fn main() {
    let plain = PlainConfig {
        user: "app".into(),
        password: "hunter2-demo".into(),
    };
    println!(
        "user {} has a password of {} characters",
        plain.user,
        plain.password.len()
    );
    println!("plain: {plain:?}");

    let safe = SafeConfig {
        user: "app".into(),
        password: SecretString::from("hunter2-demo"),
    };
    println!("safe:  {safe:?}");

    // To use the value you must say so, and the word is easy to search for in a code review:
    println!("length: {}", safe.password.expose_secret().len());
    println!("user: {}", safe.user);
}
```

```text
user app has a password of 12 characters
plain: PlainConfig { user: "app", password: "hunter2-demo" }
safe:  SafeConfig { user: "app", password: SecretBox<str>([REDACTED]) }
length: 12
user: app
```

**What happens:**

1. `PlainConfig` stores the password as a `String`. `#[derive(Debug)]` prints it (as in Lesson 10.1). `SafeConfig` has the same shape
   but the password is a `SecretString`.
2. `SecretString` from the `secrecy` crate is a wrapper around text. Its `Debug` prints only `[REDACTED]`. It
   has **no** `Display`, and it does not implement `Serialize` or `Clone` unless you ask, so it is hard to copy or
   print by mistake. It also overwrites the text with zeros when it is dropped.
3. To read the value you must call `expose_secret()` (from the `ExposeSecret` trait). The word is easy to search for in a
   code review: every use is a place to ask "is this safe?". Keep these places few and short.

Now the library. Save `src/lib.rs`:

```rust
use std::fmt;
use std::fs;
use std::path::PathBuf;

use secrecy::SecretString;

#[derive(Debug, PartialEq, Eq)]
pub enum SecretError {
    Missing(String),
    Unreadable(PathBuf),
    TooPermissive(PathBuf),
    Empty(String),
}

// The messages name the secret and the file, never the value.
impl fmt::Display for SecretError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            SecretError::Missing(name) => {
                write!(f, "secret {name} is not set (set {name} or {name}_FILE)")
            }
            SecretError::Unreadable(path) => write!(f, "cannot read {}", path.display()),
            SecretError::TooPermissive(path) => write!(
                f,
                "{} can be read by other users; run: chmod 600 {}",
                path.display(),
                path.display()
            ),
            SecretError::Empty(name) => write!(f, "secret {name} is empty"),
        }
    }
}

impl std::error::Error for SecretError {}

/// A secret file must not be readable by the group or by everybody.
#[cfg(unix)]
fn check_permissions(path: &PathBuf) -> Result<(), SecretError> {
    use std::os::unix::fs::PermissionsExt;
    let meta = fs::metadata(path).map_err(|_| SecretError::Unreadable(path.clone()))?;
    if meta.permissions().mode() & 0o077 != 0 {
        return Err(SecretError::TooPermissive(path.clone()));
    }
    Ok(())
}

#[cfg(not(unix))]
fn check_permissions(_path: &PathBuf) -> Result<(), SecretError> {
    Ok(())
}

/// Finds the secret `name`. `env` looks up an environment variable (a parameter, so tests need no real variables).
/// Order: 1. the file named by `{name}_FILE`   2. the variable `{name}`.
pub fn load_secret(
    name: &str,
    env: impl Fn(&str) -> Option<String>,
) -> Result<SecretString, SecretError> {
    let value = if let Some(path) = env(&format!("{name}_FILE")) {
        let path = PathBuf::from(path);
        check_permissions(&path)?;
        let text = fs::read_to_string(&path).map_err(|_| SecretError::Unreadable(path.clone()))?;
        text.trim_end_matches(['\n', '\r']).to_string()
    } else if let Some(value) = env(name) {
        value
    } else {
        return Err(SecretError::Missing(name.to_string()));
    };
    if value.is_empty() {
        return Err(SecretError::Empty(name.to_string()));
    }
    Ok(SecretString::from(value))
}

/// Settings of a program. `Debug` is safe: the password prints as [REDACTED].
#[derive(Debug)]
pub struct DbConfig {
    pub user: String,
    pub password: SecretString,
}
```

**What happens:**

1. `SecretError` has four cases. Its `Display` text names the secret and the file, and **never the value**.
   The message even says how to fix the problem (`chmod 600`).
2. `check_permissions` is compiled only on Unix (`#[cfg(unix)]`; macOS and Linux). `mode() & 0o077` keeps
   the six permission bits of the **group** and **others**; if any is set, others can read the file, and we refuse. On other systems the
   second version of the function does nothing.
3. `load_secret` takes the name and a function `env` that looks up variables. In `main` we pass the real
   `std::env::var`; in tests we pass a fake. (Setting real environment variables in tests is unsafe in Rust
   2024 and makes tests interfere with each other, so injecting the lookup is the cleaner design. This is the same idea
   as passing `now` in Lesson 10.7.)
4. It tries `{name}_FILE` first. The file is checked, read, and the final newline is removed (editors and `echo` add one; a
   password with a hidden newline is a classic bug). Otherwise it uses the variable `{name}`, and if neither
   exists it returns `Missing`.
5. An empty value is refused: it almost always means a mistake in the configuration.
6. The result is a `SecretString`, so the secret has a safe wrapper from the first moment.
7. `DbConfig` holds a normal `user` and a secret `password`. Its derived `Debug` is now safe.

Save `src/bin/load.rs`; it reads `DB_PASSWORD` or `DB_PASSWORD_FILE` from the real environment:

```rust
use p10_secrets::{DbConfig, SecretError, load_secret};
use secrecy::ExposeSecret;

fn run() -> Result<(), SecretError> {
    let env = |name: &str| std::env::var(name).ok();

    let config = DbConfig {
        user: String::from("app"),
        password: load_secret("DB_PASSWORD", env)?,
    };

    println!("loaded: {config:?}");
    println!(
        "the password has {} characters",
        config.password.expose_secret().len()
    );
    Ok(())
}

fn main() {
    if let Err(error) = run() {
        eprintln!("error: {error}");
        std::process::exit(1);
    }
}
```

Build it, then try the four situations. Each command is a separate run, and the outputs are listed in the same order below:

```bash
cargo build --bin load
./target/debug/load
DB_PASSWORD=hunter2-demo ./target/debug/load
mkdir -p sandbox && printf 'hunter2-demo\n' > sandbox/db_password && chmod 644 sandbox/db_password
DB_PASSWORD_FILE=sandbox/db_password ./target/debug/load
chmod 600 sandbox/db_password
DB_PASSWORD_FILE=sandbox/db_password ./target/debug/load
rm -r sandbox
```

```text
error: secret DB_PASSWORD is not set (set DB_PASSWORD or DB_PASSWORD_FILE)
loaded: DbConfig { user: "app", password: SecretBox<str>([REDACTED]) }
the password has 12 characters
error: sandbox/db_password can be read by other users; run: chmod 600 sandbox/db_password
loaded: DbConfig { user: "app", password: SecretBox<str>([REDACTED]) }
the password has 12 characters
```

**What happens:**

1. With nothing set, the program fails at start with a clear message and exit code 1. It does not run "half configured".
2. With `DB_PASSWORD` the secret loads. `{config:?}` shows `[REDACTED]`. The password has 12 characters (the
   trailing newline in the file case was removed).
3. The file with mode `644` is refused: other users on the machine could read it.
4. After `chmod 600` the file is accepted.

Finally, the tests, at the end of `src/lib.rs`:

```rust
#[cfg(test)]
mod tests {
    use std::collections::HashMap;

    use secrecy::ExposeSecret;

    use super::*;

    type EnvFn = Box<dyn Fn(&str) -> Option<String>>;

    fn env_of(pairs: &[(&str, &str)]) -> EnvFn {
        let map: HashMap<String, String> = pairs
            .iter()
            .map(|(k, v)| (k.to_string(), v.to_string()))
            .collect();
        Box::new(move |name| map.get(name).cloned())
    }

    #[test]
    fn a_missing_secret_is_an_error_that_names_it() {
        let error = load_secret("DB_PASSWORD", env_of(&[])).unwrap_err();
        assert_eq!(error, SecretError::Missing("DB_PASSWORD".into()));
        assert!(error.to_string().contains("DB_PASSWORD_FILE"));
    }

    #[test]
    fn the_variable_works_and_empty_is_refused() {
        let secret = load_secret("K", env_of(&[("K", "value")])).unwrap();
        assert_eq!(secret.expose_secret(), "value");
        assert_eq!(
            load_secret("K", env_of(&[("K", "")])).unwrap_err(),
            SecretError::Empty("K".into())
        );
    }

    #[cfg(unix)]
    #[test]
    fn a_file_wins_over_the_variable_and_must_be_private() {
        use std::os::unix::fs::PermissionsExt;

        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("secret.txt");
        fs::write(&path, "from-file\n").unwrap();
        let path_text = path.to_str().unwrap().to_string();
        let env = env_of(&[("K_FILE", &path_text), ("K", "from-variable")]);

        fs::set_permissions(&path, fs::Permissions::from_mode(0o644)).unwrap();
        assert_eq!(
            load_secret("K", &env).unwrap_err(),
            SecretError::TooPermissive(path.clone())
        );

        fs::set_permissions(&path, fs::Permissions::from_mode(0o600)).unwrap();
        let secret = load_secret("K", &env).unwrap();
        assert_eq!(secret.expose_secret(), "from-file"); // the newline was removed
    }

    #[test]
    fn debug_output_never_contains_the_secret() {
        let config = DbConfig {
            user: "app".into(),
            password: SecretString::from("hunter2-demo"),
        };
        let shown = format!("{config:?} {config:#?}");
        assert!(!shown.contains("hunter2"));
        assert!(shown.contains("REDACTED"));
    }
}
```

```bash
cargo test
```

```text
running 4 tests
test tests::a_missing_secret_is_an_error_that_names_it ... ok
test tests::debug_output_never_contains_the_secret ... ok
test tests::the_variable_works_and_empty_is_refused ... ok
test tests::a_file_wins_over_the_variable_and_must_be_private ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

**What happens:** the helper `env_of` builds a fake environment from a list of pairs, boxed as a function.
The tests cover a missing secret (the error names `DB_PASSWORD_FILE`), a good variable and an empty one, a file that
wins over the variable and is refused at mode `644` but accepted at `600` (with the newline removed; `tempfile` creates a folder that is
deleted afterwards), and, most important, that the `Debug` output of the config never contains the secret, in both
the compact `{:?}` and the pretty `{:#?}` form.

> 🧪 **Testing:** a test that formats your secret-holding types and checks that the secret is **not** in the output
> is a cheap guard against the most common leak. Write one for every type that holds a secret.

## 4. Wiping memory with `zeroize`

When a program has finished with a secret, the bytes stay in memory until something overwrites them. A memory
dump, a swap file or a bug that reads freed memory could expose them. The `zeroize` crate overwrites memory in a way that
the compiler is **not allowed to optimise away** (an ordinary `for` loop writing zeros may be removed, because
nobody reads the memory afterwards). Save `src/bin/zero.rs`:

```rust
use zeroize::{Zeroize, Zeroizing};

fn main() {
    // Explicit wiping of a buffer:
    let mut key = [7u8; 8];
    println!("before: {key:?}");
    key.zeroize();
    println!("after:  {key:?}");

    // `Zeroizing<T>` wipes the value for you when it goes out of scope.
    {
        let token = Zeroizing::new(String::from("temporary-token"));
        println!(
            "inside the block the value is usable: {} bytes",
            token.len()
        );
    } // <- the memory of `token` is overwritten with zeros here
    println!("the block ended");
}
```

```text
before: [7, 7, 7, 7, 7, 7, 7, 7]
after:  [0, 0, 0, 0, 0, 0, 0, 0]
inside the block the value is usable: 15 bytes
the block ended
```

**What happens:**

1. `key.zeroize()` (from the `Zeroize` trait) overwrites all eight bytes with zeros.
2. `Zeroizing::new(value)` wraps any value that can be zeroized. When the wrapper is dropped at the end of the block,
   it wipes the value for you. It behaves like the inner value (`token.len()` works).

> ⚠️ **Common mistake:** believing that `zeroize` makes secrets unreadable in memory. It wipes **the copy you hold**.
> Copies made earlier (a `String` that grew and moved, a buffer in the network library, the operating
> system's swap file) are not wiped. Treat it as one layer of defense in depth, not a guarantee. The best strategy is to
> keep secrets short-lived, in few places, and never to clone them. `SecretString` already uses `zeroize` for you.

## 5. Writing a secret file safely

If your program creates a file with a secret, create it **private from the first byte**: if you create the file first
and call `chmod` afterwards, other users can read it in the gap. Exercise 3 asks you to write this function.

## 6. When a secret leaks

Treat every leak as a theft, even if "nobody saw it". Do these steps **in this order**:

1. **Revoke and replace** the secret at the service (delete the API key, change the password, rotate the
   signing key). This is the only step that really stops the harm.
2. **Find out what was exposed**: check the logs of the service for use of the old key.
3. **Remove the secret** from where it leaked (code, history with a tool such as `git filter-repo`, chat). This is
   cleanup, not a cure: copies may exist already.
4. **Learn**: add the check that would have caught it.

**Prevention** costs little: put `.env`, `*.pem` and `*.key` in `.gitignore` before the first commit; commit a
`.env.example` with fake values; turn on your Git host's secret scanning (GitHub does it for known token formats) and
run a scanner such as `gitleaks` in CI ([Lesson 09.12](../09-testing-debugging-benchmarking/12-continuous-integration.md))
or as a pre-commit hook.

## 7. Good habits

- **One secret per purpose and per environment**, so that a leaked test key cannot open production (a small **blast radius**).
- **Least privilege**: a database account for the app needs only its own tables.
- **Rotate** on a schedule and after any leak, and prefer **short-lived** credentials.
- **Do not log, print or send secrets in URLs** (URLs are stored in logs and browser history), and do not paste them into chat.

## Common mistakes

> ⚠️ **Common mistake:** `println!("{:?}", config)` on a type that holds a secret field of type `String`.

> ⚠️ **Common mistake:** base64 is not encryption. A Kubernetes Secret or a CI variable that is only Base64 can be read
> by anyone who can read it. Check that your platform encrypts secrets at rest and limits who can read them.

## Try it yourself

1. Type the library, the three programs and the tests. Run `cargo test` and the experiments of Section 3.
2. Run `load` with `DB_PASSWORD=""` (an empty value). Which error do you get?
3. Write a struct of your own with a field `api_key: SecretString`, derive `Debug`, and prove with a test
   that `{:#?}` hides it.

## Exercises

### Exercise 1 — Where should it live? (easy)

Choose the best place for each secret, and say why: (a) a database password for a program in a container on a
server; (b) a test API key for local development; (c) the signing key of a JWT that all services must share;
(d) a password that a script needs once.

<details>
<summary>Hint</summary>

Use the order of Section 2. Think about who can read each place.

</details>

<details>
<summary>Answer</summary>

(a) A file mounted by the platform (Docker or Kubernetes secret) and read with `DB_PASSWORD_FILE`, not an
image layer. (b) An environment variable or a `0600` file outside the repository, with a `.env.example` of fake
values committed. (c) A secret manager, delivered to each service as a file or at start, with a rotation plan.
(d) Ask for it at a prompt without echo, or read it from a `0600` file; never put it in an argument, because it would
show in the process list.

</details>

### Exercise 2 — A missing file (medium)

Write a **test** that proves `load_secret` returns `SecretError::Unreadable(path)` (and does not panic) when
`NAME_FILE` points to a file that does not exist.

<details>
<summary>Hint</summary>

Use `env_of(&[("K_FILE", "/definitely/not/here")])`. `check_permissions` calls `fs::metadata`, which fails.

</details>

<details>
<summary>Answer</summary>

```rust
    #[test]
    fn a_missing_file_is_unreadable_not_a_panic() {
        let env = env_of(&[("K_FILE", "/definitely/not/here")]);
        assert_eq!(
            load_secret("K", env).unwrap_err(),
            SecretError::Unreadable(PathBuf::from("/definitely/not/here"))
        );
    }
```

Put it inside `mod tests`. Why it works: the permission check asks the file system for the file's data first and
turns the failure into `Unreadable`, so the program reports a clear error instead of crashing.

</details>

### Exercise 3 — A private file from the first byte (hard)

Write `write_secret_file(path: &Path, text: &str) -> io::Result<()>` for Unix. It must create the file with mode
`0o600`, and **fail** if the file already exists. Write a **test** that checks the mode and that a second call fails and leaves the content unchanged.

<details>
<summary>Hint</summary>

`std::fs::OpenOptions::new().write(true).create_new(true).mode(0o600).open(path)`; the `mode` method comes from
`std::os::unix::fs::OpenOptionsExt`. Read the mode back with `metadata(...).permissions().mode() & 0o777`.

</details>

<details>
<summary>Answer</summary>

```rust
#[cfg(unix)]
pub fn write_secret_file(path: &std::path::Path, text: &str) -> std::io::Result<()> {
    use std::io::Write;
    use std::os::unix::fs::OpenOptionsExt;

    let mut file = fs::OpenOptions::new()
        .write(true)
        .create_new(true) // never overwrite an existing file
        .mode(0o600) // owner: read and write; group and others: nothing
        .open(path)?;
    file.write_all(text.as_bytes())
}
```

```rust
    #[cfg(unix)]
    #[test]
    fn secret_files_are_created_private_and_never_overwritten() {
        use std::os::unix::fs::PermissionsExt;

        let dir = tempfile::tempdir().unwrap();
        let path = dir.path().join("token");
        write_secret_file(&path, "abc\n").unwrap();
        let mode = fs::metadata(&path).unwrap().permissions().mode() & 0o777;
        assert_eq!(mode, 0o600);
        assert!(write_secret_file(&path, "other").is_err());
        assert_eq!(fs::read_to_string(&path).unwrap(), "abc\n");
    }
```

Why it works: the mode is given to the operating system **when the file is created**, so there is no moment when it
is readable by others. `create_new(true)` makes the call fail with an error if the file exists, so an older file (or a
link planted by an attacker) is never overwritten.

</details>

## Quiz

1. Why is a command-line argument a bad place for a password?

<details>
<summary>Answer</summary>

The list of processes shows arguments to other users of the machine, and they may end up in shell history and logs.

</details>

2. What does `SecretString` protect against, and what does it not?

<details>
<summary>Answer</summary>

It protects against accidental printing (`Debug` shows `[REDACTED]`, no `Display`) and wipes its memory on drop. It
does not protect against code that calls `expose_secret()` and prints the value, nor against copies made elsewhere.

</details>

3. What is the first thing to do after a secret was committed to Git?

<details>
<summary>Answer</summary>

Revoke and replace the secret at the service that issued it. Cleaning the history comes after, because copies of the old
value may already exist.

</details>

4. Why does `load_secret` take the environment lookup as a parameter?

<details>
<summary>Answer</summary>

So tests can give a fake environment without changing real process-wide variables, which is unsafe in Rust 2024 and
makes tests depend on each other.

</details>

## Summary

- A secret is anything that gives access; it leaks mostly through ordinary places: code, Git history, logs, arguments, environment, images.
- Prefer files provided by the platform (mode `0600`) or a secret manager; avoid arguments and images; never commit secrets.
- Accept both `NAME` and `NAME_FILE`, fail fast at start, and refuse empty values and files others can read.
- Hold secrets in `SecretString` (hidden `Debug`, wiped on drop); call `expose_secret()` rarely and briefly.
- `zeroize` wipes the copy you hold; it is one layer, not a guarantee.
- Create secret files private from the first byte (`mode(0o600)`, `create_new`).
- When a secret leaks: revoke and replace first, investigate, clean up, and add a check.
- Separate secrets per environment, use least privilege, rotate, and prefer short-lived credentials.

## Sources and further reading

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html) — storage, rotation, access control and what to do after a leak.
- [The Twelve-Factor App — Config](https://12factor.net/config) — why configuration and secrets come from the environment and not from code.
- [`secrecy` crate documentation](https://docs.rs/secrecy/0.10.3/secrecy/) — `SecretString`, `SecretBox` and `ExposeSecret`.
- [`zeroize` crate documentation](https://docs.rs/zeroize/1.9.1/zeroize/) — `Zeroize`, `Zeroizing` and the limits of wiping memory.
- [Docker — Manage sensitive data with Docker secrets](https://docs.docker.com/engine/swarm/secrets/) — how a platform mounts secrets as files.
- [Kubernetes — Secrets](https://kubernetes.io/docs/concepts/configuration/secret/) — how Secrets are stored and the good practices around them.
- [GitHub Docs — About secret scanning](https://docs.github.com/en/code-security/secret-scanning/introduction/about-secret-scanning) — automatic detection of leaked tokens.
- [gitleaks](https://github.com/gitleaks/gitleaks) — a scanner for secrets in Git repositories.
- [Rust — `std::os::unix::fs::OpenOptionsExt`](https://doc.rust-lang.org/std/os/unix/fs/trait.OpenOptionsExt.html) — setting the file mode when a file is created.

---

[🏠 Start](../START-HERE.md) · [← 10.7 Authorization, sessions and tokens](07-authorization-sessions-and-tokens.md) · [📋 Phase 10](README.md) · [10.9 Cryptography I: hashes, MACs and randomness →](09-cryptography-hashes-macs-and-randomness.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
