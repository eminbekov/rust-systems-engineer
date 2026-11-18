<!-- nav:top -->
[🏠 Start](../START-HERE.md) · [← 10.9 Cryptography I: hashes, MACs and randomness](09-cryptography-hashes-macs-and-randomness.md) · [📋 Phase 10](README.md) · [10.11 TLS and certificates →](11-tls-and-certificates.md) · [📖 Glossary](../GLOSSARY.md) · [🇷🇺 Русский](../../ru/10-security-fundamentals/10-cryptography-encryption-and-signatures.md) <!-- nav -->

# Lesson 10.10 — Cryptography II: encryption and signatures

> **Phase 10 — Security fundamentals** · Lesson 10 of 15 · ⏱ about 135 minutes

## What you will learn

- The difference between symmetric encryption (one shared key) and asymmetric cryptography (a key pair).
- What AEAD (authenticated encryption) is, and why you should never use encryption without authentication.
- Why a nonce must never repeat with the same key, and what happens if it does.
- How to encrypt and decrypt with XChaCha20-Poly1305, and how to store the nonce next to the ciphertext.
- How to sign data with Ed25519 and verify the signature with the public key.

## Before you start

- You finished [Lesson 10.9 — Cryptography I: hashes, MACs and randomness](09-cryptography-hashes-macs-and-randomness.md). Everything here builds on keys, MACs (message authentication codes) and secure random bytes.
- You know `Result`, `Option` and tests ([Lesson 04.10](../04-structs-enums-collections-errors/10-error-handling.md), [Lesson 05.6](../05-rust-intermediate/06-testing-in-depth.md)).
- Create the practice package (examples go in `src/bin/`, as in the previous lesson):

```bash
cd ~/projects/rust
cargo new code/p10-encryption
cd code/p10-encryption
cargo add chacha20poly1305@0.11 aes-gcm@0.11 ed25519-dalek@3 getrandom@0.4 hex@0.4
mkdir src/bin
```

## New words

| Word | Meaning |
|---|---|
| **plaintext / ciphertext** | The readable data before encryption / the unreadable result after encryption. |
| **encryption / decryption** | Turning plaintext into ciphertext with a key / turning it back with the right key. |
| **symmetric encryption** | Encryption where the **same** secret key encrypts and decrypts. |
| **asymmetric cryptography** | Cryptography with a **key pair**: a *public key* that anyone may know and a *private key* that stays secret. |
| **cipher** | An encryption algorithm, for example ChaCha20 or AES (Advanced Encryption Standard). |
| **AEAD** (Authenticated Encryption with Associated Data) | Encryption that also produces an authentication tag. It hides the data **and** detects any change. |
| **nonce** ("number used once") | A value that must be unique for every message encrypted with the same key. It is not secret. |
| **authentication tag** | A 16-byte check value that AEAD appends to the ciphertext. Decryption fails if the data or the tag changed. |
| **associated data** (AAD) | Data that is not encrypted but is covered by the tag, for example a header. |
| **digital signature** | A value made with a private key that anyone can check with the matching public key. It proves who signed and that the data did not change. |
| **Ed25519** | A modern, fast signature algorithm. Keys are 32 bytes, signatures are 64 bytes. |
| **KDF** (Key Derivation Function) | A function that turns a password into a key of the right length. Slow ones, such as Argon2, are used for passwords. |

## 1. Two families

**Symmetric** cryptography uses one secret key for both directions, like a house key that locks and unlocks the same door.
It is fast and used for the actual data. Its problem is the key: both sides must have it, and nobody else may.

**Asymmetric** cryptography uses a key pair. What the private key does, only the public key can check (signatures), and
what the public key does, only the private key can undo (public-key encryption and key exchange). You can publish the public key to the whole world.

| | Symmetric | Asymmetric |
|---|---|---|
| Keys | one shared secret | public + private |
| Speed | very fast | slower |
| Typical use | encrypt files, databases, network traffic | signatures, agreeing on a shared key (TLS, Transport Layer Security, next lesson) |
| Example in this lesson | XChaCha20-Poly1305 | Ed25519 |

Real systems combine both: asymmetric cryptography proves identity and agrees on a key, then symmetric encryption protects the data.

