<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.13 Supply-chain security: cargo-audit and cargo-deny](13-supply-chain-security.md) · [📋 Phase 10](README.md) · [10.15 Checkpoint →](15-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/14-project-encrypted-secrets-vault.md) <!-- nav -->

# Lesson 10.14 — Project: an encrypted secrets vault

> **Phase 10 — Security fundamentals** · Lesson 14 of 15 · ⏱ about 8 hours (4 sittings)

## What you will learn

- How to combine a password hash (Argon2id), authenticated encryption (XChaCha20-Poly1305), secure randomness and zeroizing into one working tool.
- How to design a small binary file format with a version, a salt, a nonce and parameters, and how to read it safely when the file is untrusted.
- How to test security properties: wrong password, tampering with every byte, garbage input.
- How to keep secrets out of logs, command lines, error messages and (as far as Rust allows) memory.
- How to write a short threat model and a README for a security tool.

This is a **project lesson**. It does not give you the finished program. It gives you the target (the contract), a plan with milestones, hints, and commands that prove each milestone is done.

## Before you start

- You finished [Lesson 10.13 — Supply-chain security](13-supply-chain-security.md) and the earlier lessons of this phase. You will use Argon2 ([10.6](06-authentication-and-password-hashing.md)), secrets handling ([10.8](08-secrets-management.md)), hashes and randomness ([10.9](09-cryptography-hashes-macs-and-randomness.md)), AEAD ([10.10](10-cryptography-encryption-and-signatures.md)) and input limits ([10.12](12-memory-safety-and-reviewing-unsafe.md)).
- You know CLI (Command-Line Interface) tools with `clap`, error types with `thiserror`, and end-to-end tests ([Lesson 05.11](../05-rust-intermediate/11-practical-crates-serde-clap-errors.md), [Lesson 09.2](../09-testing-debugging-benchmarking/02-testing-command-line-programs.md)).
- Git works. Your project goes to `~/projects/rust/code/p10-vault`, inside your learning repository.
- You are online: Cargo must download crates. **Read the documentation of the exact version** of `argon2`, `chacha20poly1305` and `zeroize` on docs.rs (links at the end): their APIs changed recently and old examples do not compile.

> 🔒 **Ethics and safety:** this is a learning project. It teaches the design of an encrypted store; it has not been audited and must not protect real production secrets. Use throwaway passwords and fake secrets such as `hunter2`.

## New words

| Word | Meaning |
|---|---|
| **vault** | A file that stores named secrets in encrypted form. |
| **master password** | The one password from which the encryption key is derived. If it is forgotten, the data cannot be recovered. |
| **KDF parameters** (KDF, key derivation function: it turns a password into a key) | The cost settings of Argon2: memory (`m_cost`, in KiB, kibibytes), passes (`t_cost`) and parallelism (`p_cost`). Higher cost makes guessing slower for an attacker, and unlocking slower for you. |
| **file header** | The first bytes of a file, describing its format (here: magic bytes, version, KDF parameters, salt). |
| **magic bytes** | A fixed sequence at the start of a file that says what type of file it is (`VLT1` here). |
| **atomic write** | Writing a file so that a crash leaves either the complete old file or the complete new file, never a half-written one. Done with a temporary file and `rename`. |
| **threat model** | A short list of what you protect, against whom, and what you do **not** protect against (Lesson 10.2). |
| **`Zeroizing<T>`** | A wrapper from the `zeroize` crate that overwrites the value with zeros when it is dropped. |
| **permissions `0600`** | A Unix file mode: only the owner may read and write the file. Permissions are explained in Lesson 11.4; here you only need to set this one mode. |

## The project

### What you will build

`vault` is a command-line tool that keeps named secrets (API, Application Programming Interface, tokens; passwords) in one encrypted file:

```text
vault init                 create an empty vault, asking for a master password
vault add github-token     read the secret from standard input, store it under that name
vault get github-token     print the secret
vault list                 print the names of all secrets (never the values)
vault remove github-token  delete a secret
```

Everything is protected by one master password: **Argon2id** turns it (with a random salt) into a 32-byte key, and **XChaCha20-Poly1305** encrypts the whole list of secrets and detects any change to the file.

### Contract

**Command line**

```text
vault [--file <PATH>] <COMMAND>
```

