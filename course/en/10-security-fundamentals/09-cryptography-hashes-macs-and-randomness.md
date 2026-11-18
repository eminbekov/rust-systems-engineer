<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.8 Secrets management](08-secrets-management.md) · [📋 Phase 10](README.md) · [10.10 Cryptography II: encryption and signatures →](10-cryptography-encryption-and-signatures.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/09-cryptography-hashes-macs-and-randomness.md) <!-- nav -->

# Lesson 10.9 — Cryptography I: hashes, MACs and randomness

> **Phase 10 — Security fundamentals** · Lesson 9 of 15 · ⏱ about 135 minutes

## What you will learn

- What a hash function is, which properties make it useful, and where SHA-256 fits (and where it does not).
- How to find out whether a message was changed, with a hash (accidental change) and with a MAC (deliberate change).
- Why comparing secrets with `==` is dangerous, and how constant-time comparison fixes it.
- What a "cryptographically secure" random number is, and where your program should get its secret bytes.
- The first rule of cryptography: **never invent your own**. Use audited crates and their high-level APIs.

## Before you start

- You finished [Lesson 10.8 — Secrets management](08-secrets-management.md). Passwords and keys appear in this lesson, but only as bytes to protect.
- You know error handling with `Result` and `?` ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md)) and dependencies in Cargo ([Lesson 05.8](../05-rust-intermediate/08-cargo-dependencies-and-versions.md)).
- Create the practice package. In this phase every lesson has its own package in `code/`, and example programs live in `src/bin/`, one file per program. You run one with `cargo run --bin <name>`.

```bash
cd ~/projects/rust
cargo new code/p10-hashing
cd code/p10-hashing
cargo add sha2@0.11 hmac@0.13 hex@0.4 subtle@2.6 getrandom@0.4 rand@0.10 base64@0.23
mkdir src/bin
```