> 📏 **Convention:** use only **AEAD** ciphers: XChaCha20-Poly1305, ChaCha20-Poly1305 or AES-256-GCM (AES with a 256-bit key in GCM,
> Galois/Counter Mode). A raw cipher (AES alone, or an old *mode* — a way to apply a block cipher to long data — such as ECB, Electronic
> Codebook, or CBC, Cipher Block Chaining, without a MAC) hides data but does not detect changes, and it is easy to misuse. An attacker who can flip bits in your ciphertext can often flip the matching bits in the plaintext.

## 2. Encrypting with XChaCha20-Poly1305

An AEAD cipher takes four inputs and gives one output:

```text
  key (32 bytes, secret) ──┐
  nonce (24 bytes)        ─┤
  plaintext               ─┼──►  encrypt  ──►  ciphertext + tag (16 bytes)
  associated data (opt.)  ─┘

  decrypt(key, nonce, ciphertext + tag)  =  plaintext   or   Error  (never garbage)
```

Create `src/bin/encrypt.rs`:

```rust
use chacha20poly1305::aead::{Aead, Generate, Key, KeyInit};
use chacha20poly1305::{XChaCha20Poly1305, XNonce};

fn main() {
    // 1. A random 32-byte key and a random 24-byte nonce (from the operating system).
    let key = Key::<XChaCha20Poly1305>::generate();
    let nonce = XNonce::generate();
    let cipher = XChaCha20Poly1305::new(&key);

    // 2. Encrypt.
    let plaintext = b"meet me at noon";
    let ciphertext = cipher.encrypt(&nonce, plaintext.as_ref()).unwrap();
    println!("plaintext  : {} bytes", plaintext.len());
    println!("ciphertext : {} bytes = {}", ciphertext.len(), hex::encode(&ciphertext));

    // 3. Decrypt with the right key and nonce.
    let back = cipher.decrypt(&nonce, ciphertext.as_ref()).unwrap();
    println!("decrypted  : {}", String::from_utf8(back).unwrap());

    // 4. Flip one bit of the ciphertext: decryption fails, nothing is returned.
    let mut tampered = ciphertext.clone();
    tampered[0] ^= 1;
    println!("tampered   : {:?}", cipher.decrypt(&nonce, tampered.as_ref()));

    // 5. Wrong key: also fails.
    let other = XChaCha20Poly1305::new(&Key::<XChaCha20Poly1305>::generate());
    println!("wrong key  : {:?}", other.decrypt(&nonce, ciphertext.as_ref()));
}
```

**What happens:**

1. `Key::<XChaCha20Poly1305>::generate()` makes a random key of the right length (32 bytes), and `XNonce::generate()` a random 24-byte nonce. Both use the operating system's random source from Lesson 10.9. The trait `Generate` provides `generate`; `KeyInit` provides `new`; `Aead` provides `encrypt` and `decrypt`. All three must be imported.
2. `encrypt` returns a `Vec<u8>`: the ciphertext followed by the 16-byte authentication tag. That is why 15 bytes of plaintext became 31 bytes.
3. `decrypt` with the same key and nonce returns the original bytes.
4. One flipped bit makes the tag check fail. You get an `Err`, **not** corrupted plaintext. This is the "authenticated" part of AEAD.
5. A wrong key fails in the same way. The error is deliberately empty: it does not say whether the key, the nonce or the data was wrong.

```text
plaintext  : 15 bytes
ciphertext : 31 bytes = fc60277b93f988d3faef3d95172669cef1919c30a9beaae837cb1fb0915ae4
decrypted  : meet me at noon
tampered   : Err(Error)
wrong key  : Err(Error)
```

The ciphertext hex and the key change on every run (they are random); the sizes and the results do not.

> 🔒 **Security:** a nonce is **not secret**: you store it or send it next to the ciphertext. The key is the secret.

## 3. The nonce rule

For each key, every message needs a nonce that has **never been used before** with that key. If you encrypt two messages with the
same key and the same nonce, an attacker who sees both ciphertexts learns a lot. Create `src/bin/nonce_reuse.rs` to see it (this
uses your own key and your own messages, so it is a safe experiment):