| Command | Behaviour | Output on success |
|---|---|---|
| `init` | Creates the vault file. Fails if it exists. | `Created <path>` |
| `add <NAME>` | Reads the secret from standard input (one value; a trailing newline is removed). If `NAME` exists it is replaced. | `Added <name>` or `Updated <name>` |
| `get <NAME>` | Prints the secret followed by a newline. | the secret |
| `list` | Prints the names, sorted, one per line. | names |
| `remove <NAME>` | Deletes the secret. | `Removed <name>` |

- `--file` (or the environment variable `VAULT_FILE`) is the path of the vault. The default is `vault.bin` in the current folder.
- **Password:** if the environment variable `VAULT_PASSWORD` is set, use it (this is for scripts and tests, and it is the weaker way: Lesson 10.8). Otherwise ask on the terminal **without echo** (crate `rpassword`). `init` asks twice and refuses if the two answers differ.
- **Secret value for `add`:** if standard input is a terminal, prompt without echo; otherwise read it from the pipe. Secrets are never taken from command-line arguments (they would show up in `ps`, the command that lists running programs with their arguments, and in the shell history).
- **Exit codes:** `0` success; `1` any error from the tool; `2` wrong command-line usage (this is what `clap` does by default). Errors go to standard error as `error: <message>`.
- **Names:** 1 to 64 characters from `A-Z a-z 0-9 . _ -`.
- **Messages** (use exactly these, the "Done when" commands show them):

| Situation | Message on standard error |
|---|---|
| Wrong password **or** a modified file | `error: wrong password, or the vault file was modified` |
| First four bytes are not `VLT1` | `error: not a vault file (bad magic bytes)` |
| Unknown version byte | `error: unsupported vault version 7` |
| File shorter than the minimum or larger than the limit | `error: the vault file is too short or too large` |
| No such secret | `error: no secret named "x"` |
| `init` and the file exists | `error: <path> already exists` |
| Bad name | `error: invalid secret name "bad name!" (use 1-64 characters from A-Z a-z 0-9 . _ -)` |

(Why one message for "wrong password" and "modified"? The authentication tag cannot tell them apart, and a program that tried would only help an attacker.)

**File format, version 1** (all numbers big-endian)

| Bytes | Size | Content |
|---|---|---|
| 0..4 | 4 | magic `VLT1` |
| 4 | 1 | format version `1` |
| 5..9 | 4 | Argon2 `m_cost` in KiB |
| 9..13 | 4 | Argon2 `t_cost` |
| 13..17 | 4 | Argon2 `p_cost` |
| 17..33 | 16 | salt (random) |
| 33..57 | 24 | XChaCha20-Poly1305 nonce (random) |
| 57.. | n + 16 | ciphertext of the secrets, followed by the 16-byte tag |

- **Plaintext:** a JSON (JavaScript Object Notation) object `{"name": "value", ...}` (use `serde_json`). An empty vault is `{}`, so an empty vault file is `33 + 24 + 2 + 16 = 75` bytes.
- **Associated data:** bytes `0..33` (the header). The tag therefore also covers the version, the parameters and the salt.
- **Defaults:** `m_cost = 19456` (19 MiB), `t_cost = 2`, `p_cost = 1` (a commonly recommended minimum for Argon2id). Tests may use cheap parameters (`8`, `1`, `1`) to run fast.
- **A fresh random salt and a fresh random nonce at every save.** The key is derived again at every save, from the same password and the new salt.
- **Limits** (Lesson 10.12): a vault file larger than 1 MiB is rejected before it is read; a secret larger than 64 KiB is rejected; when reading, the Argon2 parameters from the file must be reasonable (`m_cost` at most 1,048,576 KiB, `t_cost` at most 20, `p_cost` at most 16), because they come from an **untrusted file** and could otherwise force the program to use gigabytes of memory.

**Security requirements**

1. The master password, the derived key, the decrypted plaintext and the secret read from standard input live in `Zeroizing` containers (or are zeroized by hand) and are not copied needlessly.
2. Secret values appear in output only for `get`. They never appear in error messages, `Debug` output or logs.
3. The file is created with permissions `0600` and replaced **atomically** (temporary file in the same folder, `sync_all`, `rename`).
4. No input can make the program panic: a missing, empty, truncated, random or modified file is always an error with a message and exit code `1`.
5. Dependencies are checked with `cargo audit` and `cargo deny check` (Lesson 10.13), and `deny.toml` is committed.