(`cargo add` writes the dependencies into `Cargo.toml` for you. The outputs below omit Cargo's `Compiling …` lines and show only what the program prints.)

> 🔒 **Ethics:** you practise only on your own programs and your own files. Never test attacks on systems you do not own.

## New words

| Word | Meaning |
|---|---|
| **cryptography** | The science of keeping information secret, unchanged and attributable, using mathematics. |
| **hash function** | A function that turns any amount of data into a short, fixed-size value (the *hash*, *digest* or *fingerprint*). The same input always gives the same output. |
| **SHA-256** (Secure Hash Algorithm, 256 bits) | A standard hash function that produces 32 bytes (256 bits). Part of the SHA-2 family. |
| **collision** | Two different inputs with the same hash. A good hash function makes finding one practically impossible. |
| **avalanche effect** | A tiny change in the input changes about half of the output bits. |
| **MAC** (Message Authentication Code) | A short tag computed from a message **and a secret key**. Only someone who knows the key can create a valid tag. |
| **HMAC** (Hash-based Message Authentication Code) | The standard way to build a MAC from a hash function such as SHA-256. |
| **key** | A secret value (random bytes) that an algorithm uses. The algorithm is public; the key is the secret. |
| **timing attack** | Learning a secret by measuring how long a program takes to answer. |
| **constant-time** | Code whose running time does not depend on secret values. |
| **entropy** | A measure of unpredictability, counted in bits. A key with 128 bits of entropy has 2^128 equally likely values. |
| **CSPRNG** (Cryptographically Secure Pseudo-Random Number Generator) | A random generator whose output nobody can predict, even after seeing a lot of earlier output. |
| **seed** | The starting value of a pseudo-random generator. Same seed, same sequence. |
| **hex** (hexadecimal) | Writing bytes as two characters from `0-9a-f` each. 32 bytes become 64 characters. |
| **base64** | Writing bytes with 64 printable characters (`A-Z a-z 0-9 + /`). A variant, *base64url*, uses `-` and `_` so the text is safe in URLs. |

## 1. The first rule: do not invent cryptography

Cryptography is easy to write and extremely hard to write **correctly**. A home-made algorithm can look
random, pass all your tests and still be broken in an afternoon by someone who knows the field. Even a
correct algorithm fails if it is used wrongly, for example with a reused value or a leaked key.

So the rules of this phase are:

1. Never design your own algorithm or protocol.
2. Use an **audited**, widely used crate (a crate whose code was reviewed by experts).
3. Use its **high-level API** (Application Programming Interface: the functions a library offers, such as "encrypt this message") and not the low-level pieces.
4. Read the crate's documentation for the exact version you use. Several of these crates changed their API in 2025 and 2026, so old tutorials may not compile.

This lesson gives you three building blocks: hashes, MACs and randomness. Lesson 10.10 adds encryption
and signatures. Lesson 10.11 shows how all of them work together in TLS (Transport Layer Security, the protocol that protects network connections).

## 2. Hash functions and SHA-256

A hash function is like a **fingerprint machine for data**. You put in anything (one letter, a photo, a
4 GB file; GB means gigabyte, about a billion bytes). Out comes a short value of fixed size. In the picture below `H` is the hash function:

```text
  "hello"                      ┐
  a 4 GB video file            ├──►  H  ──►  2cf24dba5fb0a30e...  (always 32 bytes)
  (nothing at all)             ┘
```

A hash function used in security has these properties:

| Property | Meaning |
|---|---|
| **Deterministic** | The same input always gives the same hash. |
| **Fixed size** | SHA-256 always gives 32 bytes, for any input length. |
| **One-way** | Given a hash, you cannot find an input that makes it (except by guessing inputs). |
| **Collision resistant** | You cannot find two different inputs with the same hash. |
| **Avalanche** | Changing one bit of the input changes about half of the bits of the output. |

Create `src/bin/sha_basics.rs`:

```rust
use sha2::{Digest, Sha256};

fn main() {
    // One call: hash a byte slice.
    let hash = Sha256::digest(b"hello");
    println!("hello          -> {}", hex::encode(hash));

    // Same input, same output; a tiny change gives a completely different output.
    println!("hello (again)  -> {}", hex::encode(Sha256::digest(b"hello")));
    println!("Hello          -> {}", hex::encode(Sha256::digest(b"Hello")));

    // Incremental: feed the data in pieces.
    let mut hasher = Sha256::new();
    hasher.update(b"hel");
    hasher.update(b"lo");
    let pieces = hasher.finalize();
    println!("hel + lo       -> {}", hex::encode(pieces));

    // The output is always 32 bytes (256 bits), however long the input is.
    let long = vec![b'a'; 1_000_000];
    println!("1 million 'a'  -> {} bytes of output", Sha256::digest(&long).len());
}
```

**What happens:**

1. `use sha2::{Digest, Sha256}` imports the type `Sha256` and the trait `Digest`. The trait provides the methods `digest`, `new`, `update` and `finalize`. Without it in scope they do not exist (Rust only lets you call trait methods when the trait is imported).
2. `Sha256::digest(b"hello")` hashes a whole byte slice at once. `b"hello"` is a *byte string*: the bytes of the text. The result is 32 bytes.
3. `hex::encode(hash)` turns the 32 bytes into 64 readable hexadecimal characters. The raw bytes are not printable, so we always show a hash in hex.
4. For large data you create a hasher with `Sha256::new()`, call `update` as many times as you like, and `finalize` once at the end. Feeding `hel` and then `lo` gives exactly the same hash as `hello`.
5. A million bytes of input still produce 32 bytes of output.

Run it with `cargo run --bin sha_basics`:

```text
hello          -> 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
hello (again)  -> 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
Hello          -> 185f8db32271fe25f561a6fc938b2e264306ec304eda518007d1764826381969
hel + lo       -> 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824
1 million 'a'  -> 32 bytes of output
```

You can check the first value with a tool that is already on your Mac. It is a different program that
implements the same standard, so it must agree:

```bash
echo -n hello | shasum -a 256
```

```text
2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824  -
```

(`echo -n` prints without a trailing newline. Without `-n` you would hash the six bytes `hello\n` and get a different value.)

The lines `hello` and `Hello` above show the **avalanche effect**: one changed letter gives a completely different hash
(about half of the 256 output bits flip). That is why a hash is a good fingerprint: nobody can make a "slightly modified" file with a "slightly different" hash.

### What a hash is for

- **Integrity against accidents and for downloads.** Publish the hash of a file next to the file; the user hashes their copy and compares. `Cargo.lock` and package registries use the same idea (Lesson 10.13).
- **Fingerprints and identifiers.** Git names every commit and file by a hash of its content (Phase 01).
- **A building block** inside MACs, signatures and password hashing.

### What a hash is NOT for

> ⚠️ **Common mistake:** "I stored `SHA-256(password)` in the database, so the passwords are safe." They are not. A hash is
> **fast** by design, so an attacker with a stolen database can try billions of guesses per second.
> Passwords need a deliberately slow, salted algorithm. That is Argon2, from [Lesson 10.6](06-authentication-and-password-hashing.md).

> ⚠️ **Common mistake:** "A hash proves who sent the file." It does not. An attacker who changes the file simply
> computes the new hash too. If both the file and its hash travel over the same untrusted way, the hash protects against accidents only.
> To protect against a deliberate attacker you need a **key**: a MAC (Section 4) or a signature (Lesson 10.10).

> 📏 **Convention:** use SHA-256 (or SHA-512, SHA-3) for new code. **MD5** (Message Digest 5) and **SHA-1** are broken for security purposes:
> researchers have produced real collisions (two different files with the same hash). They are still fine as a quick checksum
> against accidental damage, but never where an attacker is involved.

> ⚡ **Performance:** SHA-256 runs at hundreds of megabytes per second or more on a modern CPU (Central Processing Unit).
> That speed is a feature for checksums and a disaster for passwords. To hash a big file, call `update` on each chunk you read, so the whole file never sits in memory.

## 3. Encoding is not encryption

Hashes and keys are raw bytes. To print them or put them in JSON (JavaScript Object Notation, a text format for data), URLs (web addresses) or configuration files, we **encode**
them as text. The two common encodings are hex and base64. Create `src/bin/encode.rs`:

```rust
use base64::Engine;
use base64::engine::general_purpose::{STANDARD, URL_SAFE_NO_PAD};

fn main() {
    let bytes = [0xfb, 0xff, 0x00, 0x42];
    println!("hex            : {}", hex::encode(bytes));
    println!("base64         : {}", STANDARD.encode(bytes));
    println!("base64url      : {}", URL_SAFE_NO_PAD.encode(bytes));

    // Decoding needs no key at all.
    let decoded = URL_SAFE_NO_PAD.decode("cGFzc3dvcmQxMjM").unwrap();
    println!("decoded        : {}", String::from_utf8(decoded).unwrap());
}
```

**What happens:**

1. In `base64` 0.23 you choose an *engine* (a configuration). `STANDARD` uses `+` and `/` and ends with `=` padding; `URL_SAFE_NO_PAD` uses `-` and `_` and no padding. The `Engine` trait must be imported to get `encode` and `decode`.
2. The same four bytes look different in each encoding.
3. We decode a string that looks like a secret. It took no key and no effort: it was just `password123`.

```text
hex            : fbff0042
base64         : +/8AQg==
base64url      : -_8AQg
decoded        : password123
```

> 🔒 **Security:** encoding hides nothing. Anyone can decode base64 in a second. A "base64-encoded password" is a plain password.
> If you see base64 in a configuration file, a cookie or a token, assume that a reader can read it.

## 4. MAC and HMAC: a hash with a secret key

Problem: Alice sends Bob the message `user=alice;role=reader` together with its SHA-256 hash. Mallory
sits between them. She changes the message to `role=admin` and computes a new hash. Bob cannot tell.

A **MAC** solves this by mixing a **secret key** into the computation. Alice and Bob share the key;
Mallory does not, so she cannot produce the right tag for her changed message.

```text
  message ──┐
            ├──►  HMAC-SHA256  ──►  tag (32 bytes)        sent together: message + tag
  key ──────┘

  Bob recomputes the tag with the same key and compares it with the received tag.
  Different message or different key  =>  different tag  =>  reject.
```

You might think "just hash `key + message`". Do not: SHA-256 has a known weakness (*length extension*) that lets
an attacker who sees `hash(key + message)` compute `hash(key + message + extra)` without knowing the key.
**HMAC** is the standard construction (it hashes twice with the key mixed in two different ways) that avoids this.
You never write it yourself; you call the crate. Create `src/bin/hmac_demo.rs`:

```rust
use hmac::{Hmac, KeyInit, Mac};
use sha2::Sha256;

type HmacSha256 = Hmac<Sha256>;

fn main() {
    let key = b"a secret key that only the server knows";
    let message = b"user=alice;role=reader";

    // Compute the tag.
    let mut mac = HmacSha256::new_from_slice(key).expect("HMAC accepts keys of any length");
    mac.update(message);
    let tag = mac.finalize().into_bytes();
    println!("tag      = {}", hex::encode(tag));

    // Verify the genuine message.
    let mut check = HmacSha256::new_from_slice(key).unwrap();
    check.update(message);
    println!("genuine  = {:?}", check.verify_slice(&tag));

    // Verify a message that an attacker changed.
    let mut check = HmacSha256::new_from_slice(key).unwrap();
    check.update(b"user=alice;role=admin ");
    println!("tampered = {:?}", check.verify_slice(&tag));

    // Verify with the wrong key.
    let mut check = HmacSha256::new_from_slice(b"some other key").unwrap();
    check.update(message);
    println!("wrong key= {:?}", check.verify_slice(&tag));
}
```

**What happens:**

1. `type HmacSha256 = Hmac<Sha256>;` gives a short name to "HMAC built on SHA-256".
2. `new_from_slice(key)` creates the MAC state from the key. It comes from the trait `KeyInit`, and `update` and `verify_slice` come from the trait `Mac`. Both traits must be imported.
3. `mac.finalize().into_bytes()` returns the 32-byte tag.
4. `verify_slice(&tag)` recomputes and compares. It returns `Ok(())` when the tag is valid and `Err(MacError)` when it is not. The comparison inside is **constant-time** (explained next).
5. A changed message and a wrong key both give an error.

```text
tag      = 1b237df19bae7910ef38f6977156889d50fa1374f37f2ccd455f4840d2e3644b
genuine  = Ok(())
tampered = Err(MacError)
wrong key= Err(MacError)
```

You can check the tag with the `openssl` tool that comes with macOS (a second, independent implementation):

```bash
echo -n 'user=alice;role=reader' | openssl dgst -sha256 -hmac 'a secret key that only the server knows'
```

```text
1b237df19bae7910ef38f6977156889d50fa1374f37f2ccd455f4840d2e3644b
```

(Other builds of OpenSSL print a label such as `SHA2-256(stdin)= ` before the same hash value.)

### Why `verify_slice` and not `==`

If you compare two tags with `==`, the comparison usually stops at the **first different byte**. A guess
whose first byte is right takes a tiny bit longer to reject than a guess whose first byte is wrong. An attacker who can
send many guesses and measure time very precisely can learn the tag byte by byte. This is a **timing attack** (you met it briefly in [Lesson 10.6](06-authentication-and-password-hashing.md)).

Create `src/bin/compare.rs` to see the idea without measuring any time. It counts how many bytes the naive comparison looked at:

```rust
use subtle::ConstantTimeEq;

/// A naive comparison that stops at the first difference. Returns (equal, bytes_looked_at).
fn naive_eq(a: &[u8], b: &[u8]) -> (bool, usize) {
    if a.len() != b.len() {
        return (false, 0);
    }
    for i in 0..a.len() {
        if a[i] != b[i] {
            return (false, i + 1);
        }
    }
    (true, a.len())
}

fn main() {
    let secret = b"s3cr3t-t0ken";
    for guess in [&b"x3cr3t-t0ken"[..], b"s3cr3t-t0kex", b"s3cr3t-t0ken"] {
        let (equal, looked) = naive_eq(secret, guess);
        println!("naive    {:<14} equal={equal:<5} looked at {looked} bytes", String::from_utf8_lossy(guess));
    }

    // Constant-time: always looks at every byte. `ct_eq` returns a `Choice`, not a bool.
    let choice = secret.ct_eq(b"s3cr3t-t0ken");
    println!("constant-time equal = {}", bool::from(choice));
}
```

```text
naive    x3cr3t-t0ken   equal=false looked at 1 bytes
naive    s3cr3t-t0kex   equal=false looked at 12 bytes
naive    s3cr3t-t0ken   equal=true  looked at 12 bytes
constant-time equal = true
```

**What happens:**

1. The naive function leaks information: "1 byte" means the first byte was wrong; "12 bytes" means the first eleven were right. Each answer narrows the search.
2. `ct_eq` from the crate `subtle` always processes every byte and returns a `Choice`, a value designed so the compiler does not turn it back into an early exit. Convert it with `bool::from` only at the end.

> 🔒 **Security:** use `verify_slice` for MAC tags, and `subtle::ConstantTimeEq` for any other secret you compare
> (API tokens, session identifiers, reset codes). Do not write `if provided == expected` for secrets.

### From tag to token

A very common use of a MAC is a *signed token*: the text `payload.tag`, with both parts in base64url. A server can hand it to a browser
and later trust it without a database, as long as the tag verifies. Two rules apply. First, check the tag **before** you read or parse the payload.
Second, a MAC does not hide the payload (anyone can base64-decode it) and does not stop **replay** (sending an old valid token again), so a
real token also carries an expiry time inside the signed data. You build exactly this in Exercise 3. Hiding data is the job of encryption (Lesson 10.10).

## 5. Randomness

Keys, salts, nonces, session tokens and reset codes all need numbers that nobody can guess. Two kinds of
random generators exist:

| Kind | Example | Use for |
|---|---|---|
| **Pseudo-random, predictable** | a generator you create from a `seed` | simulations, games, tests that must be repeatable |
| **Cryptographically secure (CSPRNG)** | bytes from the operating system | keys, tokens, salts, nonces, anything an attacker must not guess |

A pseudo-random generator is just a formula. Start it from the same seed and you get the same numbers.
(Lesson 10.7 used `getrandom::fill` for session tokens; this is why that was the right choice.) If an attacker guesses the seed (the current time in seconds is a classic bad seed), they can compute all your
"random" values. Create `src/bin/random_demo.rs`:

```rust
use rand::rngs::StdRng;
use rand::{Rng, RngExt, SeedableRng};

fn main() {
    // 1. A *seeded* generator is a pure function of its seed: fully predictable.
    let mut a = StdRng::seed_from_u64(42);
    let mut b = StdRng::seed_from_u64(42);
    let (x, y): (u32, u32) = (a.random(), b.random());
    println!("seed 42 twice : {x} and {y}");

    // 2. Secret bytes straight from the operating system.
    let mut key = [0u8; 32];
    getrandom::fill(&mut key).expect("the operating system could not give random bytes");
    println!("OS random     : {}", hex::encode(key));

    // 3. rand's thread-local generator is seeded from the OS and is cryptographically secure.
    let mut token = [0u8; 16];
    rand::rng().fill_bytes(&mut token);
    println!("rand token    : {}", hex::encode(token));
}
```

**What happens:**

1. Two generators created from the seed `42` produce the same number. Anything derived from a seed you chose is not a secret.
2. `getrandom::fill` asks the operating system's random source (on macOS and Linux the kernel collects unpredictable events and feeds a CSPRNG). It fills the byte array and returns a `Result`, because in rare cases (very early boot, a locked-down sandbox) the system cannot answer. For a secret you must handle that error, not ignore it.
3. `rand::rng()` returns a per-thread generator that the `rand` crate documents as cryptographically secure and reseeds from the operating system. `fill_bytes` fills the array. In `rand` 0.10 the trait `Rng` holds the core methods (`fill_bytes`) and `RngExt` holds the convenient ones (`random`, `random_range`).

```text
seed 42 twice : 572990626 and 572990626
OS random     : 5e26b893468ccd97371c7640045c63ce4d6526c1988c907bdccf2f619a7c5f8b
rand token    : 6b97f6729c97166fb974f25fb553e2f0
```

The last two lines are different on every run; only the first line repeats. How many bytes are enough?
Think in **bits of entropy**: a random 128-bit value has 2^128 possibilities, about 3.4 × 10^38.
Even at a trillion guesses per second, trying them all would take far longer than the age of the universe.

| Use | Random bytes |
|---|---|
| Session token, reset token, API key | 32 (256 bits) |
| Encryption key (Lesson 10.10) | 32 |
| Password-hash salt (Lesson 10.6) | 16 |
| Nonce for XChaCha20-Poly1305 (Lesson 10.10) | 24 |

> 🔒 **Security:** if a number protects something, it comes from the operating system or from a CSPRNG, never from the time,
> a counter, a seeded generator, or a hash of something guessable. Never reduce a secret to "a random number from 0 to 999999"
> when you can use 32 random bytes.

> 🧪 **Testing:** you cannot test that bytes are "random enough" with one assertion, but you can test the shape: the length, the allowed characters, and that two tokens differ (Exercise 2).

## Common mistakes

**1. Forgetting to import the trait.** Remove `KeyInit` from the `use hmac::…` line of `hmac_demo.rs`. The compiler cannot find `new_from_slice` even though `Hmac` is imported:

```text
error[E0599]: no associated function or constant named `new_from_slice` found for struct `Hmac<D>` in the current scope
  --> src/bin/hmac_demo.rs:11:31
   |
11 |     let mut mac = HmacSha256::new_from_slice(key).expect("HMAC accepts keys of any length");
   |                               ^^^^^^^^^^^^^^ associated function or constant not found in `Hmac<Sha256>`
   |
   = help: items from traits can only be used if the trait is in scope
help: trait `KeyInit` which provides `new_from_slice` is implemented but not in scope; perhaps you want to import it
   |
 1 + use hmac::KeyInit;
   |
```

The compiler's help is exact: add `KeyInit` (and `Mac` for `update`) to the `use` line. This is the same rule you met with `Digest` above.

**2. Printing a hash with `{:x}`.** Older tutorials do `println!("{:x}", hash)`. With `sha2` 0.11 the result type no longer implements `LowerHex`
(`error[E0277]: … LowerHex is not satisfied`). Use `hex::encode(hash)`.

**3. Comparing tags with `==`.** It compiles and works, and it leaks timing information. Use `verify_slice` or `ct_eq`.

**4. Using a seeded or time-based generator for secrets.** It compiles, works, and is guessable.

**5. Hardcoding the key in source code.** The key in `hmac_demo.rs` is for teaching. Real keys come from the secret store you learned about in Lesson 10.8 and are random bytes, not words.

## Try it yourself

1. Run `cargo run --bin sha_basics`. Change `b"hello"` to `b"hellp"` and watch the whole output change.
2. In `hmac_demo.rs`, change one character of the key in the "genuine" check only (not in the first computation) and see `genuine  = Err(MacError)`.

## Exercises

### Exercise 1 — Verify a download (easy)

Create `note.txt` with any text. Save its hash with `shasum -a 256 note.txt > note.sha256`, then check it with
`shasum -a 256 -c note.sha256`. Edit `note.txt` and check again. What is printed each time?

<details>
<summary>Hint</summary>

`-c` means "check": it recomputes the hash of every file listed in the `.sha256` file and compares.

</details>

<details>
<summary>Answer</summary>

```bash
printf 'my note\n' > note.txt
shasum -a 256 note.txt > note.sha256
shasum -a 256 -c note.sha256
printf 'changed\n' >> note.txt
shasum -a 256 -c note.sha256
```

```text
note.txt: OK
note.txt: FAILED
shasum: WARNING: 1 computed checksum did NOT match
```

Why it works: one changed byte gives a completely different hash (avalanche), so the comparison fails.

</details>

### Exercise 2 — A reset-token generator with a test (medium)

In a new library package (`cargo new --lib code/p10-token`) write `pub fn new_token() -> String` that returns 32 random bytes as 64 hex
characters, taken from the operating system. Write a test that checks the length, that every character is a hex digit, and that two tokens differ.

<details>
<summary>Hint</summary>

`cargo add getrandom@0.4 hex@0.4`. Fill a `[0u8; 32]` array with `getrandom::fill`. A test for random data checks the *shape*, not the values.

</details>

<details>
<summary>Answer</summary>

```rust
pub fn new_token() -> String {
    let mut bytes = [0u8; 32];
    getrandom::fill(&mut bytes).expect("no OS randomness");
    hex::encode(bytes)
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn tokens_have_right_shape_and_differ() {
        let (a, b) = (new_token(), new_token());
        assert_eq!(a.len(), 64);
        assert!(a.chars().all(|c| c.is_ascii_hexdigit()));
        assert_ne!(a, b);
    }
}
```

`cargo test` prints `test tests::tokens_have_right_shape_and_differ ... ok`.

Why it works: two random 256-bit values are equal with probability about 1 in 10^77, so `assert_ne!` will not fail by bad luck. The test would catch a bug such as returning a fixed string.

</details>

### Exercise 3 — A token that expires (hard)

Build a signed token as a library with `sign(key, payload, expires_at)`, which returns `body.tag` (both parts in base64url, `body` being the text `payload|expires_at`), and `verify(key, token, now)`. Times are Unix seconds
(whole seconds since 1970-01-01). The expiry must be **inside the signed data**, so an attacker cannot extend it. Test: a valid token, an
expired token, an edited expiry, and the wrong key. Pass `now` as a parameter so the tests do not depend on the clock.

<details>
<summary>Hint</summary>

Sign the text `"{payload}|{expires_at}"`. In `verify`, check the tag **first**, then split at the last `|` with `rsplit_once('|')`, parse the number and compare it with `now`.

</details>

<details>
<summary>Answer</summary>

```rust
use base64::Engine;
use base64::engine::general_purpose::URL_SAFE_NO_PAD as B64;
use hmac::{Hmac, KeyInit, Mac};
use sha2::Sha256;

type HmacSha256 = Hmac<Sha256>;

pub fn sign(key: &[u8], payload: &str, expires_at: u64) -> String {
    let body = format!("{payload}|{expires_at}");
    let mut mac = HmacSha256::new_from_slice(key).unwrap();
    mac.update(body.as_bytes());
    let tag = mac.finalize().into_bytes();
    format!("{}.{}", B64.encode(&body), B64.encode(tag))
}

pub fn verify(key: &[u8], token: &str, now: u64) -> Option<String> {
    let (body_b64, tag_b64) = token.split_once('.')?;
    let body = B64.decode(body_b64).ok()?;
    let tag = B64.decode(tag_b64).ok()?;
    let mut mac = HmacSha256::new_from_slice(key).ok()?;
    mac.update(&body);
    mac.verify_slice(&tag).ok()?; // check the tag BEFORE trusting anything inside
    let body = String::from_utf8(body).ok()?;
    let (payload, expires) = body.rsplit_once('|')?;
    let expires: u64 = expires.parse().ok()?;
    if now >= expires {
        return None;
    }
    Some(payload.to_string())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn valid_token_is_accepted_until_it_expires() {
        let key = b"k";
        let token = sign(key, "user=alice", 1000);
        assert_eq!(verify(key, &token, 999).as_deref(), Some("user=alice"));
        assert_eq!(verify(key, &token, 1000), None);
    }

    #[test]
    fn tampering_is_rejected() {
        let key = b"k";
        let token = sign(key, "user=alice", 1000);
        let (_, tag) = token.split_once('.').unwrap();
        // Same tag, new expiry far in the future.
        let forged = format!("{}.{}", B64.encode("user=alice|9999999"), tag);
        assert_eq!(verify(key, &forged, 10), None);
        assert_eq!(verify(b"other key", &token, 10), None);
    }
}
```

`cargo test` shows both tests `ok`.

Why it works: the expiry is part of the text that the tag covers, so changing it invalidates the tag. Passing `now` in makes the function pure (same inputs, same answer), which makes it easy to test.

</details>

## Quiz

1. SHA-256 receives an input of 5 bytes and another of 5 gigabytes. How long are the two outputs?

<details>
<summary>Answer</summary>

Both are 32 bytes (64 hex characters). A hash function always gives a fixed-size output.

</details>

2. Why is `SHA-256(password)` a poor way to store passwords?

<details>
<summary>Answer</summary>

SHA-256 is fast, so an attacker with the stolen hashes can test billions of guesses per second. Passwords need a slow, salted algorithm such as Argon2 (Lesson 10.6).

</details>

3. Mallory can change a message and recompute its plain SHA-256 hash. Why can she not do the same with an HMAC tag?

<details>
<summary>Answer</summary>

A valid HMAC tag needs the secret key, which Mallory does not have.

</details>

4. What is wrong with `if token == expected_token` for a secret token?

<details>
<summary>Answer</summary>

The comparison can stop at the first different byte, so the time it takes depends on how much of the guess is right. That can leak the secret in a timing attack. Use `subtle::ConstantTimeEq` (or `verify_slice` for MAC tags).

</details>

5. You need a session token. Which is right: `StdRng::seed_from_u64(now_in_seconds)` or 32 bytes from `getrandom::fill`?

<details>
<summary>Answer</summary>

32 bytes from `getrandom::fill`. A generator seeded from the clock can be reproduced by anyone who guesses the time. Also remember that base64 or hex of a token is not protection: they are encodings without a key.

</details>

## Summary

- Never invent cryptography. Use audited crates, their high-level APIs, and the documentation of the exact version.
- A hash (SHA-256) is a fixed-size fingerprint of data: deterministic, one-way and collision resistant. MD5 and SHA-1 are broken.
- A plain hash detects accidents, not attackers, and it is far too fast for passwords.
- A MAC (HMAC-SHA256) adds a secret key, so only key holders can create a valid tag. It authenticates a message but does not hide it.
- Compare secrets in constant time: `verify_slice` for tags, `subtle::ConstantTimeEq` for everything else.
- Hex and base64 are encodings, not protection.
- Secret bytes come from the operating system (`getrandom`) or a CSPRNG (`rand::rng()`), never from a seeded or time-based generator. Use 16 to 32 bytes.

## Sources and further reading

- [sha2 crate documentation](https://docs.rs/sha2/0.11.1/sha2/) — the hashing API used in this lesson.
- [hmac crate documentation](https://docs.rs/hmac/0.13.0/hmac/) — HMAC types and the `Mac` trait.
- [subtle crate documentation](https://docs.rs/subtle/2.6.1/subtle/) — constant-time comparison and the `Choice` type.
- [getrandom crate documentation](https://docs.rs/getrandom/0.4.3/getrandom/) — where the operating system's random bytes come from, per platform.
- [RFC 2104 — HMAC](https://datatracker.ietf.org/doc/html/rfc2104) — the specification of HMAC (an RFC, Request for Comments, is a document of the official internet standards series).
- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html) — practical advice on which algorithms to use and which to avoid, from OWASP (the Open Worldwide Application Security Project).

---

[🏠 Start](../START-HERE.md) · [← 10.8 Secrets management](08-secrets-management.md) · [📋 Phase 10](README.md) · [10.10 Cryptography II: encryption and signatures →](10-cryptography-encryption-and-signatures.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