```rust
use chacha20poly1305::aead::{Aead, Generate, Key, KeyInit};
use chacha20poly1305::{ChaCha20Poly1305, Nonce};

fn main() {
    let key = Key::<ChaCha20Poly1305>::generate();
    let cipher = ChaCha20Poly1305::new(&key);
    let nonce = Nonce::generate();

    // THE MISTAKE: two different messages encrypted with the same key AND the same nonce.
    let m1 = b"attack at dawn!!";
    let m2 = b"retreat at noon!";
    let c1 = cipher.encrypt(&nonce, m1.as_ref()).unwrap();
    let c2 = cipher.encrypt(&nonce, m2.as_ref()).unwrap();

    // An eavesdropper sees only c1 and c2 (without the 16-byte tag at the end),
    // and XORs them. The secret keystream cancels out.
    let xor_of_ciphertexts: Vec<u8> = c1.iter().zip(&c2).take(m1.len()).map(|(a, b)| a ^ b).collect();
    let xor_of_plaintexts: Vec<u8> = m1.iter().zip(m2).map(|(a, b)| a ^ b).collect();
    println!("c1 xor c2 = {}", hex::encode(&xor_of_ciphertexts));
    println!("m1 xor m2 = {}", hex::encode(&xor_of_plaintexts));
    println!("equal: {}", xor_of_ciphertexts == xor_of_plaintexts);
}
```

**What happens:**

1. ChaCha20 is a *stream cipher*: from the key and the nonce it produces a long secret "keystream" and combines it with the plaintext by XOR (exclusive or, a bit operation whose result bit is 1 where the two input bits differ): `c = m ^ keystream`.
2. With the same key and nonce, both messages use the **same** keystream. XOR of the two ciphertexts is `(m1 ^ k) ^ (m2 ^ k) = m1 ^ m2`: the keystream disappears.
3. The attacker now holds the XOR of the two plaintexts, without knowing the key. With guessing and known text patterns (such as `attack at`), both messages can usually be recovered.

```text
c1 xor c2 = 13110013060a54411554440f18014f00
m1 xor m2 = 13110013060a54411554440f18014f00
equal: true
```

How to choose nonces safely:

| Cipher | Key | Nonce | Nonce strategy |
|---|---|---|---|
| **XChaCha20-Poly1305** | 32 bytes | 24 bytes | A random nonce for each message is safe. **Use this by default.** |
| ChaCha20-Poly1305 | 32 bytes | 12 bytes | A counter, or random only for a limited number of messages. |
| AES-256-GCM | 32 bytes | 12 bytes | The same limit: random 12-byte nonces are safe for only about 2^32 messages per key. Needs care. |

A 24-byte random nonce is so large that two equal values will practically never occur, so with XChaCha20-Poly1305 you simply generate a fresh random nonce for every encryption.
(AES-GCM is also a good, widely used AEAD, and CPUs (Central Processing Units, the processors) with AES instructions run it very fast. The crate `aes-gcm` has the same API (Application Programming Interface: the functions the crate offers); see Exercise 1.)

## 4. A sealed box: nonce and ciphertext together

Decryption needs the nonce, so you store it with the ciphertext. A simple, common format is `nonce || ciphertext+tag`, where `||`
means "followed by":

```text
 offset:   0                          24
           ┌──────────────────────────┬──────────────────────────────┐
           │ nonce (24 bytes, random) │ ciphertext  +  tag (16 bytes) │
           └──────────────────────────┴──────────────────────────────┘
```

Create the library `src/lib.rs` (a package can have both a library and `src/bin/` programs):