**Crates:** `clap` (features `derive`, `env`), `argon2`, `chacha20poly1305`, `getrandom`, `zeroize`, `serde_json`, `thiserror`, `rpassword`; dev-dependencies `assert_cmd`, `predicates`, `tempfile`. Add each with `cargo add`, which picks the current version.

**Suggested layout** (you may change it): a library `src/lib.rs` for the logic and a thin `src/main.rs` for the terminal, like in Lesson 05.13.

```text
src/lib.rs        pub mod error; pub mod kdf; pub mod format; pub mod vault; pub mod fileio;
src/main.rs       clap definition, password prompts, calls into the library
tests/format.rs   tests of key derivation and the file format
tests/cli.rs      end-to-end tests with assert_cmd
```

**Suggested signatures** (the tests below refer to them):

```rust
pub struct KdfParams { pub m_cost: u32, pub t_cost: u32, pub p_cost: u32 }
pub fn derive_key(password: &[u8], salt: &[u8; 16], params: KdfParams) -> Result<Zeroizing<[u8; 32]>, VaultError>;
pub fn seal(vault: &Vault, password: &[u8], params: KdfParams) -> Result<Vec<u8>, VaultError>;
pub fn open(bytes: &[u8], password: &[u8]) -> Result<Vault, VaultError>;
```

**Threat model** (write it in the README as well)

| The vault protects against | The vault does **not** protect against |
|---|---|
| Someone who copies the vault file and tries to read it | Malware or a keylogger on your computer while you type the password |
| Someone who modifies the file (any change is detected) | A weak master password (Argon2 slows guessing, but cannot stop it) |
| Guessing the password offline (slowed by Argon2id) | Someone who reads the memory of the running program or a swap file |
| Secrets leaking through the command line and logs | A deleted file with no backup (there is no recovery) |

### Step-by-step plan

Work through the milestones in order. After each milestone run its "Done when" commands and make a Git commit. Try before you open a hint.

> 📏 **Convention:** commit messages have a type, a scope and a short sentence: `feat(vault): derive the key with Argon2id`. Branch: `git switch -c project/10-vault`.

---

#### M1 — The skeleton: command line, errors, no logic yet

**Goal:** a package that builds, parses the command line of the contract, and has the error type.

**Guidance:**

1. Create the package: `cargo new --lib code/p10-vault` (a library) and add `src/main.rs` yourself. Add the dependencies with `cargo add`.
2. In `src/error.rs` define `VaultError` with `thiserror`. One variant per row of the message table, plus `Io(#[from] std::io::Error)`. The `#[error("…")]` texts must be exactly the texts of the contract.
3. In `src/main.rs` define the command line with `clap` derive (`--file` global, with `env = "VAULT_FILE"`; the five subcommands). For now every command only prints that it is not implemented, and `main` turns an error into `error: …` and `ExitCode::FAILURE`.
4. Commit.

<details>
<summary>Hint 1 — a global option</summary>

`#[arg(long, global = true, env = "VAULT_FILE", default_value = "vault.bin")] file: PathBuf` on the struct that has `#[command(subcommand)]`. The feature `env` of `clap` is required for `env =`.

</details>

<details>
<summary>Hint 2 — the errors</summary>

`#[error("no secret named {0:?}")] NoSuchSecret(String)`: the `{0:?}` formatting puts the name in double quotes, as in the contract.

</details>

**Done when:** `cargo run -q -- --help` lists the five commands (descriptions are yours):

```text
Usage: vault [OPTIONS] <COMMAND>

Commands:
  init    Create a new, empty vault
  add     Store a secret (read from standard input)
  get     Print a secret
  list    List the names of all secrets
  remove  Delete a secret
  help    Print this message or the help of the given subcommand(s)
```

and `cargo run -q --; echo "exit: $?"` prints the usage error of `clap` and ends with:

```text
exit: 2
```

---

#### M2 — Key derivation

**Goal:** `derive_key` turns a password and a salt into a 32-byte key with Argon2id, and the key is wiped when dropped.

**Guidance:**

1. Read `argon2::Params::new`, `Argon2::new` and `hash_password_into` in the documentation of the version you use. Choose `Algorithm::Argon2id` and `Version::V0x13`.
2. Return the key as `Zeroizing<[u8; 32]>`. Map Argon2 errors to `VaultError::BadParams` (add that variant, with a message of your choice).
3. Write tests (cheap parameters `8, 1, 1`). Name them so that the word `kdf` is in each name: the same password and salt give the same key; another salt gives another key; another password gives another key; another `m_cost` gives another key.
4. Commit.

<details>
<summary>Hint 1 — the API shape</summary>

`let params = Params::new(m, t, p, Some(32))?; let argon = Argon2::new(Algorithm::Argon2id, Version::V0x13, params); argon.hash_password_into(password, salt, &mut key[..])?;` where `key` is a `Zeroizing<[u8; 32]>` (it dereferences to an array).

</details>

<details>
<summary>Hint 2 — comparing keys in tests</summary>

`assert_eq!(*key_a, *key_b)` compares the arrays inside the wrappers. In tests it is fine to compare with `==`; in the program you never compare keys, the AEAD tag does the checking.

</details>

**Done when:**

```bash
cargo test kdf
```

prints at least four tests whose names contain `kdf`, all `ok`, and ends with `test result: ok.`.

---

#### M3 — The sealed file format and tampering tests

**Goal:** `seal` and `open` implement the file format of the contract; every kind of damage is an error.

**Guidance:**

1. `seal`: create a random 16-byte salt and a random 24-byte nonce with `getrandom`; derive the key; build the 33-byte header; encrypt the JSON plaintext with the header as associated data; return `header || nonce || ciphertext+tag`. Put the in-memory secrets in a `Vault` type for now (a `BTreeMap<String, String>` inside).
2. `open`: check the length limits **first**, then the magic, then the version, then the parameter limits. Only then derive the key and decrypt. Map a failed decryption to the single "wrong password, or modified" error. Never `unwrap` or index without a check.
3. Write the tests below; give each a name containing the word shown in brackets.
4. Commit.

Tests to write:

- round trip: `open(seal(v)) == v` (`round_trip`);
- wrong password gives exactly the "wrong password" error (`wrong_password`);
- **flip every single byte** of a sealed vault, one at a time, and check that `open` returns an error each time and never panics (`tamper`);
- truncated files of several lengths, and an empty input, are errors (`truncated`);
- two seals of the same vault with the same password are different byte strings (`fresh_nonce`);
- a header with `m_cost` of `u32::MAX` is rejected without trying to allocate (`absurd_params`).

<details>
<summary>Hint 1 — building the header</summary>

Collect the pieces in a `Vec<u8>`: `extend_from_slice(b"VLT1")`, `push(1)`, then `m_cost.to_be_bytes()`, `t_cost.to_be_bytes()`, `p_cost.to_be_bytes()`, then the salt. Its length must be 33. Read numbers back with `u32::from_be_bytes([b0, b1, b2, b3])`.

</details>

<details>
<summary>Hint 2 — associated data</summary>

Use `Payload { msg: &plaintext, aad: &header }` for both encrypt and decrypt (Lesson 10.10, Exercise 2). `split_at` gives you the header, the nonce and the ciphertext without indexing.

</details>

<details>
<summary>Hint 3 — the "flip every byte" test</summary>

`for i in 0..bytes.len() { let mut bad = bytes.clone(); bad[i] ^= 1; assert!(open(&bad, pw).is_err(), "byte {i}"); }`. A flip in the magic, the version or the length-related parts is caught by the early checks; a flip anywhere else is caught by the tag. All of them must be errors.

</details>

**Done when:**

```bash
cargo test tamper && cargo test wrong_password && cargo test absurd_params
```

all pass, and `cargo test` as a whole ends with `test result: ok.`.

---

#### M4 — The commands and the file on disk

**Goal:** the five commands work with a real file.

**Guidance:**