```rust
use chacha20poly1305::aead::{Aead, Generate, KeyInit};
use chacha20poly1305::{XChaCha20Poly1305, XNonce};

const NONCE_LEN: usize = 24;

/// Encrypts `plaintext`. The result is `nonce (24 bytes) || ciphertext+tag`.
pub fn seal(key: &[u8; 32], plaintext: &[u8]) -> Vec<u8> {
    let cipher = XChaCha20Poly1305::new_from_slice(key).expect("the key is 32 bytes");
    let nonce = XNonce::generate(); // fresh random nonce for every message
    let mut out = nonce.to_vec();
    out.extend(cipher.encrypt(&nonce, plaintext).expect("encryption cannot fail for in-memory data"));
    out
}

/// Decrypts what `seal` produced. Returns `None` if the data was changed, truncated or the key is wrong.
pub fn open(key: &[u8; 32], sealed: &[u8]) -> Option<Vec<u8>> {
    if sealed.len() < NONCE_LEN {
        return None;
    }
    let (nonce, ciphertext) = sealed.split_at(NONCE_LEN);
    let cipher = XChaCha20Poly1305::new_from_slice(key).expect("the key is 32 bytes");
    let nonce = XNonce::try_from(nonce).ok()?;
    cipher.decrypt(&nonce, ciphertext).ok()
}

#[cfg(test)]
mod tests {
    use super::*;

    const KEY: [u8; 32] = [7; 32];

    #[test]
    fn round_trip() {
        let sealed = seal(&KEY, b"hello");
        assert_eq!(open(&KEY, &sealed).as_deref(), Some(&b"hello"[..]));
    }

    #[test]
    fn same_message_gives_different_output_each_time() {
        assert_ne!(seal(&KEY, b"hello"), seal(&KEY, b"hello"));
    }

    #[test]
    fn any_flipped_bit_is_detected() {
        let sealed = seal(&KEY, b"hello");
        for i in 0..sealed.len() {
            let mut bad = sealed.clone();
            bad[i] ^= 1;
            assert_eq!(open(&KEY, &bad), None, "flip at byte {i} was not detected");
        }
    }

    #[test]
    fn wrong_key_and_truncation_are_rejected() {
        let sealed = seal(&KEY, b"hello");
        assert_eq!(open(&[8; 32], &sealed), None);
        assert_eq!(open(&KEY, &sealed[..10]), None);
        assert_eq!(open(&KEY, &[]), None);
    }
}
```

**What happens:**

1. `seal` creates a fresh nonce, starts the output with the nonce bytes, and appends the encrypted data. A function that takes `&[u8; 32]` cannot be called with a key of the wrong length: the type checks it.
2. `open` is written for **untrusted input**: it first checks that there are at least 24 bytes (otherwise `split_at` would panic), splits off the nonce, and turns every failure into `None`. It never panics on bad data.
3. `new_from_slice(key).expect(…)` cannot fail here, because the array type guarantees 32 bytes.
4. The tests cover the properties that matter: round trip, a different output each time (new nonce), **every single flipped byte detected** (including bytes in the nonce), and wrong key or truncated data rejected.

Run `cargo test`:

```text
running 4 tests
test tests::wrong_key_and_truncation_are_rejected ... ok
test tests::round_trip ... ok
test tests::same_message_gives_different_output_each_time ... ok
test tests::any_flipped_bit_is_detected ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out; finished in 0.00s
```

> 🧪 **Testing:** for every encryption function, test tampering. A function that "round-trips" but accepts modified data is broken in the way that matters most.

### Associated data

Sometimes part of a message must stay readable but must not be changeable: a file-format header, a version number, the user name a record belongs to.
That is *associated data*: it is not encrypted, but the tag covers it. Create `src/bin/aad.rs`:

```rust
use chacha20poly1305::aead::{Aead, Generate, Key, KeyInit, Payload};
use chacha20poly1305::{XChaCha20Poly1305, XNonce};

fn main() {
    let key = Key::<XChaCha20Poly1305>::generate();
    let cipher = XChaCha20Poly1305::new(&key);
    let nonce = XNonce::generate();

    // The header ("vault v1") is NOT encrypted, but it is authenticated.
    let header = b"vault v1";
    let sealed = cipher
        .encrypt(&nonce, Payload { msg: b"top secret", aad: header })
        .unwrap();

    let ok = cipher.decrypt(&nonce, Payload { msg: &sealed, aad: header });
    println!("same header : {:?}", ok.map(|v| String::from_utf8(v).unwrap()));

    let bad = cipher.decrypt(&nonce, Payload { msg: &sealed, aad: b"vault v2" });
    println!("other header: {bad:?}");
}
```