1. Write `write_file(path, bytes)`: create `<path>.tmp` with mode `0o600` (`std::os::unix::fs::OpenOptionsExt::mode`), write, `sync_all`, then `rename` over the final path. Write `read_file(path)` that checks the file size against the 1 MiB limit **before** reading.
2. Add `Vault::set`, `get`, `remove` and `names` (sorted: a `BTreeMap` does it for you) with the name and size validation of the contract.
3. In `main.rs`, implement the commands. `load` = read file, ask the password, `open`. `save` = `seal` with `KdfParams::default()` (the contract values), then `write_file`.
4. Read the password with `rpassword::prompt_password` unless `VAULT_PASSWORD` is set. Read the secret for `add` from standard input as in the contract (prompt without echo when it is a terminal).
5. Commit.

<details>
<summary>Hint 1 — a terminal or a pipe?</summary>

`use std::io::IsTerminal; std::io::stdin().is_terminal()`. For the pipe, read with `Read::take(MAX_VALUE as u64 + 1)` so a huge input cannot fill the memory, and reject it if it is longer than the limit.

</details>

<details>
<summary>Hint 2 — the same code for all commands</summary>

`get`, `list` read only; `add` and `remove` read, change and save. A pair of small functions `load` and `save` keeps each command to a few lines.

</details>

**Done when:** (build with `cargo build`; in the listing the program is called `vault`, you run `./target/debug/vault`. The first three lines prepare a throwaway demo; `VAULT_PASSWORD` is exported **only for this experiment**: the `export` command itself is stored in your shell history, so never do that with a real password.)

```bash
mkdir -p /tmp/vault-demo
export VAULT_FILE=/tmp/vault-demo/vault.bin VAULT_PASSWORD='correct horse battery staple'
vault init
```

```text
Created /tmp/vault-demo/vault.bin
```

```bash
wc -c < $VAULT_FILE
ls -l $VAULT_FILE | cut -c1-10
```

```text
      75
-rw-------
```

```bash
echo hunter2 | vault add github-token
echo s3cr3t | vault add db.password
vault list
vault get github-token
```

```text
Added github-token
Added db.password
db.password
github-token
hunter2
```

```bash
vault get missing; echo "exit: $?"
VAULT_PASSWORD=wrong vault get github-token; echo "exit: $?"
vault init; echo "exit: $?"
```

```text
error: no secret named "missing"
exit: 1
error: wrong password, or the vault file was modified
exit: 1
error: /tmp/vault-demo/vault.bin already exists
exit: 1
```

And the file itself must not reveal the secrets. (`grep -a -c` counts the lines of a binary file that contain the text; `xxd` prints the first bytes of a file as hexadecimal.)

```bash
grep -a -c hunter2 $VAULT_FILE
xxd $VAULT_FILE | head -1
```

```text
0
00000000: 564c 5431 0100 004c 0000 0000 0200 0000  VLT1...L........
```

(`VLT1`, version `01`, `m_cost` = `0000 4c00` = 19456, `t_cost` = `0000 0002`, `p_cost` starts with `0000 00..`.) Finally, damage a copy of the file by overwriting byte 40 (inside the nonce) with a zero byte (`dd` copies bytes into a file at a given position) and read it:

```bash
cp $VAULT_FILE /tmp/vault-demo/copy.bin
printf '\x00' | dd of=/tmp/vault-demo/copy.bin bs=1 seek=40 conv=notrunc 2>/dev/null
vault --file /tmp/vault-demo/copy.bin get github-token; echo "exit: $?"
```

```text
error: wrong password, or the vault file was modified
exit: 1
```

(If your byte 40 already happened to be `00`, change a different byte, for example with `\xff`.) Clean up with `rm -rf /tmp/vault-demo` when you finish.

---

#### M5 — Hardening

**Goal:** the tool survives hostile files and hostile input, and keeps secrets out of unexpected places.

**Guidance:**

1. **Garbage input.** Test with a missing file, an empty file, a text file, 200 random bytes, a valid file with a changed version byte (`unsupported vault version`). Each must give a clean `error:` line and exit code `1`, and no panic message.
2. **Zeroizing audit.** Search your code: `grep -n "Zeroizing\|zeroize" src/*.rs`. The password, the key, the decrypted plaintext bytes and the secret read from standard input must each be in a `Zeroizing` value. Values inside the `Vault` map must be wiped in `Drop` for `Vault` and when a secret is removed (`String::zeroize`).
3. **No leaks.** Check that no `println!`, `format!` or error message includes a secret or the password; that `Vault` does not derive `Debug` (or has a manual `Debug` that prints only the names).
4. **Limits.** A name longer than 64 characters, a secret over 64 KiB (use `head -c 70000 /dev/zero | tr '\0' a | vault add big`) and a file over 1 MiB are rejected before any heavy work.
5. **Dependencies.** Run `cargo audit` and create a `deny.toml` for `cargo deny check` (Lesson 10.13) with the licenses your tree really uses.
6. Commit.

<details>
<summary>Hint 1 — a test with random garbage</summary>

Fill a buffer with `getrandom::fill` for lengths `0..300` and call `open`; every call must return `Err`. This is a small home-made fuzz test (real fuzzing: Lesson 09.8). The chance that random bytes start with `VLT1` and have a valid tag is nil.

</details>

<details>
<summary>Hint 2 — wiping on remove</summary>

`BTreeMap::remove` returns `Some(String)`: call `zeroize()` on that string before it drops. In `Drop for Vault`, loop over `self.entries.values_mut()`. Remember the `Zeroize` trait must be imported to call `.zeroize()`.

</details>

**Done when:**

```bash
echo 'x' > not-a-vault.txt; head -c 200 /dev/urandom > random.bin
vault --file not-a-vault.txt list; echo "exit: $?"
vault --file random.bin list; echo "exit: $?"
vault --file does-not-exist.bin list; echo "exit: $?"
```

(with `VAULT_PASSWORD` still exported from M4; otherwise the tool asks for the password first) print an `error:` line each (the first two are `the vault file is too short or too large` and `not a vault file (bad magic bytes)`, the third mentions the missing file; its wording is yours), all with `exit: 1`, and none prints the word `panicked`. `cargo audit` prints no vulnerability and `cargo deny check` ends with `advisories ok, bans ok, licenses ok, sources ok`.

---

#### M6 — End-to-end tests, README, final checks

**Goal:** the project is tested from the outside, documented and clean.

**Guidance:**

1. In `tests/cli.rs` use `assert_cmd` and `tempfile`: for each test create a temporary folder, set `VAULT_FILE` and `VAULT_PASSWORD` for the child process only (`.env(...)`; never in the test process), feed secrets with `.write_stdin(...)`. Test the full session of M4, the wrong password, the duplicate `init` and the invalid name.
2. Write `README.md`: what it is, the usage, the file format table, the **threat model** (what it does and does not protect against), and the warning that it is a learning project.
3. Run the final checks (below) and fix everything they report.
4. Commit and, if you want, open a pull request into your learning repository (Lesson 01.9).

<details>
<summary>Hint 1 — a helper for the tests</summary>

A function `fn vault(dir: &TempDir) -> Command` that creates `Command::cargo_bin("vault")`, sets `.current_dir(dir.path())` and the two environment variables saves repetition. (Lesson 09.2.)

</details>

<details>
<summary>Hint 2 — Clippy and the path type</summary>

If Clippy suggests `&Path` instead of `&PathBuf` for a parameter, take its advice: it accepts more callers.

</details>

**Done when:**

```bash
cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test
```

prints no formatting difference, no warning, and every test result is `ok` (the end-to-end tests take a few seconds because Argon2 runs with the real parameters). `git log --oneline` shows one commit per milestone.

### Stretch goals

- **`vault passwd`**: change the master password (open with the old one, save with the new one; a new salt is generated automatically).
- **Key file**: allow `--key-file <path>` in addition to the password (mix both into the Argon2 input), as a second factor.
- **Versioning**: add format version `2` (for example with a different cipher) and keep reading version `1`; write a test with a stored version 1 file.
- **`vault export` / `import`**: an encrypted backup of the whole vault with a different password.
- **`secrecy`**: use `SecretString` for the password and secrets, so that they cannot be printed by accident (`Debug` shows `[REDACTED]`).
- **Fuzzing**: add a `cargo fuzz` target for `open` (Lesson 09.8) and run it for a few minutes.
- **Prevent swapping**: read about `mlock` and why zeroizing does not remove copies that the operating system already made.
- **Timing check**: measure how long `init` takes with `hyperfine` (Lesson 09.11) and discuss how to choose `m_cost` and `t_cost` for your computer.

## Common mistakes