```text
same header : Ok("top secret")
other header: Err(Error)
```

**What happens:** `Payload { msg, aad }` passes the message and the associated data together. Decryption succeeds only if the `aad` is byte for byte the same. Project 10.14 uses this idea to protect its file header.

## 5. Where does the key come from?

An encryption key is 32 **random** bytes. There are three honest sources:

1. **Generated randomly** and stored safely (a secret store, an operating-system keychain, a protected file). See Lesson 10.8.
2. **Derived from a password** with a slow KDF such as Argon2 (Lesson 10.6): `key = Argon2id(password, salt)`. The salt is random, not secret, and stored beside the data. Project 10.14 does exactly this.
3. **Agreed between two parties** with an asymmetric key exchange (TLS, next lesson).

Never use the password's own bytes as a key, and never use `SHA-256(password)` as a key (SHA-256, the Secure Hash Algorithm with 256 bits): it is fast to guess, as Lesson 10.9 explained.

> ⚠️ **Common mistake:** keeping the key next to the ciphertext ("encrypted" database with the key in the same config file). Encryption only moves the problem to the key: protect the key at least as well as the data.

## 6. Signatures with Ed25519

A signature answers "who made this, and was it changed?". The signer uses the **private** key; everyone checks with the **public** key.
Create `src/bin/sign.rs`:

```rust
use ed25519_dalek::{Signature, Signer, SigningKey, Verifier, VerifyingKey};

fn main() {
    // 1. A signing (private) key is 32 random bytes from the operating system.
    let mut seed = [0u8; 32];
    getrandom::fill(&mut seed).expect("no OS randomness");
    let signing_key = SigningKey::from_bytes(&seed);
    let verifying_key: VerifyingKey = signing_key.verifying_key();
    println!("private key: {} bytes (keep secret)", signing_key.to_bytes().len());
    println!("public key : {}", hex::encode(verifying_key.to_bytes()));

    // 2. Sign a message.
    let message = b"release v1.2.0 sha256=2cf24dba";
    let signature: Signature = signing_key.sign(message);
    println!("signature  : {} bytes", signature.to_bytes().len());

    // 3. Anyone with the PUBLIC key can verify.
    println!("genuine    : {:?}", verifying_key.verify(message, &signature));

    // 4. A changed message fails.
    println!("tampered   : {:?}", verifying_key.verify(b"release v1.2.0 sha256=deadbeef", &signature));

    // 5. A different key pair fails.
    let mut other_seed = [0u8; 32];
    getrandom::fill(&mut other_seed).unwrap();
    let other_public = SigningKey::from_bytes(&other_seed).verifying_key();
    println!("wrong key  : {:?}", other_public.verify(message, &signature));
}
```

**What happens:**

1. An Ed25519 private key is just 32 random bytes. `SigningKey::from_bytes` builds the signing key from them, and `verifying_key()` computes the matching 32-byte public key.
2. `sign` returns a 64-byte `Signature`. The trait `Signer` provides `sign` and `Verifier` provides `verify`.
3. `verify` returns `Ok(())` for the genuine message and an error for a changed message or for a different public key.

```text
private key: 32 bytes (keep secret)
public key : 41f320cdb76504b67fcd037d72544c5afac68ec61a4b4b0ea83c4b739c0a7c40
signature  : 64 bytes
genuine    : Ok(())
tampered   : Err(signature::Error {})
wrong key  : Err(signature::Error {})
```

(The public key differs on every run.) Compare with a MAC from Lesson 10.9:

| | MAC (HMAC) | Signature (Ed25519) |
|---|---|---|
| Key | one shared secret | private key signs, public key verifies |
| Who can create a valid tag | everyone who has the key (so any of them could forge) | only the owner of the private key |
| Who can check it | everyone who has the key | **anyone**, with the public key |
| Use for | messages between two parties that trust each other | software releases, certificates, documents, tokens checked by many services |

One question remains: how do you know a public key really belongs to the person you think? If an attacker swaps the public key, your
check is worthless. The answer is *certificates*: a trusted party signs "this public key belongs to this name". The next lesson shows how it works in TLS.