**1. Reusing the nonce.** Using one fixed nonce for every save is the mistake of Lesson 10.10. A fresh random nonce **and** a fresh salt on every save avoids it.

**2. Authenticating only the ciphertext.** Without the header as associated data, an attacker could change the Argon2 parameters or the version without detection. Pass the header as `aad`.

**3. Trusting the parameters in the file.** `m_cost` from a hostile file can ask for terabytes of memory. Check limits before calling Argon2.

**4. Panics on bad files.** `bytes[33..57]` on a 10-byte file panics. Use `split_at` after a length check, or `get`.

**5. Putting the secret on the command line.** `vault add token hunter2` would store the secret in the shell history and `ps`. That is why the value comes from standard input.

**6. Writing the file in place.** A crash in the middle of `std::fs::write` destroys the only copy of your secrets. Write a temporary file and rename it.

**7. Printing the password in a panic message or a log.** Never `{:?}` a struct that contains secrets without checking its `Debug` implementation.

## Quiz

1. Why does the vault use a random salt, and where is it stored?

<details>
<summary>Answer</summary>

The salt makes the key unique per file, so an attacker cannot reuse precomputed work across vaults or users and equal passwords give different keys. It is not secret, so it is stored in the file header.

</details>

2. What is protected by the associated data, and what would go wrong without it?

<details>
<summary>Answer</summary>

The header (magic, version, KDF parameters, salt). Without it an attacker could change the parameters or the version of a file without invalidating the tag.

</details>

3. Why is "wrong password" and "file modified" a single error message?

<details>
<summary>Answer</summary>

The AEAD tag check fails in both cases and the two cannot be told apart. A more specific message would be a guess, and an oracle for attackers.

</details>

4. Why is the secret read from standard input and not from an argument?

<details>
<summary>Answer</summary>

Arguments are visible to other users through `ps` and are saved in the shell history. Standard input is neither.

</details>

5. Does `Zeroizing` guarantee that no copy of the secret remains in memory? What does it do?

<details>
<summary>Answer</summary>

No. It overwrites that one value when it is dropped, and the compiler is prevented from optimizing the write away. It cannot erase copies made elsewhere (moves, buffers of other libraries, swap, core dumps). It reduces the exposure; it does not remove it.

</details>

## Summary

- The vault combines the tools of this phase: Argon2id derives the key from the password and a random salt, XChaCha20-Poly1305 encrypts and authenticates the data, and the header is authenticated as associated data.
- A good file format has a magic number, a version, the parameters it was made with, the salt and the nonce, and is parsed with explicit length checks and limits.
- Test security properties directly: wrong password, every flipped byte, truncated and random input, and absurd parameters.
- Secrets stay out of arguments, logs, error messages and `Debug` output, and are zeroized where Rust allows; know the limits of this protection.
- Replace the file atomically with restrictive permissions, and check the dependencies with `cargo audit` and `cargo deny`.
- Write the threat model down: it says what the tool does and what it does not do.

## Sources and further reading

- [argon2 crate documentation](https://docs.rs/argon2/0.6.0/argon2/) — `Argon2`, `Params` and `hash_password_into`.
- [chacha20poly1305 crate documentation](https://docs.rs/chacha20poly1305/0.11.0/chacha20poly1305/) — `XChaCha20Poly1305`, `Payload` and nonce generation.
- [zeroize crate documentation](https://docs.rs/zeroize/1.9.1/zeroize/) — `Zeroize`, `Zeroizing` and their limits.
- [rpassword crate documentation](https://docs.rs/rpassword/) — reading a password without echo.
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html) — recommended Argon2id parameters, from OWASP (the Open Worldwide Application Security Project).
- [RFC 9106 — Argon2](https://datatracker.ietf.org/doc/html/rfc9106) — the specification of Argon2 and parameter advice (an RFC, Request for Comments, is an official internet standard document).
- [Rust Command Line Book](https://rust-cli.github.io/book/) — exit codes, testing CLIs and good terminal behaviour.

---

[🏠 Start](../START-HERE.md) · [← 10.13 Supply-chain security: cargo-audit and cargo-deny](13-supply-chain-security.md) · [📋 Phase 10](README.md) · [10.15 Checkpoint →](15-checkpoint.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