> 🔒 **Security:** the private signing key is the most sensitive file you have. Never commit it to Git (Phase 01), never log it, and keep it out of
> source code. Publish only the public key. Lesson 10.8 covers storing secrets, and Lesson 10.13 shows how the checksums in `Cargo.lock` and the advisory database protect the crates you download.

## Common mistakes

**1. Using an API from an old tutorial.** Many examples call `XChaCha20Poly1305::generate_nonce(&mut OsRng)`. In `chacha20poly1305` 0.11 this function no longer exists:

```text
error[E0599]: no associated function or constant named `generate_nonce` found for struct `ChaChaPoly1305<C, N>` in the current scope
 --> src/bin/encrypt.rs:7:36
  |
7 |     let nonce = XChaCha20Poly1305::generate_nonce().expect("no OS randomness");
  |                                    ^^^^^^^^^^^^^^ associated function or constant not found in `ChaChaPoly1305<…>`
```

Fix: import the trait `Generate` and write `XNonce::generate()`. When code from a blog does not compile, open the documentation for the **exact version** in `Cargo.toml`.

**2. Reusing a nonce** (a constant such as `[0u8; 24]`, or one nonce for a whole file). It compiles and it "works", and it breaks the encryption (Section 3).

**3. Encrypting without authentication, or ignoring the error from `decrypt`.** `decrypt(...).unwrap_or_default()` turns an attack into an empty file. Treat `Err` as "this data is not trustworthy".

**4. Believing that encryption proves identity.** Anyone who has the key can create valid ciphertext. If you need to know *who* wrote something, sign it.

**5. Writing your own mix of primitives** ("I will XOR with a SHA-256 of the key"). Use the AEAD API as it is.

## Try it yourself

1. Run `cargo run --bin encrypt` three times. Which values change, and which stay the same?
2. In `encrypt.rs`, flip a bit in the **last** byte of the ciphertext instead of the first. Is it detected? (The last 16 bytes are the tag.)
3. In `sign.rs`, change one byte of the signature before verifying: `let mut bytes = signature.to_bytes(); bytes[0] ^= 1; let bad = Signature::from_bytes(&bytes);`.

## Exercises

### Exercise 1 — Switch to AES-256-GCM (easy)

Write `src/bin/aes.rs` that encrypts and decrypts `meet me at noon` with `Aes256Gcm` instead. Print the nonce length and the ciphertext length.

<details>
<summary>Hint</summary>

The crate `aes-gcm` has the same traits: `use aes_gcm::aead::{Aead, Generate, Key, KeyInit};` and `use aes_gcm::{Aes256Gcm, Nonce};`.

</details>

<details>
<summary>Answer</summary>

```rust
use aes_gcm::aead::{Aead, Generate, Key, KeyInit};
use aes_gcm::{Aes256Gcm, Nonce};

fn main() {
    let key = Key::<Aes256Gcm>::generate();
    let cipher = Aes256Gcm::new(&key);
    let nonce = Nonce::generate(); // 12 bytes for AES-GCM
    let ct = cipher.encrypt(&nonce, b"meet me at noon".as_ref()).unwrap();
    println!("nonce {} bytes, ciphertext {} bytes", nonce.len(), ct.len());
    let pt = cipher.decrypt(&nonce, ct.as_ref()).unwrap();
    println!("{}", String::from_utf8(pt).unwrap());
}
```

```text
nonce 12 bytes, ciphertext 31 bytes
meet me at noon
```

Why it works: all AEAD ciphers in the RustCrypto family share the `aead` traits, so changing the algorithm changes only the type names. The nonce is 12 bytes and the tag adds 16 bytes (15 + 16 = 31).

</details>

### Exercise 2 — Bind a header to the sealed box (medium)

Add `seal_with_header(key, header, plaintext)` and `open_with_header(key, header, sealed)` to `src/lib.rs`. They work like `seal` and `open` but pass the header as associated data. Write a test that checks that opening with another header fails.

<details>
<summary>Hint</summary>

Build a `Payload { msg: plaintext, aad: header }` for encryption and `Payload { msg: ciphertext, aad: header }` for decryption. Import `Payload` from `chacha20poly1305::aead`.

</details>

<details>
<summary>Answer</summary>

Change the first line to `use chacha20poly1305::aead::{Aead, Generate, KeyInit, Payload};` and add:

```rust
/// Like `seal`, but also binds a plain-text `header` to the message.
pub fn seal_with_header(key: &[u8; 32], header: &[u8], plaintext: &[u8]) -> Vec<u8> {
    let cipher = XChaCha20Poly1305::new_from_slice(key).expect("the key is 32 bytes");
    let nonce = XNonce::generate();
    let mut out = nonce.to_vec();
    let payload = Payload { msg: plaintext, aad: header };
    out.extend(cipher.encrypt(&nonce, payload).expect("encryption cannot fail for in-memory data"));
    out
}

/// Opens data made by `seal_with_header`; fails if the header is not exactly the same.
pub fn open_with_header(key: &[u8; 32], header: &[u8], sealed: &[u8]) -> Option<Vec<u8>> {
    if sealed.len() < NONCE_LEN {
        return None;
    }
    let (nonce, ciphertext) = sealed.split_at(NONCE_LEN);
    let cipher = XChaCha20Poly1305::new_from_slice(key).expect("the key is 32 bytes");
    let nonce = XNonce::try_from(nonce).ok()?;
    cipher.decrypt(&nonce, Payload { msg: ciphertext, aad: header }).ok()
}
```

And inside `mod tests`:

```rust
    #[test]
    fn header_is_authenticated() {
        let sealed = seal_with_header(&KEY, b"vault v1", b"hello");
        assert_eq!(open_with_header(&KEY, b"vault v1", &sealed).as_deref(), Some(&b"hello"[..]));
        assert_eq!(open_with_header(&KEY, b"vault v2", &sealed), None);
    }
```

`cargo test` now shows 5 passing tests.

Why it works: the tag is computed over the ciphertext **and** the associated data, so a different header gives a different expected tag.

</details>

### Exercise 3 — Verify a release (hard)

In a new library package, write `pub fn verify_release(public_key: &[u8; 32], data: &[u8], signature: &[u8; 64]) -> bool` that returns `true` only for a valid
Ed25519 signature. All three inputs are untrusted bytes, so it must never panic. Test: a genuine release, changed data, a changed signature, a wrong public key, and
an invalid public key (`[0xff; 32]`).

<details>
<summary>Hint</summary>

`VerifyingKey::from_bytes(public_key)` returns a `Result` (not every 32 bytes are a valid key). `Signature::from_bytes(signature)` takes `&[u8; 64]`. Use `getrandom` in the tests only: `cargo add --dev getrandom@0.4`.

</details>

<details>
<summary>Answer</summary>

```rust
use ed25519_dalek::{Signature, Verifier, VerifyingKey};

pub fn verify_release(public_key: &[u8; 32], data: &[u8], signature: &[u8; 64]) -> bool {
    let Ok(key) = VerifyingKey::from_bytes(public_key) else {
        return false;
    };
    let signature = Signature::from_bytes(signature);
    key.verify(data, &signature).is_ok()
}

#[cfg(test)]
mod tests {
    use super::*;
    use ed25519_dalek::{Signer, SigningKey};

    fn key_pair() -> SigningKey {
        let mut seed = [0u8; 32];
        getrandom::fill(&mut seed).unwrap();
        SigningKey::from_bytes(&seed)
    }

    #[test]
    fn genuine_release_verifies() {
        let sk = key_pair();
        let sig = sk.sign(b"app-1.0.tar.gz bytes");
        assert!(verify_release(&sk.verifying_key().to_bytes(), b"app-1.0.tar.gz bytes", &sig.to_bytes()));
    }

    #[test]
    fn changed_data_changed_signature_and_wrong_key_fail() {
        let sk = key_pair();
        let pk = sk.verifying_key().to_bytes();
        let sig = sk.sign(b"original").to_bytes();
        assert!(!verify_release(&pk, b"modified", &sig));

        let mut bad_sig = sig;
        bad_sig[0] ^= 1;
        assert!(!verify_release(&pk, b"original", &bad_sig));

        let other = key_pair().verifying_key().to_bytes();
        assert!(!verify_release(&other, b"original", &sig));
    }

    #[test]
    fn garbage_public_key_does_not_panic() {
        assert!(!verify_release(&[0xff; 32], b"x", &[0; 64]));
    }
}
```

`cargo test` shows 3 passing tests.

Why it works: `let … else` turns a parsing failure into an early `false`. Every way of failing ends in `false`, and none of them panics, which is what a function that handles hostile input must do.

</details>

## Quiz

1. What is the difference between a MAC and AEAD?

<details>
<summary>Answer</summary>

A MAC only authenticates a message (the message stays readable). AEAD encrypts the data **and** authenticates it (and optionally some extra associated data) in one operation.

</details>

2. What happens if you encrypt two messages with the same key and the same nonce?

<details>
<summary>Answer</summary>

The same keystream is used twice, so the XOR of the two ciphertexts equals the XOR of the two plaintexts. This leaks information and often allows recovering both messages. Always use a fresh nonce.

</details>

3. Is the nonce secret? Where do you store it?

<details>
<summary>Answer</summary>

No, it is not secret. Store it next to the ciphertext, for example in front of it (`nonce || ciphertext+tag`).

</details>

4. `decrypt` returns `Err`. What should your program do?

<details>
<summary>Answer</summary>

Treat the data as untrusted and report a clear error (wrong key, or the data was damaged or tampered with). Never continue with default or partial data.

</details>

5. Who can create a valid Ed25519 signature, and who can verify it?

<details>
<summary>Answer</summary>

Only the holder of the private key can create it. Anyone who has the public key can verify it.

</details>

6. Why is `SHA-256(password)` not a good encryption key?

<details>
<summary>Answer</summary>

It is fast to compute, so an attacker can try billions of passwords per second. Derive keys from passwords with a slow, salted function such as Argon2 (Lessons 10.6 and 10.14).

</details>

## Summary

- Symmetric encryption uses one shared key and is fast; asymmetric cryptography uses a key pair and is used for signatures and key agreement.
- Always use AEAD (XChaCha20-Poly1305, ChaCha20-Poly1305 or AES-256-GCM): it hides the data and detects any change.
- A nonce is public but must never repeat with the same key. XChaCha20-Poly1305 lets you use a fresh random 24-byte nonce for every message.
- Store the nonce with the ciphertext (`nonce || ciphertext+tag`). Associated data lets you authenticate a header without encrypting it.
- Decrypt functions take untrusted input: check lengths, never panic, return an error, and test tampering.
- Keys are random bytes. If they come from a password, use a slow KDF (Argon2) with a random salt.
- Ed25519 signs with a private key and verifies with a public key. Trusting a public key needs certificates (next lesson).

## Sources and further reading

- [chacha20poly1305 crate documentation](https://docs.rs/chacha20poly1305/0.11.0/chacha20poly1305/) — the AEAD API used in this lesson.
- [aes-gcm crate documentation](https://docs.rs/aes-gcm/0.11.1/aes_gcm/) — AES-256-GCM with the same traits.
- [ed25519-dalek crate documentation](https://docs.rs/ed25519-dalek/3.0.0/ed25519_dalek/) — Ed25519 keys, signing and verification.
- [RFC 8439 — ChaCha20 and Poly1305 for IETF Protocols](https://datatracker.ietf.org/doc/html/rfc8439) — the specification of ChaCha20-Poly1305 (an RFC, Request for Comments, is an internet standard document; the IETF, Internet Engineering Task Force, is the group that writes these standards).
- [RFC 8032 — Edwards-Curve Digital Signature Algorithm (EdDSA)](https://datatracker.ietf.org/doc/html/rfc8032) — the specification of Ed25519.
- [OWASP Cryptographic Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cryptographic_Storage_Cheat_Sheet.html) — which algorithms and key practices to choose.

---

[🏠 Start](../START-HERE.md) · [← 10.9 Cryptography I: hashes, MACs and randomness](09-cryptography-hashes-macs-and-randomness.md) · [📋 Phase 10](README.md) · [10.11 TLS and certificates →](11-tls-and-certificates.md) · [📖 Glossary](../GLOSSARY.md) <!-- nav -->
<!-- nav:bottom -->
